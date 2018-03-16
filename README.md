# Edgemicro-k8

## Overview

This project allows you to install Apigee Edge Microgateway as a sidecar gateway in front of your services deployed in kubernetes cluster. Developers faces challenges in exposing their microservices and rely on API Management providers for exposing,securing and managing their apis.
This project brings native api management to the microservices development on kubernetes platform.

# Architecture
![Architecture](/docs/images/arch.png)


## Getting Started

###Prerequisites

Kubernetes version 1.9 or greater
Kubernetes CLI kubectl v1.9 or greater

- Minikube - Coming soon ...
- GKE - Tested
- Openshift - Coming soon ...

### Install Edgemicro

Refer [here](https://docs.apigee.com/api-platform/microgateway/2.5.x/installing-edge-microgateway) for more details about installing edgemicro.

You can install edgemicro as follows

```
npm install edgemicro -g
```


### Initialize Containers. 

If you are using GKE, use these commands to initialize containers

```

gcloud container clusters create edge-micro  --cluster-version=1.9.2-gke.1 --zone us-central1-a --project edge-apigee --num-nodes 4 -m n1-standard-2

gcloud container clusters get-credentials edge-micro --zone us-central1-a --project edge-apigee
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)

```


### Automatic sidecar injection


Install the base edgemicro setup. This will create namespaces and cluster roles for edgemicro sidecar.

```
kubectl apply -f install/kubernetes/edgemicro.yaml
```

#### Install the sidecar injection configmap.

```
kubectl apply -f install/kubernetes/edgemicro-sidecar-injector-configmap-release.yaml
```

#### Install Webhook

Webhooks requires a signed cert/key pair. Use install/kubernetes/webhook-create-signed-cert.sh to generate a cert/key pair signed by the Kubernetes’ CA. The resulting cert/key file is stored as a Kubernetes secret for the sidecar injector webhook to consume.

Note: Kubernetes CA approval requires permissions to create and approve CSR. See https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster and install/kubernetes/webhook-create-signed-cert.sh for more information.

```
./install/kubernetes/webhook-create-signed-cert.sh \
    --service edgemicro-sidecar-injector \
    --namespace edgemicro-system \
    --secret sidecar-injector-certs
```


Set the caBundle in the webhook install YAML that the Kubernetes api-server uses to invoke the webhook.

```
cat install/kubernetes/edgemicro-sidecar-injector.yaml | \
     ./install/kubernetes/webhook-patch-ca-bundle.sh > \
     install/kubernetes/edgemicro-sidecar-injector-with-ca-bundle.yaml

```

Install the sidecar injector webhook.

```
kubectl apply -f install/kubernetes/edgemicro-sidecar-injector-with-ca-bundle.yaml

```

The sidecar injector webhook should now be running.

```
kubectl -n edgemicro-system get deployment -ledgemicro=sidecar-injector

NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
edgemicro-sidecar-injector   1         1         1            1           12m

```

The sidecar injector pod should look like 

```
kubectl get pods -n edgemicro-system

NAME                                          READY     STATUS    RESTARTS   AGE
edgemicro-sidecar-injector-78bffbd44b-bct2r   1/1       Running   0          14m
```

#### Deploy helloworld app

```
kubectl apply -f samples/helloworld/helloworld.yaml --namespace=default
kubectl get pods --namespace=default

NAME                          READY     STATUS    RESTARTS   AGE
helloworld-569d6565f9-lwrrv   1/1       Running   0          17m

```
As you can see that helloworld pod came up with  only 1 container. The injection is not yet enabled. 

Delete this deployment

```
kubectl delete -f samples/helloworld/helloworld.yaml --namespace=default
```

#### Edgemicro Configuration Profile 

Creae a edgemicro configuration and associate it with a kubernetes namespace. You can create different configuration profiles based on your apigee edge settings or edge micro configuration settings.

```
./install/kubernetes/webhook-edgemicro-patch.sh
```

You can also pass parameters for non interactive usage. Refer usage instructions.

```
./install/kubernetes/webhook-edgemicro-patch.sh -h
Usage: ./install/kubernetes/webhook-edgemicro-patch.sh [option...]

   -o, --apigee_org           * Apigee Organization.
   -e, --apigee_env           * Apigee Environment.
   -v, --virtual_host         * Virtual Hosts with comma seperated values.The values are like default,secure.
   -t, --private              y,if you are configuring Private Cloud. Default is n.
   -m, --mgmt_url             Management API URL needed if its Private Cloud
   -r, --api_base_path        API Base path needed if its Private Cloud
   -u, --user                 * Apigee Admin Email
   -p, --password             * Apigee Admin Password
   -n, --namespace            Namespace where your application is deployed. Default is default
   -k, --key                  * Edgemicro Key. If not specified it will generate.
   -s, --secret               * Edgemicro Secret. If not specified it will generate.
   -c, --config_file          * Specify the path of org-env-config.yaml. If not specified it will generate in ./install/kubernetes/config directory

```
For ex:
```
./install/kubernetes/webhook-edgemicro-patch.sh -t n -o gaccelerate5 -e test -v default -u <apigee email> -p <apigee-password>  -k <edgemicro key> -s <edgemicro secret> -c "/Users/rajeshmi/.edgemicro/gaccelerate5-test-config.yaml" -n default

```
Run command below to inject the edgemicro config profile in kubernetes.

```
kubectl apply -f install/kubernetes/edgemicro-config-namespace-bundle.yaml
```

#### Enable Injection

NamespaceSelector decides whether to run the webhook on an object based on whether the namespace for that object matches the selector (see https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors). The default webhook configuration uses edgemicro-injection=enabled.

```
kubectl get namespace -L edgemicro-injection

AME               STATUS    AGE       EDGEMICRO-INJECTION
default            Active    1d        
edgemicro-system   Active    1d
kube-public        Active    1d
kube-system        Active    1d
```

Label the default namespace with edgemicro-injection=enabled. In case you configured edgemicro with different namespace, specify your namespace.

```
kubectl label namespace default edgemicro-injection=enabled
kubectl get namespace -L edgemicro-injection

AME               STATUS    AGE       EDGEMICRO-INJECTION
default            Active    1d        enabled
edgemicro-system   Active    1d
kube-public        Active    1d
kube-system        Active    1d

```

#### Deploying the helloworld app with Injection

```
kubectl apply -f samples/helloworld/helloworld.yaml --namespace=default
kubectl get pods --namespace=default

NAME                          READY     STATUS    RESTARTS   AGE
helloworld-569d6565f9-lwrrv   2/2       Running   0          17m

```
As you can see that helloworld pod came up with 2 containers.


#### Accessing the services
```
kubectl get services --namespace=default
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
helloworld   LoadBalancer   10.19.252.245   35.188.24.156   8081:32296/TCP   24m
kubernetes   ClusterIP      10.19.240.1     <none>          443/TCP          1d
```

Wiat till the External IP of Service shows up

```
export GATEWAY_IP=$(kubectl describe services helloworld  --namespace=default | grep "LoadBalancer Ingres" | cut -d ':' -f2 | tr -d "[:space:]")

echo $GATEWAY_IP

echo "Call with no API Key:"
curl $GATEWAY_IP:8081;echo
echo "Call with API Key:"
curl -H 'x-api-key:your-edge-api-key' $GATEWAY_IP:8081;echo
```

### Manual sidecar injection

Coming soon ....

## Deleting the setup
```
kubectl delete -f samples/helloworld/helloworld.yaml --namespace=default
kubectl delete -f install/kubernetes/edgemicro-sidecar-injector-with-ca-bundle.yaml
kubectl -n edgemicro-system delete secret sidecar-injector-certs
kubectl delete csr edgemicro-sidecar-injector.edgemicro-system
kubectl label namespace default edgemicro-injection-
kubectl delete -f install/kubernetes/edgemicro-config-namespace-bundle.yaml
kubectl delete -f install/kubernetes/edgemicro-sidecar-injector-configmap-release.yaml

rm -fr  install/kubernetes/edgemicro-sidecar-injector-with-ca-bundle.yaml
rm -fr  install/kubernetes/config/*config.yaml
rm -fr  install/kubernetes/edgemicro-config-namespace-bundle.yaml

kubectl delete -f install/kubernetes/edgemicro.yaml
gcloud beta container clusters delete edge-micro

```

## References
It uses istio-sidecar-proxy-injector and istio-init docker from istio project.

## License

Apache 2.0 - See [LICENSE](LICENSE) for more information.
