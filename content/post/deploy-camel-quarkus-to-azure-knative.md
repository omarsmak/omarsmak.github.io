---
title: "Deploy Native Camel Quarkus Routes To Azure Kubernetes Cluster With Knative"
date: 2020-06-24T10:10:00+02:00
draft: true
tags: ["camel", "quarkus", "graalvm", "azure", "knative"]
---

![image](/images/header.png)

In this blog, we will take close look on how we can deploy Camel Quarkus routes compiled natevly using GraalVM on Azure Kubernets cluster with Knative installed. But first, we will give a brief overview.

&nbsp;

#### Overview
[Apache Camel](https://camel.apache.org/) is a swiss knife integration engine that can be used to integrate between different components within the system using [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/).

[Apache Camel Quarkus](https://camel.apache.org/camel-quarkus/latest/) is a runtime to run Apache Camel routes as [Quarkus](https://quarkus.io/) extensions, which as result utilizes all the performance optimizations brought by Quarkus with GraalVM and thus, allows Camel Routes to be compiled natively using [GraalVM native-image generator](https://www.graalvm.org/docs/reference-manual/native-image/). 

[Knative](https://knative.dev/) is a project that allows deploying, running and managing cloud-native applications to Kubernetes. It allows developers to focus on code instead of worrying about setting up a infrastructure. For instance, many of the complex and repetitive tasks that Kubernetes such as configuration managements, logging, tracing .. etc, can be provided by Knative facilitating other important components like Istio to accomplish these tasks. 

Next, we will take step by step approach on how we can setup our cluster in Azure Kubernetes Cloud (AKS), setting up Knative there and deploying our routes for development environment.

&nbsp;

#### Setting up Azure Kubernetes Cloud (AKS) 
1. Create the K8s cluster AKS great [walkthrough](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough). However for the sake of our testing environment, you will need to enable [HTTP application routing](https://docs.microsoft.com/en-us/azure/aks/http-application-routing) which allows us to access deployed Camel routes publicly. However this should be only one if you need a **development** environment.
2. Once you have everything setup, you can access your K8s dashboard through this command:
```
az aks browse --resource-group camel-dev --name test
```

At this point, we are ready to proceed to the next step on setting up Knative on AKS.

&nbsp;

#### Setting up Knative on Azure Kubernetes Cloud (AKS) 
For simplicity, we will only install the Serving component on Knative onto our AKS cluster with Istio for our networking layer. Based on the steps mentioned [here](https://knative.dev/docs/install/any-kubernetes-cluster/):

1. Install Custom Resource Definitions:
```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.15.0/serving-crds.yaml
```
2. Install Serving core component:
```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.15.0/serving-core.yaml
```
3. To make sure that Knative has installed correctly in your cluster, rin the following commands:
```
kubectl get pods --namespace knative-serving
```
It should report something like this:
```
NAME                                READY   STATUS    RESTARTS   AGE
activator-7fff689bcb-sc2xf          2/2     Running   5          5d23h
autoscaler-5bcff95856-jdxs6         2/2     Running   6          5d23h
controller-bbdd78bc4-sr5t6          2/2     Running   10         5d23h
istio-webhook-7578df5df5-gl5d2      2/2     Running   8          5d23h
networking-istio-8444c7c995-ksxd6   1/1     Running   2          5d23h
webhook-68bb66b676-wxgf4            2/2     Running   9          5d23h
```

Now we have the basic component Knative setup, we need now to install the networking layer using Istio.

&nbsp;

#### Setting up Istio
Knative uses [Istio](https://istio.io/) as newtwork layer facilitator, which means in theory you are not bound to it. However for the sake of this blog, we will stick to Istio. To setup, we need to execute the following steps:

1. Download the latest release of Istio using this [guide](https://istio.io/latest/docs/setup/getting-started/#download). You will need to make sure to have `istioctl` installed into your `PATH` variable.

2. We will install Istio using a default profile:
```
istioctl install --set profile=default
```

3. We need to make sure that istio will inject Envoy sidecar automatically, we need to the following label our namespace (change `default` namespace with yours):
```
kubectl label namespace default istio-injection=enabled
```

4. Also we will need to enable the automatic side-car injection on our `knative-serving` system namespace:
```
kubectl label namespace knative-serving istio-injection=enabled
```

5. Let's verify everything using this command:
```
kubectl get pods --namespace istio-system
```
We should see the core pods up and running like the following, note this maybe different depending on the Istio components you chose to install:

```
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-74dc798895-2x5rp                1/1     Running   2          5d23h
istio-egressgateway-6b9fff7b95-dnpxn    1/1     Running   2          5d23h
istio-ingressgateway-5f5b59696c-rwkkw   1/1     Running   2          5d23h
istio-tracing-8584b4d7f9-nqncf          1/1     Running   2          5d23h
istiod-69bbcb65c-g9dfp                  1/1     Running   2          5d23h
kiali-6f457f5964-jshp8                  1/1     Running   2          5d23h
```

&nbsp;

#### Configure DNS
In this blog, we would like to setup our cluster in such way that we can access our deployed Camel routes publicly for development purposes. The route URL will be in this format:
```
{APP_NAME}.{NAMESPACE_NAME}.{DNS_ZONE_URL}
```

To do this:

1. We will need to make sure to have [HTTP application routing](https://docs.microsoft.com/en-us/azure/aks/http-application-routing) enabled in our cluster. 

2. We will need to obtain the External IP for the istio ingressgateway. We can use the following command: 
```
kubectl --namespace istio-system get service istio-ingressgateway
```
It will show something similar to this:
```
NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.0.62.123   51.663.129.394   15020:31360/TCP,80:30538/TCP,443:32235/TCP,31400:31897/TCP,15443:32677/TCP   5d23h
```

In the above command, note the external IP `51.663.129.394`. 

3. Login to your Azure Portal, by enabling HTTP application routing in Azure, it will create for us a DNS zone as well loadbalancer. Search for `DNS Zone`. You will find an entry similar to this:
```
f54545e10oos66652.germanywestcentral.aksapp.io
``` 
This would be our suffix custom domain for the routes URLs. We will explain later how the final URL will look like.

4. Clicking on the above DNS Zone, you should see an entry to configure record DNS records. Click on `+ Record Set` and configure the following:

Name: `*` \
Type: `A` \
Alias record set: `No` \
IP Address: `51.663.129.394`

This will pipeline all requests to our istio ingress.

5. Now we need to tell Knative about our custom domain that we just obtained previously from Azure through this command:
```
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"f54545e10oos66652.germanywestcentral.aksapp.io":""}}'
```

After all these steps, whenever you deploy an app to knative, you should be able to access in url of this format `{APP_NAME}.{NAMESPACE_NAME}.{DNS_ZONE_URL}`. For example running command `kubectl get ksvc -n default` should gives list of deployed application in `default` namespace with the URL to access. Example:
```
NAME                     URL                                                                                       LATESTCREATED               LATESTREADY                 READY   REASON
camel-quarkus-rest       http://camel-quarkus-rest.default.f54545e10oos66652.germanywestcentral.aksapp.io       camel-quarkus-rest-v1       camel-quarkus-rest-v1       True
```

Now we can move on to Camel and deploy an example route to our newly cluster.

&nbsp;

#### Building and Deploying a Camel Quarkus Route

1. As we want to build our route natively, you will need first to setup GraalVM correctly, this includes setting `native-image` generator. Take a look at this guide [here](https://quarkus.io/guides/building-native-image).

2. Clone [this example route](https://github.com/omarsmak/camel-talk-examples/tree/master/camel-quarkus-rest) and build it using ` ../mvnw clean package`.

3. Since we are targeting Knative, we will need to build our native route with `Docker` runtime set through this command:
```
./mvnw package -Pnative -Dquarkus.native.container-runtime=docker .
```
4. Once we have our build done, we will build our Docker image with the provided `Dockerfile` using this command, you will need exchange `{USERNAME}` with your Dockerhub username:
```
docker build -f Dockerfile.native -t {USERNAME}/camel-quarkus-rest .
```
5. Push the generated Docker image to your Docker hub via this command, you will need exchange `{USERNAME}` with your Dockerhub username:
```
docker push {USERNAME}/camel-quarkus-rest.
```
6. This example comes with a [`service.yml`](https://github.com/omarsmak/camel-talk-examples/blob/master/camel-quarkus-rest/service.yaml) file which is responsible for deploying our route to Knative. The file looks like this:
```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: camel-quarkus-rest
  namespace: default
spec:
  template:
    metadata:
      name: camel-quarkus-rest-v1
    spec:
      containers:
        - image: docker.io/{USERNAME}/camel-quarkus-rest
  traffic:
    - tag: latest
      latestRevision: true
      percent: 100
```

As above, `metadata` block contains our route name in Knative. In `spec` block, we will Knative to fetch our app from a specific docker image. However, you will need to exchange `{USERNAME}` with your Docker hub username. Also, notice our suffix in the metadata name `camel-quarkus-rest-v1`, `v1` here is used for our deployment revision tagging. For example, if we deploy a `v2` of this app, we will need to change revision name to `camel-quarkus-rest-v2`. This is can used for in the traffic splitting or blue/green deployment, for instance, we can tell Knative to serve 70% of the traffic with `v1` and 30% of the traffic with `v2` using the `traffic` block: 
```
traffic:
    - tag: v2
      percent: 30
    - tag: v1
      percent: 70  
``` 
However, there many great ways that Knative could do this, you may refer to this [documentation](https://knative.dev/docs/serving/samples/blue-green-deployment/) for more detailed explanation on how to use it.

7. Once you have `service.yml` ready, now we can deploy our route using `kubectl apply --filename service.yaml`. 

8. To access our application, we run this command `kubectl get ksvc -n default` which should give us the URLs for our deployed applications in `default` namespace. It will give something like this:
```
NAME                     URL                                                                                       LATESTCREATED               LATESTREADY                 READY   REASON
camel-quarkus-rest       http://camel-quarkus-rest.default.f54545e10oos66652.germanywestcentral.aksapp.io       camel-quarkus-rest-v1       camel-quarkus-rest-v1       True
```
Notice the URL, head off to your browser or any cURL/REST utility and:
```
curl http://camel-quarkus-rest.default.f54545e10oos66652.germanywestcentral.aksapp.io/hello\?q\=awesome%20Camel                                                                                                                                       
Text is: awesome Camel%
```
**And viola!** \
 Now you can deploy any Camel routes or applications with ease and utilize all the good benefits from Knative. 

&nbsp;

 #### Monitoring 

 Monitoring and observability are important for any Cloud environment. Lucky, Knative include all the needed batteries to have your monitoring environment set in just few commands. 
 
1. We need to install the core of the observability features using this command:
```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.15.0/monitoring-core.yaml
```
2. Install Prometheus and Grafana for metrics:
```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.15.0/monitoring-metrics-prometheus.yaml
```
3. Install the ELK stack (Elasticsearch, Logstash and Kibana) for logs:
```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.15.0/monitoring-logs-elasticsearch.yaml
```
4. [To open Grafana](https://knative.dev/docs/serving/accessing-metrics/):
```
kubectl port-forward --namespace knative-monitoring \
$(kubectl get pods --namespace knative-monitoring \
--selector=app=grafana --output=jsonpath="{.items..metadata.name}") \
3000
```
The Grafana UI is accessible through: http://localhost:3000

5. [To open Kibana](https://knative.dev/docs/serving/accessing-logs/):
```
kubectl proxy
```
The Kibana UI is accessible through http://localhost:8001/api/v1/namespaces/knative-monitoring/services/kibana-logging/proxy/app/kibana. 

To install more observability features and detailed guide, you can take a look at this [documentation](https://knative.dev/docs/serving/accessing-metrics/)