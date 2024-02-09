## Lab 19 - Portal Backstage <a name="lab-19---portal-backstage-"></a>

An important feature for dev teams is being able to publish and consume APIs in a common location.  Developer portals based on OpenAPI specifications have grown in popularity and the open source project Backstage is the most popular in this space.  In this lab, you will see how you can utilize Gloo's Backstage integration to publish APIs for your teams and provide some advanced functionality when combining this with Gloo Gateway such as usage plans, rate limiting and advanced metrics.

### Installing Portal
```shell
helm upgrade --install gloo-platform-addons gloo-platform/gloo-platform \
  --version=2.5.0 \
  --namespace=gloo-platform-addons \
  --kube-context web \
  --reuse-values \
  -f data/gloo-addon-values.yaml
```

In order to simplify operations, we will delete the current VirtualGateway serving the online boutique application.
```shell
kubectl delete virtualgateway ingress -n ops-team --context mgmt
```

We are going to create a new VirtualGateway to host all of our APIS.
```shell
kubectl apply --context mgmt -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: VirtualGateway
metadata:
  name: istio-ingressgateway
  namespace: ops-team
spec:
  listeners:
    - http: {}
      port:
        number: 80
      allowedRouteTables:
        - host: api.example.com
        - host: graphql.api.example.com
        - host: developer.example.com
        - host: developer.partner.example.com
        - host: keycloak.example.com
        - host: grafana.example.com
        - host: argocd.example.com
  workloads:
  - selector:
      labels:
        app: gloo-gateway
      cluster: web
      namespace: istio-ingress
EOF
```

Let's grab the hostname for the gateway for the next step.
```shell
GW_HOST=$(kubectl get svc --context web -n istio-ingress istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
[[ -z "$GW_HOST" ]] && { GW_HOST=$(kubectl get svc --context web -n istio-ingress istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}');}
printf "\nIngress gateway hostame: %s\n" $GW_HOST
```

### Installing Keycloak
Keycloak is an open source IdP that provides several authentication methods.  We will use it to hold our users.

Let's install Keycloak.
```shell
kubectl create ns keycloak --context web

kubectl --context web -n keycloak create -f data/keycloak.yaml
```

Wait for Keycloak to be ready.
```shell
kubectl --context web -n keycloak rollout status deploy/keycloak
```

We need to modify the ops-team Workspace to include the keycloak namespace
```shell
kubectl apply -f data/ops-team.yaml --context mgmt
```

Create a RouteTable for Keycloak
```shell
kubectl apply -f data/keycloak-example-com-rt.yaml --context mgmt
```

### Installing Backstage
Now, let's create backstage.

```shell
kubectl create ns backstage --context web

kubectl -n backstage create serviceaccount backstage-kube-sa --context web

kubectl apply --context web -f - <<EOF
  apiVersion: v1
  kind: Secret
  metadata:
    name: backstage-kube-sa-secret
    namespace: backstage
    annotations:
      kubernetes.io/service-account.name: backstage-kube-sa
  type: kubernetes.io/service-account-token
EOF

kubectl apply --context web -f - <<EOF
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: backstage-read-only
  rules:
    - apiGroups:
        - '*'
      resources:
        - pods
        - configmaps
        - services
        - deployments
        - replicasets
        - horizontalpodautoscalers
        - ingresses
        - statefulsets
        - limitranges
        - daemonsets
        - routetables
      verbs:
        - get
        - list
        - watch
    - apiGroups:
        - batch
      resources:
        - jobs
        - cronjobs
      verbs:
        - get
        - list
        - watch
    - apiGroups:
        - metrics.k8s.io
      resources:
        - pods
      verbs:
        - get
        - list
EOF

kubectl apply --context web -f - <<EOF
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: backstage-kube-sa-read-only
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: backstage-read-only
  subjects:
  - kind: ServiceAccount
    name: backstage-kube-sa
    namespace: backstage
EOF

printf "\nDeploying Backstage.\n"
helm repo add ddoyle-gloo-demo https://duncandoyle.github.io/gloo-demo-helm-charts
helm repo update
helm upgrade --install gp-portal-demo-backstage ddoyle-gloo-demo/gp-portal-demo-backstage \
  --namespace backstage \
  --create-namespace \
  --version 0.1.4 \
  --kube-context=web \
  --set kubernetes.skipTLSVerify=true
```

### Keycloak Configuration
Now, we need to configure Keycloak.