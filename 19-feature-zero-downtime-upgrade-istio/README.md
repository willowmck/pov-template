## Lab 19 - Zero Downtime Istio Upgrades <a name="lab-19---zero-downtime-istio-upgrades-"></a>


![Upgrade Istio](images/upgrade-istio.png)
Historically, upgrading Istio without downtime has been a complicated ordeal. Today, Gloo Platform can easily facilitate zero downtime upgrades using its Istio Lifecycle Manager. Gloo Platform allows you to deploy multiple versions of Istio side-by-side and transition from one version to the other. 
This lab will show you how to upgrade Istio quickly and safely without impacting client traffic. This is done through the use of Istio revisions. 

Links:
- [Istio Production Best Practices](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/prod/manual/namespaces/)
- [Gloo Platform Managed Istio](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/installation/istio/gm_managed_istio/)
- [GatewayLifecycleManager API](https://docs.solo.io/gloo-mesh-enterprise/latest/reference/api/gateway_lifecycle_manager/)
- [IstioLifecycleManager API](https://docs.solo.io/gloo-mesh-enterprise/latest/reference/api/istio_lifecycle_manager/)
## Upgrade Istio using Helm in Cluster: web

* To upgrade Istio we will deploy a whole new canary version beside it. We will also deploy new gateways and migrate traffic to them.

```shell
helm upgrade --install istio-base istio/base \
  -n istio-system \
  --kube-context=web \
  --version 1.20.2 \
  --set defaultRevision=1-20

helm upgrade -i istiod-1-20 istio/istiod \
  --set revision=1-20 \
  --version 1.20.2 \
  --namespace istio-system  \
  --kube-context=web \
  --set "global.multiCluster.clusterName=web" \
  --set "meshConfig.trustDomain=web" \
  -f data/istiod-values.yaml
```

* In this lab we will show you two options for upgrading gateways, in-place and via revisions. Depending on your needs and requirements you can choose the best option that will work for you.

* Upgrade the Istio eastwest gateway in place
```shell
helm upgrade -i istio-eastwestgateway istio/gateway \
  --set revision=1-20 \
  --version 1.20.2 \
  --namespace istio-eastwest  \
  --kube-context=web \
  -f data/eastwest-values.yaml
```

* Deploy new Istio 1-20 ingress gateway using revisions
```shell
helm upgrade -i istio-ingressgateway-1-20 istio/gateway \
  --set revision=1-20 \
  --version 1.20.2 \
  --namespace istio-ingress  \
  --kube-context=web \
  -f data/ingress-values.yaml
```

* Update the standalone Kubernetes service send traffic to the new Istio ingressgateway.
```shell
kubectl apply --context web -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress
  labels:
    app: gloo-gateway
  annotations:
    # Uncomment the following to use an external lb
    # service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
spec:
  type: LoadBalancer
  selector:
    istio: ingressgateway-1-20
  ports:
  # Port for health checks on path /healthz/ready.
  # For AWS ELBs, this port must be listed first.
  - name: status-port
    port: 15021
    targetPort: 15021
  # main http ingress port
  - port: 80
    targetPort: 8080
    name: http2
  # main https ingress port
  - port: 443
    targetPort: 8443
    name: https
EOF
```

* Finally update the application namespaces to the new revision and perform a rolling restart.
```shell
kubectl label namespace online-boutique --overwrite istio.io/rev=1-20 --context web -n online-boutique
kubectl rollout restart deploy --context web -n online-boutique
kubectl label namespace gloo-platform-addons --overwrite istio.io/rev=1-20 --context web -n online-boutique
kubectl rollout restart deploy --context web -n gloo-platform-addons
```

* Remove Istio 
```shell
helm uninstall istio-ingressgateway-1-19 \
  --namespace istio-ingress  \
  --kube-context=web

helm uninstall istiod-1-19 \
  --namespace istio-system  \
  --kube-context=web
```

* Verify only Istio 1-20 is running
```shell
istioctl proxy-status --context web
```

## Upgrade Istio using Helm in Cluster: lob

* Upgrade Istiod to 1-20 components
```shell
helm upgrade --install istio-base istio/base \
  --kube-context=lob \
  -n istio-system \
  --version 1.20.2 \
  --set defaultRevision=1-20

helm upgrade -i istiod-1-20 istio/istiod \
  --set revision=1-20 \
  --version 1.20.2 \
  --namespace istio-system  \
  --kube-context=lob \
  --set "global.multiCluster.clusterName=lob" \
  --set "meshConfig.trustDomain=lob" \
  -f data/istiod-values.yaml

```

* Upgrade the Istio eastwest gateway in place
```shell
helm upgrade -i istio-eastwestgateway istio/gateway \
  --set revision=1-20 \
  --version 1.20.2 \
  --namespace istio-eastwest  \
  --kube-context=lob \
  -f data/eastwest-values.yaml
```

* Finally update the application namespaces to the new revision and perform a rolling restart.
```shell
kubectl label namespace online-boutique --overwrite istio.io/rev=1-20 --context lob -n online-boutique
kubectl rollout restart deploy --context lob -n online-boutique

kubectl label namespace checkout-apis --overwrite istio.io/rev=1-20 --context lob -n checkout-apis
kubectl rollout restart deploy --context lob -n checkout-apis
```

* Remove Istio 
```shell
helm uninstall istiod-1-19 \
  --namespace istio-system  \
  --kube-context=lob
```

* Verify only Istio 1-18 is running
```shell
istioctl proxy-status --context lob
```


