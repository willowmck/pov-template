## Lab 20 - POC Clean Up <a name="lab-20---poc-clean-up-"></a>


You have completed the POC! The final step is to clean up the deployed assets and reset the environments back to their original state.

> Some of the commands here may try and cleanup things that dont exist. This is just to make sure all POC assets were accounted for. 


## Clean Up Applications

* Remove the Online Boutique Applications in cluster-1
```shell
helm uninstall online-boutique \
  --namespace online-boutique \
  --kube-context cluster-1

helm uninstall toys-catalog \
  --namespace online-boutique \
  --kube-context cluster-1
```

* Remove the Online Boutique Applications in cluster-2
```shell
helm uninstall ha-frontend \
  --namespace online-boutique \
  --kube-context cluster-2

helm uninstall checkout-apis \
  --namespace checkout-apis \
  --kube-context cluster-2
```

* Delete the namespaces
```shell
kubectl delete namespace online-boutique --context cluster-1

kubectl delete namespace online-boutique --context cluster-2
kubectl delete namespace checkout-apis --context cluster-2
```

## Clean up Gloo Addons

* Remove the Gloo Addons
```shell
helm uninstall gloo-platform-addons \
  --namespace gloo-platform-addons \
  --kube-context cluster-1
```

* Delete the Gloo Addons namespace
```
kubectl delete namespace gloo-platform-addons --context cluster-1
```

## Clean Up Istio

* Depending on if Istio was upgraded or not, the following commands may need to be modified.
```shell
helm ls -n istio-system --kube-context cluster-1
helm ls -n istio-ingress --kube-context cluster-1
helm ls -n istio-eastwest --kube-context cluster-1

helm ls -n istio-system --kube-context cluster-2
helm ls -n istio-eastwest --kube-context cluster-2
```

* Uninstall the gateways in cluster-1
```shell
helm uninstall istio-eastwestgateway \
  --namespace istio-eastwest \
  --kube-context cluster-1

helm uninstall istio-ingressgateway-1-16 \
  --namespace istio-ingress \
  --kube-context cluster-1

helm uninstall istio-ingressgateway-1-17 \
  --namespace istio-ingress \
  --kube-context cluster-1
```

* Uninstall the control plane in cluster-1
```shell
helm uninstall istiod-1-16 \
  --namespace istio-system \
  --kube-context cluster-1

helm uninstall istiod-1-17 \
  --namespace istio-system \
  --kube-context cluster-1
```

* Uninstall the gateways in cluster-2
```shell
helm uninstall istio-eastwestgateway \
  --namespace istio-eastwest \
  --kube-context cluster-2
```

* Uninstall the control plane in cluster-1
```shell
helm uninstall istiod-1-16 \
  --namespace istio-system \
  --kube-context cluster-2

helm uninstall istiod-1-17 \
  --namespace istio-system \
  --kube-context cluster-2
```

* Remove the CRDS **NOTE** Make sure other teams are not using Istio on the cluster
```shell
helm template istio-base istio/base --namespace istio-system --include-crds | kubectl delete --context cluster-1 -f -

helm template istio-base istio/base --namespace istio-system --include-crds | kubectl delete --context cluster-2 -f -
```

* Cleanup the namespaces
```shell
kubectl delete namespace istio-system --context cluster-1

kubectl delete namespace istio-system --context cluster-2
```

## Clean Up Gloo Platform

* Remove Agents from workload clusters
```shell
helm uninstall gloo-agent \
  --namespace gloo-mesh \
  --kube-context cluster-1

helm uninstall gloo-agent \
  --namespace gloo-mesh \
  --kube-context cluster-2
```

* Remove Gloo CRDs
```shell
helm uninstall gloo-platform-crds \
  --namespace gloo-mesh \
  --kube-context cluster-1

helm uninstall gloo-platform-crds \
  --namespace gloo-mesh \
  --kube-context cluster-2
```

* Delete gloo-mesh namespaces
```shell
kubectl delete namespace gloo-mesh --context cluster-1

kubectl delete namespace gloo-mesh --context cluster-2
```

* Remove the Workspace namespaces in management cluster
```shell
kubectl delete namespace ops-team --context management
kubectl delete namespace app-team --context management
kubectl delete namespace checkout-team --context management
```

* Cleanup management cluster
```shell
helm uninstall jaeger \
  --namespace gloo-mesh \
  --kube-context management

helm uninstall gloo-platform \
  --namespace gloo-mesh \
  --kube-context management

helm uninstall gloo-platform-crds \
  --namespace gloo-mesh \
  --kube-context management
```

* Remove namespace
```shell
kubectl delete namespace gloo-mesh --context management
```

## Optional Deployments

* Cleanup Keycloak
```shell
kubectl delete namespace keycloak --context cluster-1
```

* Cleanup cert-manager
```shell
helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context management

helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context cluster-1

helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context cluster-2
```

* Cleanup cert-manager namespace
```shell
kubectl delete namespace cert-manager --context management
```

* Cleanup Vault
```shell
helm uninstall vault \
  --namespace vault \
  --kube-context management
```

* Cleanup vault namespace
```shell
kubectl delete namespace vault --context management
```
