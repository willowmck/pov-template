## Lab 15 - Authentication / JWT + JWKS <a name="lab-15---authentication-/-jwt-+-jwks-"></a>


One way for users to authenticate themselves is via JSON Web Tokens or JWT for short. Gloo Platform can validate JWT tokens against an external JSON Web Key Set (JWKS). Below is a lab that validates JWT tokens generated from an external provider Auth0.

![JWT Enforcement](images/jwt.png)
Links:
- [External Authorization Docs](https://docs.solo.io/gloo-mesh-enterprise/latest/policies/external-auth/)
- [JWTPolicy API](https://docs.solo.io/gloo-mesh-enterprise/latest/reference/api/jwt_policy/)
* Reminder to set the `GLOO_GATEWAY_HTTPS` environment variable
```shell
export GLOO_GATEWAY_HTTPS=$(kubectl --context shared -n istio-ingress get svc -l istio=ingressgateway -o jsonpath='{.items[0].status.loadBalancer.ingress[0].*}'):443

echo "SECURE Online Boutique available at https://$GLOO_GATEWAY_HTTPS"
```

## Enforcing JWT Authentication

* Since JWT authentication is best viewed while accessing an API, first verify that the currency application is still availabile.
```shell
# get the available currencies
curl -k https://$GLOO_GATEWAY_HTTPS/currencies

# convert a currency
curl -k "https://$GLOO_GATEWAY_HTTPS/currencies/convert" \
--header 'Content-Type: application/json' \
--data '{
  "from": {
    "currency_code": "USD",
    "nanos": 0,
    "units": 8
  },
  "to_code": "EUR"
}'
```
* Since the Auth0 servers are not known to Gloo Platform, create an ExternalService reference.
```shell
kubectl apply --context mgmt -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: ExternalService
metadata:
  name: auth0
  namespace: app-team
spec:
  hosts:
  - dev-64ktibmv.us.auth0.com
  ports:
  - name: https
    number: 443
    protocol: HTTPS
    clientsideTls: {}
EOF
```

* Enable JWT Authentication for the currency routes
```shell
kubectl apply --context mgmt -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: JWTPolicy
metadata:
  name: currency
  namespace: app-team
spec:
  applyToRoutes:
  - route:
      labels:
        route: currency
  config:
    providers:
      auth0:
        issuer: "https://dev-64ktibmv.us.auth0.com/"
        audiences:
        - "https://currency/api"
        remote:
          url: "https://dev-64ktibmv.us.auth0.com/.well-known/jwks.json"
          destinationRef:
            ref:
              name: auth0
              namespace: app-team
              cluster: mgmt
            kind: EXTERNAL_SERVICE
            port:
              number: 443
          enableAsyncFetch: true
        keepToken: true
EOF
```

* Call the currency API and get denied
```shell
curl -vk https://$GLOO_GATEWAY_HTTPS/currencies
```

Expected output
```txt
Jwt is missing
```

* Generate JWT from Auth0 using the below command
```shell
ACCESS_TOKEN=$(curl -sS --request POST \
  --url https://dev-64ktibmv.us.auth0.com/oauth/token \
  --header 'content-type: application/json' \
  --data '{"client_id":"1QEVhZ2ERqZOpTQnHChK1TUSKRBduO72","client_secret":"J_vl_qgu0pvudTfGppm_PJcQjkgy-kmy5KRCQDj5XHZbo5eFtxmSbpmqYT5ITv2h","audience":"https://currency/api","grant_type":"client_credentials"}' | jq -r '.access_token')

printf "\n\n Access Token: $ACCESS_TOKEN\n"
```

* Call currency API using the Access token to authenticate
```shell
curl -vk -H "Authorization: Bearer $ACCESS_TOKEN" https://$GLOO_GATEWAY_HTTPS/currencies
```


* Cleanup JWT policy 
```
kubectl delete jwtpolicy currency --context mgmt -n app-team
```
