
<!--bash
#!/usr/bin/env bash

source ./scripts/assert.sh
-->



# <center>Proof of Concept</center>
## <center>Prepared for Texas Capital Bank</center>



## Table of Contents
* [Introduction](#introduction)
* [Lab 1 - Deploy EKS clusters](./01-deploy-eks-clusters/README.md)
* [Lab 2 - Deploy Gloo Platform](./02-deploy-gloo-platform/README.md)
* [Lab 3 - Deploy Istio](./03-deploy-istio/README.md)
* [Lab 4 - Deploy Gloo Platform Addons](./04-deploy-gloo-platform-addons/README.md)
* [Lab 5 - Certificate Management](./05-certificates/README.md)
* [Lab 6 - Deploy Online Boutique](./06-deploy-online-boutique/README.md)
* [Lab 7 - Configure Gloo Platform](./07-configure-gloo-platform/README.md)
* [Lab 8 - Ingress](./08-ingress/README.md)
* [Lab 9 - Zero Trust Communication](./09-zero-trust/README.md)
* [Lab 10 - Multi Cluster Secure Communication](./10-multi-cluster/README.md)
* [Lab 11 - Observability](./11-observability/README.md)
* [Lab 12 - Blue Green Application Deployments](./12-feature-blue-green/README.md)
* [Lab 13 - Expose APIs](./13-expose-apis/README.md)
* [Lab 14 - Circuit Breaking and Failover](./14-circuit-breaking/README.md)
* [Lab 15 - Rate Limiting](./15-feature-rate-limiting/README.md)
* [Lab 16 - Authentication / JWT + JWKS](./16-feature-jwt/README.md)
* [Lab 17 - Gloo Platform OPA Integration](./17-feature-opa/README.md)
* [Lab 18 - Calling External Services](./18-feature-external-services/README.md)
* [Lab 19 - Day 2 Certificates](./19-feature-day2-certificates/README.md)
* [Lab 20 - Zero Downtime Istio Upgrades](./20-feature-zero-downtime-upgrade-istio/README.md)
* [Lab 21 - POC Clean Up](./21-poc-cleanup/README.md)



## Introduction <a name="introduction"></a>

![Gloo Platform UI](images/gloo-mesh.png)

[Gloo Platform](https://www.solo.io/products/gloo-platform/) integrates API gateway, API management, Istio service mesh and cloud-native networking into a unified application networking platform. It allows you to manage gateway and service mesh together across single or multiple clusters and multiple teams.

Gloo Platform can help solve some of these challenges:

- Zero trust security architecture for APIs and microservices
- Combine north-south and east-west traffic management (API gateway + service mesh)
- Unified failover and security policy across gateway and mesh 
- Aligning with the leading approaches to software development
- Scale across multiple dimensions (multi-cluster, multi-tenant, multi-cloud)

![Gloo Platform Value](images/gloo-platform-value.png)

## Before You Start

Before starting this POC workshop, it is important that you setup with the right components and tooling in place to ensure success. 

### Supporting Tools

The below tools are designed to help you understand and debug your environment.

- istioctl - Istio CLI `curl -L https://istio.io/downloadIstio | sh -`
- helm v3 - [Helm CLI](https://helm.sh/docs/intro/install/)
- curl - https://everything.curl.dev/get
- docker - [Docker CLI](https://docs.docker.com/get-docker/)

### Documentation and Examples

* Gloo Platform Docs - https://docs.solo.io/gloo-mesh-enterprise/

Solo.io documentation is great for getting to know the Gloo Platform. From examples to API documentation, you can quickly search and find help. It is recommended that you read through the Gloo Platform concepts before getting started. 

* Gloo Platform Concepts - https://docs.solo.io/gloo-mesh-enterprise/latest/concepts/
* API Reference - https://docs.solo.io/gloo-mesh-enterprise/latest/reference/api/

### Cluster Setup

![](images/3-cluster-setup.png)

This POC depends on our typical *3 cluster environment* where one cluster will be used for the administration (management) of the other two clusters (web and lob), which run the gateway, service mesh and your workloads. 

The best way to facilitate the POC is by creating 3 new Kubernetes clusters in your environment so that you can test Gloo Platform without impacting other teams and workloads.
Although, most prospects will want to test that Gloo Platform works in their "real" environments, its important for users to understand Gloo Platform first.
This not only gives the prospect time to learn Gloo Platform but also helps the Solo.io Architects understand the requirements of your environment. 
Prospects are welcome to deploy Gloo Platform to their "real" environments once they have a good grasp of the operational and architectural impacts it will have on their environment. 

If the POC has to be performed on existing web clusters there are some things to be aware of.

**Networking**

* The 3 clusters MUST have network connectivity with each other (internally or externally) via TCP with TLS.
* The management cluster must be allowed to accept mTLS traffic without termination on ports 9900 and 4317. The preferred way to expose these ports is via **TCP Passthrough** load balancers.
  * Optionally if `NodePorts` are required due to the lack of load balancer support, the prospect must be aware of the Node IP addresses it gives each Gloo Agent. There is a chance for the Node to be recreated with a different IP address and thus would drop the connection from the Gloo Agent. 


### Sizing 

* Each cluster should be able to connect to the LoadBalancer address attached to the other clusters by either internal or external networking.
* Minimum Cluster Resource Sizing
  * Management Cluster (mgmt):
    - Nodes: 2
    - CPU Per Node: 4vCPU
    - Memory Per Node: 16Gi
  * Each Workload Cluster (web and lob):
    - Nodes: 2
    - CPU Per Node: 4vCPU
    - Memory Per Node: 16Gi

### Private Image Repository

Some organizations do not allow public docker repository access and need to download the images and upload them to a private repository. 

* To view the images that are used in this POC, run `cat images.txt` 

The easiest way to get the images in your repository is to pull the listed images, retag them to your internal registry and push. 


### Helm charts

The following helm charts are used in this POC or may be needed depending on your use cases.

```shell
# required
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm pull oci://us-central1-docker.pkg.dev/solo-test-236622/solo-demos/onlineboutique

# Optional addons
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

![AwsDiagram](source=sample.svg)


@sample.svg
<?xml version="1.0" encoding="UTF-8"?>
<!-- Do not edit this file with editors other than draw.io -->
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1" width="121px" height="61px" viewBox="-0.5 -0.5 121 61" content="&lt;mxfile host=&quot;Electron&quot; modified=&quot;2023-09-28T16:47:48.282Z&quot; agent=&quot;Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) draw.io/22.0.0 Chrome/114.0.5735.289 Electron/25.8.3 Safari/537.36&quot; etag=&quot;0mPDqw0lXPZP5s4fNYNR&quot; version=&quot;22.0.0&quot; type=&quot;device&quot;&gt;&lt;diagram name=&quot;Page-1&quot; id=&quot;cMGxny8wiAKD3jum78Zm&quot;&gt;jVNNU4QwDP01HHVYyvpxdGE/HMfTHjzXkqXVQpk2ldVfbwthd9FxxktJXl6SkrwmrGiOW8s7+Wwq0EmWVseElUmWLbI8C5+IfI7I3TIfgdqqikhnYK++gMCUUK8qcDMiGqNRdXNQmLYFgTOMW2v6Oe1g9Lxrx2v4BewF17/RF1WhJPQ2Tc+BHahaTq0XU6ThE5sAJ3ll+guIrRNWWGNwtJpjATpObxrMmLf5I3q6mYUW/5PQlE/PV7ubfvXq1+UjK+7Xd49XVOWDa09/XGjvECwFHH5Oo7DGtxXEYmnCVr1UCPuOixjtw/IDJrHRwVsE070DCkmO5EJ6C9tIKvMAvKm6HsqGIayEtx+wUYiqrSnhYFrc8EbpqJoHK2IvgS7cp+Q+zBoskfbG2+EGEjEoIluyh3CEEcQjEtx1bUzoxTvlroVphoBwA3VzGFsEc9Zkma1+thl1mcUfp4GBRTj+uYnFab/hZYBpAG3ok1JCfk+SoEfBcvL7C4lNspEX6rohjJOq61Pp896DQauf3LPEhtjFS2Xrbw==&lt;/diagram&gt;&lt;/mxfile&gt;" style="background-color: rgb(255, 255, 255);"><defs><style type="text/css">@import url(https://fonts.googleapis.com/css?family=Architects+Daughter);&#xa;</style></defs><g><rect x="0" y="0" width="120" height="60" fill="none" stroke="none" pointer-events="all"/><path d="M 3.16 -2.22 L 116.48 -3.95 L 121.18 62.59 L -3.14 62.61" fill="rgb(255, 255, 255)" stroke="none" pointer-events="all"/><path d="M 0 0 C 29.15 3.38 64.66 -2.45 120 0 M 0 0 C 44.84 2.86 91.24 2.69 120 0 M 120 0 C 118.69 17.59 115.29 28.96 120 60 M 120 0 C 120.8 22.15 120.83 44.71 120 60 M 120 60 C 82.41 58.58 48.7 63.78 0 60 M 120 60 C 93.45 61.07 71.9 61.5 0 60 M 0 60 C 1.27 44.05 -0.56 32.59 0 0 M 0 60 C 2.49 43.45 -0.64 29.84 0 0" fill="none" stroke="rgb(0, 0, 0)" stroke-linejoin="round" stroke-linecap="round" stroke-miterlimit="10" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 118px; height: 1px; padding-top: 30px; margin-left: 1px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 20px; font-family: &quot;Architects Daughter&quot;; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">Cluster1</div></div></div></foreignObject><text x="60" y="36" fill="rgb(0, 0, 0)" font-family="Architects Daughter" font-size="20px" text-anchor="middle">Cluster1</text></switch></g></g><switch><g requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"/><a transform="translate(0,-5)" xlink:href="https://www.drawio.com/doc/faq/svg-export-text-problems" target="_blank"><text text-anchor="middle" font-size="10px" x="50%" y="100%">Text is not SVG - cannot display</text></a></switch></svg>
@sample.svg

