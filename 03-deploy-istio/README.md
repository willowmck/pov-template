## Lab 03 - Deploy Istio <a name="lab-03---deploy-istio-"></a>


![Istio Gloo UI](images/gloo-ui-istio.png)

Gloo Platform works with Open Source Istio distributions but Solo.io offers a number of different distributions of Istio for different types of environments and use cases such as FIPS, Arm, and distroless.
To learn more about the different distributions view [Gloo Istio Distributions](https://docs.solo.io/gloo-mesh-enterprise/latest/reference/version/gloo_mesh_istio/).

Links:
- [Gloo Istio](https://docs.solo.io/gloo-mesh-enterprise/latest/reference/version/gloo_mesh_istio/)
- [Supported Istio Versions](https://docs.solo.io/gloo-mesh-enterprise/latest/reference/version/versions/)
- [Gloo Platform Managed Istio](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/installation/istio/gm_managed_istio/)
- [Manual Istio Installs](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/prod/manual/)

Installing Istio via `helm` is the preferred method for installing Istio manually. If you prefer to not manage you Istio installations you can trust that responsibility to Gloo Platform via its Istio [Istio Lifecycle Manager](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/installation/istio/gm_managed_istio/)
Istio will be installed using `revisions` which is the optimal way to deploy Istio for production environments. It allows for the ability to **canary** Istio components that is safe and will prevent outages.

* First add the Istio helm repositories
```shell
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

# Using the Istio Lifecycle Manager

* Create `istio-system`, `istio-eastwest`, `istio-ingress` namespaces
```shell
kubectl apply --context web -f data/namespaces.yaml
kubectl apply --context lob -f data/namespaces.yaml
```

* Install the Istio control-planes using IstioLifecycleManager
```shell
kubectl apply --context mgmt -f data/control-plane.yaml
```

* Install the Istio gateways using GatewayLifecycleManager
```shell
kubectl apply --context mgmt -f data/gateways.yaml
```

* Verify pods are running on the web cluster
```bash
kubectl get pods --context web -n istio-system
kubectl get pods --no-headers --context web -n istio-ingress
kubectl get pods --no-headers --context web -n istio-eastwest
```

* Verify the load balancer is created on the web cluster`
```shell
kubectl get service --context web -n istio-ingress
kubectl get service --context web -n istio-eastwest
```

* Verify pods are running on the lob cluster
```bash
kubectl get pods --context lob -n istio-system
kubectl get pods --no-headers --context lob -n istio-eastwest
```

* Verify the load balancer is created on the lob cluster`
```shell
kubectl get service --context lob -n istio-eastwest
```

# Using Helm

## Install Istio on Cluster: web

![Istio Components web](images/istio-cluster1.png)

* Create `istio-system`, `istio-eastwest`, `istio-ingress` namespaces
```shell
kubectl apply --context web -f data/namespaces.yaml
```

* Before installing Istio or upgrading the istio/base must be run to install or reconcile the CRDs within the kubernetes cluster. 
```shell
helm upgrade -i istio-base istio/base \
  -n istio-system \
  --version 1.17.5 \
  --set defaultRevision=1-17 \
  --kube-context=web
```

* Install the `istiod` control plane 
```shell
helm upgrade -i istiod-1-17 istio/istiod \
  --set revision=1-17 \
  --version 1.17.5 \
  --namespace istio-system  \
  --kube-context=web \
  --set "global.multiCluster.clusterName=web" \
  --set "meshConfig.trustDomain=web" \
  -f data/istiod-values.yaml
```

* Install the Istio eastwest gateway which is used for multi-cluster communication between clusters using mTLS.
```shell
helm upgrade -i istio-eastwestgateway istio/gateway \
  --set revision=1-17 \
  --version 1.17.5 \
  --namespace istio-eastwest  \
  --kube-context=web \
  -f data/eastwest-values.yaml
```

* Install the Istio ingress gateway with a ClusterIP service type. For best proudction practices and to support multiple revisions a standalone Service will be created to allow easy migration of traffic.
```shell
helm upgrade -i istio-ingressgateway-1-17 istio/gateway \
  --set revision=1-17 \
  --version 1.17.5 \
  --namespace istio-ingress  \
  --kube-context=web \
  -f data/ingress-values.yaml
```

* Create the standalone Kubernetes service to sit in front of the Istio ingressgateway.
```shell
kubectl apply --context web -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress
  annotations:
    # Uncomment the following to use an external lb
    # service.beta.kubernetes.io/aws-load-balancer-type: "external"
    # service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
  labels:
    istio: ingressgateway
    app: gloo-gateway
spec:
  type: LoadBalancer
  selector:
    istio: ingressgateway-1-17
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

* Verify pods are running
```bash
kubectl get pods --context web -n istio-system
kubectl get pods --no-headers --context web -n istio-ingress
kubectl get pods --no-headers --context web -n istio-eastwest
```

* Verify the load balancer is created`
```shell
kubectl get service --context web -n istio-ingress
kubectl get service --context web -n istio-eastwest
```

## Install Istio on Cluster: lob

![Istio Components lob](images/istio-cluster2.png)

* Create `istio-system`, `istio-eastwest`, `istio-ingress` namespaces
```shell
kubectl apply --context lob -f data/namespaces.yaml
```

* Install the Istio specific CRDs
```shell
helm upgrade -i istio-base istio/base \
  -n istio-system \
  --version 1.17.5 \
  --set defaultRevision=1-17 \
  --kube-context=lob
```

* Install the `istiod` control plane 
```shell
helm upgrade -i istiod-1-17 istio/istiod \
  --set revision=1-17 \
  --version 1.17.5 \
  --namespace istio-system  \
  --kube-context=lob \
  --set "global.multiCluster.clusterName=lob" \
  --set "meshConfig.trustDomain=lob" \
  -f data/istiod-values.yaml
```

* Install istio eastwest gateway
```shell
helm upgrade -i istio-eastwestgateway istio/gateway \
  --set revision=1-17 \
  --version 1.17.5 \
  --namespace istio-eastwest  \
  --kube-context=lob \
  -f data/eastwest-values.yaml
```

* Verify pods are running
```bash
kubectl get pods --context lob -n istio-system
kubectl get pods --no-headers --context lob -n istio-ingress
kubectl get pods --no-headers --context lob -n istio-eastwest
```

* Verify the load balancer is created`
```shell
kubectl get service --context lob -n istio-ingress
kubectl get service --context lob -n istio-eastwest
```
