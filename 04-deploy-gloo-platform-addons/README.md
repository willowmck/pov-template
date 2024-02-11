## Lab 04 - Deploy Gloo Platform Addons <a name="lab-04---deploy-gloo-platform-addons-"></a>

The Gloo Platform Addons are extensions that helm enable certain features that are offered within. The Gloo Platform addons contain a set of applications and cache to enable rate limiting.
![Gloo Platform Addon Components](images/gloo-platform-addons.png)

Create the Gloo Platform Addons namespace
```shell
kubectl apply --context web -f data/namespaces.yaml
```

* Install Gloo Platform Addon applications in web
```shell
helm upgrade -i gloo-platform-addons gloo-platform/gloo-platform \
   --kube-context web \
   --namespace gloo-platform-addons \
   --version v2.5.0 \
   -f data/gloo-platform-addons.yaml
```

* Verify pods are up and running
```bash
kubectl get pods -n gloo-platform-addons --context web
```

* Register the external authorization server with Gloo Platform
```shell
kubectl apply --context web -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: ExtAuthServer
metadata:
  name: ext-auth-server
  namespace: gloo-platform-addons
spec:
  destinationServer:
    port:
      name: grpc
    ref:
      cluster: web
      name: ext-auth-service
      namespace: gloo-platform-addons
  requestBody: {}
EOF
```

We also need to setup rate limiting for the usage plans
```shell
kubectl apply --context web -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: RateLimitServerSettings
metadata:
  name: rate-limit-server
  namespace: gloo-platform-addons
spec:
  destinationServer:
    ref:
      cluster: web
      name: rate-limiter
      namespace: gloo-platform-addons
    port:
      name: grpc
EOF
```