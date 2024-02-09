## Lab 06 - Deploy Applications <a name="lab-06---deploy-applications-"></a>


The Online Boutique applicaion is a set of microservices that make up an online shopping website. There is a UI application that reaches out to many APIs to retrieve its data to populate the UI. This workshop will incrementally add features to this website in the coming labs. 

![Online Boutique web](images/online-boutique-cluster1.png)

* Create the Online Boutique namespaces
```shell
kubectl apply --context web -f data/namespaces.yaml
```
* Deploy online boutique into web
```shell
helm upgrade -i online-boutique --version "5.0.3" oci://us-central1-docker.pkg.dev/field-engineering-us/helm-charts/onlineboutique \
  --namespace online-boutique  \
  --kube-context web \
  -f data/values.yaml
```

* Verify pods are running
```bash
kubectl get pods -n online-boutique --context web
```

We will also install the traditional bookinfo application into each cluster.

* Deploy bookinfo on web
```bash
curl https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml > bookinfo.yaml

kubectl --context web create ns bookinfo-frontends
kubectl --context web create ns bookinfo-backends
kubectl --context web label namespace bookinfo-frontends istio.io/rev=1-19 --overwrite
kubectl --context web label namespace bookinfo-backends istio.io/rev=1-19 --overwrite

# deploy the frontend bookinfo service in the bookinfo-frontends namespace
kubectl --context web -n bookinfo-frontends apply -f bookinfo.yaml -l 'account in (productpage)'
kubectl --context web -n bookinfo-frontends apply -f bookinfo.yaml -l 'app in (productpage)'
kubectl --context web -n bookinfo-backends apply -f bookinfo.yaml -l 'account in (reviews,ratings,details)'
# deploy the backend bookinfo services in the bookinfo-backends namespace for all versions less than v3
kubectl --context web -n bookinfo-backends apply -f bookinfo.yaml -l 'app in (reviews,ratings,details),version notin (v3)'
# Update the productpage deployment to set the environment variables to define where the backend services are running
kubectl --context web -n bookinfo-frontends set env deploy/productpage-v1 DETAILS_HOSTNAME=details.bookinfo-backends.svc.cluster.local
kubectl --context web -n bookinfo-frontends set env deploy/productpage-v1 REVIEWS_HOSTNAME=reviews.bookinfo-backends.svc.cluster.local
# Update the reviews service to display where it is coming from
kubectl --context web -n bookinfo-backends set env deploy/reviews-v1 CLUSTER_NAME=web
kubectl --context web -n bookinfo-backends set env deploy/reviews-v2 CLUSTER_NAME=web
```

You can check that the app is running using the following command.
```bash
kubectl --context web -n bookinfo-frontends get pods && kubectl --context web -n bookinfo-backends get pods
```

Note that we deployed the productpage service in the bookinfo-frontends namespace and the other services in the bookinfo-backends namespace.

And we deployed the v1 and v2 versions of the reviews microservice, not the v3 version.

Now, run the following commands to deploy the bookinfo application on lob:
```bash
kubectl --context lob create ns bookinfo-frontends
kubectl --context lob create ns bookinfo-backends
kubectl --context lob label namespace bookinfo-frontends istio.io/rev=1-19 --overwrite
kubectl --context lob label namespace bookinfo-backends istio.io/rev=1-19 --overwrite

# deploy the frontend bookinfo service in the bookinfo-frontends namespace
kubectl --context lob -n bookinfo-frontends apply -f bookinfo.yaml -l 'account in (productpage)'
kubectl --context lob -n bookinfo-frontends apply -f bookinfo.yaml -l 'app in (productpage)'
kubectl --context lob -n bookinfo-backends apply -f bookinfo.yaml -l 'account in (reviews,ratings,details)'
# deploy the backend bookinfo services in the bookinfo-backends namespace for all versions
  kubectl --context lob -n bookinfo-backends apply -f bookinfo.yaml -l 'app in (reviews,ratings,details)'
# Update the productpage deployment to set the environment variables to define where the backend services are running
kubectl --context lob -n bookinfo-frontends set env deploy/productpage-v1 DETAILS_HOSTNAME=details.bookinfo-backends.svc.cluster.local
kubectl --context lob -n bookinfo-frontends set env deploy/productpage-v1 REVIEWS_HOSTNAME=reviews.bookinfo-backends.svc.cluster.local
# Update the reviews service to display where it is coming from
kubectl --context lob -n bookinfo-backends set env deploy/reviews-v1 CLUSTER_NAME=lob
kubectl --context lob -n bookinfo-backends set env deploy/reviews-v2 CLUSTER_NAME=lob
kubectl --context lob -n bookinfo-backends set env deploy/reviews-v3 CLUSTER_NAME=lob
```

You can check that the app is running using:
```bash
kubectl --context lob -n bookinfo-frontends get pods && kubectl --context lob -n bookinfo-backends get pods
```
