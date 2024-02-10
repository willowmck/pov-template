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
