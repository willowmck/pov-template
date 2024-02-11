## Lab 19 - Portal Backstage <a name="lab-19---portal-backstage-"></a>

An important feature for dev teams is being able to publish and consume APIs in a common location.  Developer portals based on OpenAPI specifications have grown in popularity and the open source project Backstage is the most popular in this space.  In this lab, you will see how you can utilize Gloo's Backstage integration to publish APIs for your teams and provide some advanced functionality when combining this with Gloo Gateway such as usage plans, rate limiting and advanced metrics.

We will leverage bookinfo and httpbin applications for our portal.

### Update ops-team Workspace
First, we need to update our ops-team workspace to include bookinfo.
```shell
kubectl apply --context mgmt -f data/ops-team.yaml
```

### Routing to Bookinfo and HTTPBin
First, we need to create a RouteTable for our applications.
```shell
kubectl apply --context mgmt -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: main-bookinfo
  namespace: ops-team
spec:
  hosts:
    - web-bookinfo.example.com
    - lob-bookinfo.example.com
  virtualGateways:
    - name: ingress
      namespace: ops-team
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: root
      matchers:
      - uri:
          prefix: /
      delegate:
        routeTables:
          - labels:
              expose: "true"
            workspace: bookinfo
          - labels:
              expose: "true"
            workspace: ops-team
        sortMethod: ROUTE_SPECIFICITY
---
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: main-httpbin
  namespace: ops-team
spec:
  hosts:
    - web-httpbin.example.com
  virtualGateways:
    - name: ingress
      namespace: ops-team
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: root
      matchers:
      - uri:
          prefix: /
      delegate:
        routeTables:
          - labels:
              expose: "true"
            workspace: httpbin
        sortMethod: ROUTE_SPECIFICITY
EOF
```

The ops-team has created 2 RouteTables that will delegate to the bookinfo and httpbin teams.  The teams in charge of these workspaces can expose their services through the gateway.  The ops-team can use this main RouteTable to enforce a global WAF policy, but also to have control over which hostnames and paths can be used by each application team.

Then, the bookinfo team can create a RouteTable to determine how they want to handle the traffic.
```shell
kubectl apply --context mgmt -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: productpage
  namespace: bookinfo-team
  labels:
    expose: "true"
spec:
  http:
    - name: productpage
      matchers:
      - uri:
          exact: /productpage
      - uri:
          prefix: /static
      - uri:
          prefix: /api/v1/products
      forwardTo:
        destinations:
          - ref:
              name: productpage
              namespace: bookinfo-frontends
              cluster: web
            port:
              number: 9080
EOF
```

Let's add the domain to our /etc/hosts file.  First, you will need to find the IP address of the ingress gateway
```shell
nslookup ${GLOO_GATEWAY_HTTPS}
```

Then, using this ip address, modify your /etc/hosts file to map it to web-bookinfo.example.com and web-httpbin.example.com.

You can access the productpage service using this URL - https://web-bookinfo.example.com/productpage.

### Deploy Keycloak
In many use cases, you need to restrict the access to your applications to authenticated users.

OpenID Connect (OIDC) is an identity layer on top of the OAuth 2.0 protocol. In OAuth 2.0 flows, authentication is performed by an external Identity Provider (IdP) which, in case of success, returns an Access Token representing the user identity. The protocol does not define the contents and structure of the Access Token, which greatly reduces the portability of OAuth 2.0 implementations.

The goal of OIDC is to address this ambiguity by additionally requiring Identity Providers to return a well-defined ID Token. OIDC ID tokens follow the JSON Web Token standard and contain specific fields that your applications can expect and handle. This standardization allows you to switch between Identity Providers – or support multiple ones at the same time – with minimal, if any, changes to your downstream services; it also allows you to consistently apply additional security measures like Role-Based Access Control (RBAC) based on the identity of your users, i.e. the contents of their ID token.

In this lab, we're going to install Keycloak. It will allow us to setup OIDC workflows later.

```shell
kubectl apply --context web -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: gloo-platform-addons
  labels:
    app: keycloak
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: keycloak
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: gloo-platform-addons
  labels:
    app: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:22.0.5
        args: ["start-dev"]
        env:
        - name: KEYCLOAK_ADMIN
          value: "admin"
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: "admin"
        - name: PROXY_ADDRESS_FORWARDING
          value: "true"
        ports:
        - name: http
          containerPort: 8080
        readinessProbe:
          httpGet:
            path: /realms/master
            port: 8080
EOF

kubectl --context web -n gloo-platform-addons rollout status deploy/keycloak
```

Create a RouteTable for keycloak
```shell
kubectl apply --context mgmt -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: keycloak
  namespace: ops-team
spec:
  hosts:
    - web-keycloak.example.com
  virtualGateways:
    - name: ingress
      namespace: ops-team
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: keycloak
      matchers:
      - uri:
          prefix: /
      forwardTo:
        destinations:
          - ref:
              name: keycloak
              namespace: gloo-platform-addons
              cluster: web
            port:
              number: 8080
EOF
```

