## Monitoring your NGINX Plus-based Ingress controller in Kubernetes   

This setup is going to walk you through how to install several tools to monitor your NGINX Ingress controller in your Kubernetes cluster. Some of these instructions also apply to the open source version (OSS), and will be designated accordingly. [Read this blog](https://www.nginx.com/blog/wait-which-nginx-ingress-controller-kubernetes-am-i-using/) if you’re not sure which version you’re using. For this setup, we are going to use the following tools to collect metrics:


- [Prometheus](#prometheus): Deploy Prometheus into cluster using Helm (also applies to NGINX OSS)
- [Grafana](#grafana): Deploy Grafana into cluster using Helm (also applies to NGINX OSS)
- [NGINX](#nginx): Deploy and configure the NGINX Plus version of NGINX Ingress Controller


To keep things simple for this demo, we are going to use `Helm` to install everything into our cluster. This keeps the setup easier to deploy and easier to manager. Quick note, you can also install using `manifests` method or `operator` method. It will come down to personal preference or specific requirements when installing software into Kubernetes.   

NOTE: This assumes you have a Kubernetes cluster up and running. It can be on-premises deployment a cloud deployment or a development type environment like `minikube` or other tools. Just keep in mind, the following method below is using GKE (Google Kubernetes Engine).


## Prometheus

The first step will be to deploy `prometheus` into our cluster. Below are the steps to install it using Helm.  
### Prometheus install and notes using Helm

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics
helm repo update
```
Once the repos have been added to Helm, the next step will in this lab is to deploy a `release`. For the demo, I created `nginx-test`.   
### Install Prometheus chart (NOTE: RELEASE_NAME is the name you will provide)
```
helm install [RELEASE_NAME] prometheus-community/prometheus
```
## Grafana

Next step will be to setup Grafana and deploy grafana into Kubernetes.    
### Setup Grafana using Helm
```
helm install prometheus prometheus-community/prometheus -n monitor
helm repo add grafana https://grafana.github.io/helm-charts
```

The Grafana repo is added via Helm. Next we install Grafana using the below command.   
### Install Grafana
```
helm install nginx-logs grafana/grafana
```

If you want to check the status of your helm installs, you can run `helm ls -A` which will show all helm deployments across all `namespaces` in the cluster.   


## NGINX
Now we install NGINX Plus Ingress controller. Make sure you follow the [directions from our documentation page](https://docs.nginx.com/nginx-ingress-controller/installation/building-ingress-controller-image) on how to build your NGINX Plus container image to be used as your Ingress controller. 

Once you have created your custom image and deployed it to your private registry, proceed to the next section.   
### Installing NGINX Plus Ingress steps:
Once the cluster is up, create the nginx-ingress `namespace`. This is where are going to deploy the `nginx-ingress controller`. The choice of a namespace is up to you. I chose `nginx-ingress` for our demo.

#### create the namespace.   
```
kubectl create ns nginx-ingress
```

### Now we are going to deploy our NGINX Plus Ingress controller using helm.
```
helm install nginx-test -n nginx-ingress -f values-plus.yaml .  
```

In the above command, we pass our `release-name` to the install command (**nginx-test**), the `-n` is the namespace we are deploying our Ingress controller to. We pass in the `values-plus.yaml` file which holds specific settings we have configured for our NGINX Plus Ingress controller. NOTE. Do not forget to put the period at the very end of the command.

Here is the example of my `values-plus.yaml` file that was used in this demo:

### Contents of values-plus.yaml file
```
controller:
  name: nginx-plus
  nginxplus: true
  image:
    repository: <> ### Use your registry
    tag: "1.10"
    pull.Policy: IfNotPresent
  kind: deployment
  serviceAccount:
    name: nginx-ingress
    imagePullSecretName: regcred
  service:
    name: nginx-ingress
  prometheus:
    create: true
    port: 9113
```
> You will notice above that we have automatically enabled NGINX Plus Ingress Controller to have port 9113 open, enabling `Prometheus` to scrap metrics and send to `Grafana`.
> 
Additionally for the NGINX Plus dashboard, by default the API and status port are enabled when using helm. The status port is 8080. Here are the specific helm settings you can change if you need to.
```
controller.nginxStatus.enable	Enable the NGINX stub_status, or the NGINX Plus API.
controller.nginxStatus.port	Set the port where the NGINX stub_status or the NGINX Plus API is exposed.	
```
### More Helm chart values can be located here:
[NGINX Plus Ingress Controller Helm Chart Values](https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments/helm-chart)

### Demo setup files
> For our demo environment, we are using [VirtualServer and VirtualServerRoute](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/). This allows use to easily do cross-namespace routing in Kubernetes when your applications live in different namespaces.

### Here is our VirtualServer file
```
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: grafana
spec:
  host: grafana.example.com
  tls:
    secret: graf-secret
  routes:
  - path: /
    route: monitor/grafana-dash
```

### Here is our VirtualServerRoute File
```
apiVersion: k8s.nginx.org/v1
kind: VirtualServerRoute
metadata:
  name: grafana-dash
  namespace: monitor
spec:
  host: grafana.example.com
  upstreams:
  - name: grafana 
    service: nginx-logs-grafana
    port: 80
  subroutes:
  - path: / 
    action:
      pass: grafana
```
> Since we are routing to our Grafana server that lives in a different namespace (**monitor** in this demo), we specify that in our **VirtualServer/VirtualServerRoute** configuration.


> Below is the output of the Grafana service running in Kubernetes, that NGINX Plus Ingress controller is routing the requests to. Equivalent command is `kubectl get svc nginx-logs-grafana -n monitor`.


### Grafana service
```
Name:              nginx-logs-grafana
Namespace:         monitor
Labels:            app.kubernetes.io/instance=nginx-logs
                   app.kubernetes.io/managed-by=Helm
                   app.kubernetes.io/name=grafana
                   app.kubernetes.io/version=7.4.2
                   helm.sh/chart=grafana-6.4.5
Annotations:       cloud.google.com/neg: {"ingress":true}
                   meta.helm.sh/release-name: nginx-logs
                   meta.helm.sh/release-namespace: monitor
Selector:          app.kubernetes.io/instance=nginx-logs,app.kubernetes.io/name=grafana
Type:              ClusterIP
IP Families:       <none>
IP:                10.32.13.208
IPs:               <none>
Port:              service  80/TCP
TargetPort:        3000/TCP
Endpoints:         10.28.0.5:3000
Session Affinity:  None
Events:            <none>
```
