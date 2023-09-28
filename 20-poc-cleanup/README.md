## Lab 20 - POC Clean Up <a name="lab-20---poc-clean-up-"></a>


You have completed the POC! The final step is to clean up the deployed assets and reset the environments back to their original state.

> Some of the commands here may try and cleanup things that dont exist. This is just to make sure all POC assets were accounted for. 


## Clean Up Applications

* Remove the Online Boutique Applications in shared
```shell
helm uninstall online-boutique \
  --namespace online-boutique \
  --kube-context shared

helm uninstall toys-catalog \
  --namespace online-boutique \
  --kube-context shared
```

* Remove the Online Boutique Applications in lob-01
```shell
helm uninstall ha-frontend \
  --namespace online-boutique \
  --kube-context lob-01

helm uninstall checkout-apis \
  --namespace checkout-apis \
  --kube-context lob-01
```

* Delete the namespaces
```shell
kubectl delete namespace online-boutique --context shared

kubectl delete namespace online-boutique --context lob-01
kubectl delete namespace checkout-apis --context lob-01
```

## Clean up Gloo Addons

* Remove the Gloo Addons
```shell
helm uninstall gloo-platform-addons \
  --namespace gloo-platform-addons \
  --kube-context shared
```

* Delete the Gloo Addons namespace
```
kubectl delete namespace gloo-platform-addons --context shared
```

## Clean Up Istio

* Depending on if Istio was upgraded or not, the following commands may need to be modified.
```shell
helm ls -n istio-system --kube-context shared
helm ls -n istio-ingress --kube-context shared
helm ls -n istio-eastwest --kube-context shared

helm ls -n istio-system --kube-context lob-01
helm ls -n istio-eastwest --kube-context lob-01
```

* Uninstall the gateways in shared
```shell
helm uninstall istio-eastwestgateway \
  --namespace istio-eastwest \
  --kube-context shared

helm uninstall istio-ingressgateway-1-16 \
  --namespace istio-ingress \
  --kube-context shared

helm uninstall istio-ingressgateway-1-17 \
  --namespace istio-ingress \
  --kube-context shared
```

* Uninstall the control plane in shared
```shell
helm uninstall istiod-1-16 \
  --namespace istio-system \
  --kube-context shared

helm uninstall istiod-1-17 \
  --namespace istio-system \
  --kube-context shared
```

* Uninstall the gateways in lob-01
```shell
helm uninstall istio-eastwestgateway \
  --namespace istio-eastwest \
  --kube-context lob-01
```

* Uninstall the control plane in shared
```shell
helm uninstall istiod-1-16 \
  --namespace istio-system \
  --kube-context lob-01

helm uninstall istiod-1-17 \
  --namespace istio-system \
  --kube-context lob-01
```

* Remove the CRDS **NOTE** Make sure other teams are not using Istio on the cluster
```shell
helm template istio-base istio/base --namespace istio-system --include-crds | kubectl delete --context shared -f -

helm template istio-base istio/base --namespace istio-system --include-crds | kubectl delete --context lob-01 -f -
```

* Cleanup the namespaces
```shell
kubectl delete namespace istio-system --context shared

kubectl delete namespace istio-system --context lob-01
```

## Clean Up Gloo Platform

* Remove Agents from workload clusters
```shell
helm uninstall gloo-agent \
  --namespace gloo-mesh \
  --kube-context shared

helm uninstall gloo-agent \
  --namespace gloo-mesh \
  --kube-context lob-01
```

* Remove Gloo CRDs
```shell
helm uninstall gloo-platform-crds \
  --namespace gloo-mesh \
  --kube-context shared

helm uninstall gloo-platform-crds \
  --namespace gloo-mesh \
  --kube-context lob-01
```

* Delete gloo-mesh namespaces
```shell
kubectl delete namespace gloo-mesh --context shared

kubectl delete namespace gloo-mesh --context lob-01
```

* Remove the Workspace namespaces in management cluster
```shell
kubectl delete namespace ops-team --context mgmt
kubectl delete namespace app-team --context mgmt
kubectl delete namespace checkout-team --context mgmt
```

* Cleanup management cluster
```shell
helm uninstall jaeger \
  --namespace gloo-mesh \
  --kube-context mgmt

helm uninstall gloo-platform \
  --namespace gloo-mesh \
  --kube-context mgmt

helm uninstall gloo-platform-crds \
  --namespace gloo-mesh \
  --kube-context mgmt
```

* Remove namespace
```shell
kubectl delete namespace gloo-mesh --context mgmt
```

## Optional Deployments

* Cleanup Keycloak
```shell
kubectl delete namespace keycloak --context shared
```

* Cleanup cert-manager
```shell
helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context mgmt

helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context shared

helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context lob-01
```

* Cleanup cert-manager namespace
```shell
kubectl delete namespace cert-manager --context mgmt
```

* Cleanup Vault
```shell
helm uninstall vault \
  --namespace vault \
  --kube-context mgmt
```

* Cleanup vault namespace
```shell
kubectl delete namespace vault --context mgmt
```