Once again, update /etc/hosts for web-keycloak.example.com

Then, we will configure it and create two users:

    User1 credentials: user1/password Email: user1@example.com

    User2 credentials: user2/password Email: user2@solo.io

Let's set the environment variables we need:
```shell
export KEYCLOAK_URL=https://web-keycloak.example.com
```

Now, we need to get a token:
```shell
export KEYCLOAK_TOKEN=$(curl -Ssm 10 -k --fail-with-body \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=admin" \
  -d "grant_type=password" \
  "$KEYCLOAK_URL/realms/master/protocol/openid-connect/token" |
  jq -r .access_token)
```

After that, we configure Keycloak:
```shell
# Create initial token to register the client
read -r client token <<<$(curl -Ssm 10 -k --fail-with-body -H "Authorization: Bearer ${KEYCLOAK_TOKEN}" -H "Content-Type: application/json" \
  -d '{ "expiration": 0, "count": 1 }' \
  $KEYCLOAK_URL/admin/realms/master/clients-initial-access |
  jq -r '[.id, .token] | @tsv')
KEYCLOAK_CLIENT=${client}

# Register the client
read -r id secret <<<$(curl -Ssm 10 -k --fail-with-body -H "Authorization: bearer ${token}" -H "Content-Type: application/json" \
  -d '{ "clientId": "'${KEYCLOAK_CLIENT}'" }' \
  ${KEYCLOAK_URL}/realms/master/clients-registrations/default |
  jq -r '[.id, .secret] | @tsv')
KEYCLOAK_SECRET=${secret}

# Add allowed redirect URIs
curl -m 10 -k --fail-with-body -H "Authorization: Bearer ${KEYCLOAK_TOKEN}" -H "Content-Type: application/json" \
  -X PUT -d '{ "serviceAccountsEnabled": true, "directAccessGrantsEnabled": true, "authorizationServicesEnabled": true, "redirectUris": ["'https://cluster1-httpbin.example.com'/*","'https://cluster1-portal.example.com'/*","'https://cluster1-backstage.example.com'/*"] }' \
  ${KEYCLOAK_URL}/admin/realms/master/clients/${id}

# Set access token lifetime to 30m (default is 1m)
curl -m 10 -k --fail-with-body -H "Authorization: Bearer ${KEYCLOAK_TOKEN}" -H "Content-Type: application/json" \
  -X PUT -d '{ "accessTokenLifespan": 1800 }' \
  ${KEYCLOAK_URL}/admin/realms/master

# Add the group attribute in the JWT token returned by Keycloak
curl -m 10 -k --fail-with-body -H "Authorization: Bearer ${KEYCLOAK_TOKEN}" -H "Content-Type: application/json" \
  -d '{ "name": "group", "protocol": "openid-connect", "protocolMapper": "oidc-usermodel-attribute-mapper", "config": { "claim.name": "group", "jsonType.label": "String", "user.attribute": "group", "id.token.claim": "true", "access.token.claim": "true" } }' \
  ${KEYCLOAK_URL}/admin/realms/master/clients/${id}/protocol-mappers/models

# Add the show_personal_data attribute in the JWT token returned by Keycloak
curl -m 10 -k --fail-with-body -H "Authorization: Bearer ${KEYCLOAK_TOKEN}" -H "Content-Type: application/json" \
  -d '{ "name": "show_personal_data", "protocol": "openid-connect", "protocolMapper": "oidc-usermodel-attribute-mapper", "config": { "claim.name": "show_personal_data", "jsonType.label": "String", "user.attribute": "show_personal_data", "id.token.claim": "true", "access.token.claim": "true"} } ' \
  ${KEYCLOAK_URL}/admin/realms/master/clients/${id}/protocol-mappers/models

# Create first user
curl -m 10 -k --fail-with-body -H "Authorization: Bearer ${KEYCLOAK_TOKEN}" -H "Content-Type: application/json" \
  -d '{ "username": "user1", "email": "user1@example.com", "enabled": true, "attributes": { "group": "users" }, "credentials": [ { "type": "password", "value": "password", "temporary": false } ] }' \
  ${KEYCLOAK_URL}/admin/realms/master/users

# Create second user
curl -m 10 -k --fail-with-body -H "Authorization: Bearer ${KEYCLOAK_TOKEN}" -H "Content-Type: application/json" \
  -d '{ "username": "user2", "email": "user2@solo.io", "enabled": true, "attributes": { "group": "users", "show_personal_data": false }, "credentials": [ { "type": "password", "value": "password", "temporary": false } ] }' \
  ${KEYCLOAK_URL}/admin/realms/master/users

```