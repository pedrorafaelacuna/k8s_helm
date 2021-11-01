# k8s_helm

This post is intended to give a small starting guide for the installation of a Nginx Controller and a Cert Manager. 

![image-20211030092144569](./doc/nginx_k8s.png)



In k8s environments we usually need an Api Manager as well as a service endpoint controller for the deployment of the microservices. In order for these to be accessible through endpoints, we will need a piece like Nginx (ingress controller).

On the other hand, and to be able to access the microservices in a safe way through the https (secure) protocol, a piece like Cert Manager will be necessary. This service will be in charge of generating and signing certificates through a tool such as *letsencrypt*.  

As a requirement to install the following components, it is necessary to have Helm version 3.

### Nginx setup

First of all, we need to check if we have any namespace within our k8s dedicated to host this ingress controller / api manager. If this is not the case, it is highly recommended to create a namespace with the desired name (e.g. ingress, api or api-manager). In order to do so, we will simply perform the following command: 

```
#Create Namespace
$ kubectl create namespace ingress
namespace/ingress created
```

With this, we will have our namespace created and prepared to deploy our ingress-controller. 

Now we will move to the own installation of the ingress-controller process. 
According to the provided scenario, our ingress controller will do the functions that we have described in the introduction, following a pattern like the one presented in the diagram below these lines: 

![image-20211030092144569](./doc/ingress_controller.png)

In this guide, we will install our Nginx in a basic way through Helm's default values for nginx. This case scenario does not require any additional configuration. 

To do this, we must have Helm installed on our server. I will assume that we will launch or install Helm on the remote server, such as an Azure DevOps or Github actions tool.

To perform this Helm installation task, please access the link provided in the Annex. With a Helm version 3 (or higher) installed, we will proceed to install Nginx. 

We need to execute the following steps: 

1- Add the official Nginx Helm repository: 

```
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm repo update
```

2 - Install the Nginx Helm with the default values:   

```
$ helm install ingress-nginx ingress-nginx/ingress-nginx
```

3 - Check that the installation of the ingress has been carried out correctly: 

```
$ kubectl get pods -n ingress
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-57cb5bf694-kb9fk   1/1     Running   0          31s
```

**NOTES:**

In order to properly use our ingress-controller, we must include the ingress section in our microservice. This will allow us to refer the Nginx class to the ingress section and at the same time, the k8s will treat the ingress from this deployed Nginx.

Find attached below these lines an example of ingress that could be configured in the microservices section. Please bear in mind that each parameter should be adapted to the corresponding values.

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: example
  namespace: foo
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - backend:
              serviceName: exampleService
              servicePort: 80
            path: /
  # This section is only required if TLS is to be enabled for the Ingress
  tls:
      - hosts:
          - www.example.com
        secretName: example-tls
```

In this guide we will use *letsencrypt* as a certificate signing tool. 
We must include a certificate in our ingress controller to serve our https endpoints. In order to do so, we must create a secret in the namespace where our microservice is located and add the certificate in the base64-encoded tls.crt and tls.key sections.

```
apiVersion: v1
kind: Secret
metadata:
  name: example-tls
  namespace: foo
data:
  tls.crt: <base64 encoded cert>
  tls.key: <base64 encoded key>
type: kubernetes.io/tls
```

#### Cert Manager (Let's Encrypt) installation

![image-20211030092144569](./doc/cert_manager.png)

Aligned with the previous case scenario, we must know how to create a specific namespace for cert-manager. To do this, we will perform the following command:

```
#Create Namespace
$ kubectl create namespace cert-manager
```

After having the specific namespace created, we must install the Helm repository:

```
#Helm cert-manager repo add
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
```

Finally, we will deploy the Cert manager with the following command, using the default values:

```
#Helm install cert manager
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.3.1 \
  --set installCRDs=true
```

Please note that we have used the Cert manager version v1.6.0.

Once deployed, we will check that the pods are up:

```
#Kubectl status
$ kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-7998c69865-7qgpb              1/1     Running   0          15s
cert-manager-cainjector-7b744d56fb-cgr5b   1/1     Running   0          15s
cert-manager-webhook-7d6d4c78bc-mqtmq      1/1     Running   0          15s
```

At this stage of the process, we have to make a point about Cert manager. To do so, Cert manager requires a cluster-issuer to properly sign the required certificates automatically.

Please find below these lines the *letsencrypt* used in a sample project. 
To properly use it in a a real-case scenario, we should modify and adapt this chart accordingly. 

```
#Helm issuer install
$ helm install --version 0.1.0 -f common.yaml -f sample.yaml -n ingress issuer ./helm
```

The common.yaml and sample.yaml files will be created as follows:

```
acme:
  email: "email@email.com"
 
resources:
  requests:
    cpu: 10m
    memory: 32Mi
```

Finally, we will have our Cert-manager and the issuer deployed.

```
acme:
  privateKeySecretRef: sample-project
```



### Annex

[Helm | Installing Helm](https://helm.sh/docs/intro/install/)

### References

[Ingress | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/)

[Ingress Controllers | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

