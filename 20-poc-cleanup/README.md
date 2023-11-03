## Lab 20 - POC Clean Up <a name="lab-20---poc-clean-up-"></a>


You have completed the POC! The final step is to clean up the deployed assets and reset the environments back to their original state.

> Some of the commands here may try and cleanup things that dont exist. This is just to make sure all POC assets were accounted for. 


## Clean Up Applications

* Remove the Online Boutique Applications in web
```shell
helm uninstall online-boutique \
  --namespace online-boutique \
  --kube-context web

helm uninstall toys-catalog \
  --namespace online-boutique \
  --kube-context web
```

* Remove the Online Boutique Applications in lob
```shell
helm uninstall ha-frontend \
  --namespace online-boutique \
  --kube-context lob

helm uninstall checkout-apis \
  --namespace checkout-apis \
  --kube-context lob
```

* Delete the namespaces
```shell
kubectl delete namespace online-boutique --context web

kubectl delete namespace online-boutique --context lob
kubectl delete namespace checkout-apis --context lob
```

## Clean up Gloo Addons

* Remove the Gloo Addons
```shell
helm uninstall gloo-platform-addons \
  --namespace gloo-platform-addons \
  --kube-context web
```

* Delete the Gloo Addons namespace
```
kubectl delete namespace gloo-platform-addons --context web
```

## Clean Up Istio

* Depending on if Istio was upgraded or not, the following commands may need to be modified.
```shell
helm ls -n istio-system --kube-context web
helm ls -n istio-ingress --kube-context web
helm ls -n istio-eastwest --kube-context web

helm ls -n istio-system --kube-context lob
helm ls -n istio-eastwest --kube-context lob
```

* Uninstall the gateways in web
```shell
helm uninstall istio-eastwestgateway \
  --namespace istio-eastwest \
  --kube-context web

helm uninstall istio-ingressgateway-1-16 \
  --namespace istio-ingress \
  --kube-context web

helm uninstall istio-ingressgateway-1-17 \
  --namespace istio-ingress \
  --kube-context web
```

* Uninstall the control plane in web
```shell
helm uninstall istiod-1-16 \
  --namespace istio-system \
  --kube-context web

helm uninstall istiod-1-17 \
  --namespace istio-system \
  --kube-context web
```

* Uninstall the gateways in lob
```shell
helm uninstall istio-eastwestgateway \
  --namespace istio-eastwest \
  --kube-context lob
```

* Uninstall the control plane in web
```shell
helm uninstall istiod-1-16 \
  --namespace istio-system \
  --kube-context lob

helm uninstall istiod-1-17 \
  --namespace istio-system \
  --kube-context lob
```

* Remove the CRDS **NOTE** Make sure other teams are not using Istio on the cluster
```shell
helm template istio-base istio/base --namespace istio-system --include-crds | kubectl delete --context web -f -

helm template istio-base istio/base --namespace istio-system --include-crds | kubectl delete --context lob -f -
```

* Cleanup the namespaces
```shell
kubectl delete namespace istio-system --context web

kubectl delete namespace istio-system --context lob
```

## Clean Up Gloo Platform

* Remove Agents from workload clusters
```shell
helm uninstall gloo-agent \
  --namespace gloo-mesh \
  --kube-context web

helm uninstall gloo-agent \
  --namespace gloo-mesh \
  --kube-context lob
```

* Remove Gloo CRDs
```shell
helm uninstall gloo-platform-crds \
  --namespace gloo-mesh \
  --kube-context web

helm uninstall gloo-platform-crds \
  --namespace gloo-mesh \
  --kube-context lob
```

* Delete gloo-mesh namespaces
```shell
kubectl delete namespace gloo-mesh --context web

kubectl delete namespace gloo-mesh --context lob
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
kubectl delete namespace keycloak --context web
```

* Cleanup cert-manager
```shell
helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context mgmt

helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context web

helm uninstall cert-manager \
  --namespace cert-manager \
  --kube-context lob
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
