# Modern Apps Demo

This repository include files and instructions to reproduce the Modern Applications NGINX demo presented in the following Video

[![IMAGE ALT TEXT HERE](https://yt-embed.herokuapp.com/embed?v=CFcLVm8htI8)](https://www.youtube.com/watch?v=CFcLVm8htI8)


### Deploy Arcadia Finance application
```shell
kubectl apply -f Arcadia/
```
```shell
$ kubectl get deployments.apps
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
arcadia-app2      1/1     1            1           100s
arcadia-app3      1/1     1            1           100s
arcadia-backend   1/1     1            1           100s
arcadia-main      1/1     1            1           100s
```
```shell
$ kubectl get svc
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
app2                 ClusterIP   10.101.108.206   <none>        80/TCP    106s
app3                 ClusterIP   10.107.185.140   <none>        80/TCP    106s
backend              ClusterIP   10.105.169.42    <none>        80/TCP    106s
main                 ClusterIP   10.106.77.152    <none>        80/TCP    106s
```

### Install NGINX+ Ingress Controller

The following instructions explains how to use the NGINX Plus Ingress Controller image from the F5 Docker registry in your Kubernetes cluster by using your NGINX Ingress Controller subscription JWT token. Please note that an NGINX Plus subscription certificate and key will not work with the F5 Docker registry.

```shell
$ helm repo add nginx-stable https://helm.nginx.com/stable
$ helm repo update
```
Create a docker-registry secret on the cluster using the JWT token as the username, and none for password (password is unused)
```shell
kubectl create secret docker-registry mysecret --docker-server=private-registry.nginx.com --docker-username=<JWT Token> --docker-password=none
```

To install the chart with the release name my-ingress (my-ingress is the name that you choose):
```shell
$ helm install my-ingress nginx-stable/nginx-ingress \
--set controller.nginxplus=true \
--set controller.appprotect.enable=true \
--set controller.image.repository=private-registry.nginx.com/nginx-ic-nap/nginx-plus-ingress \
--set controller.serviceAccount.imagePullSecretName=mysecret \
--set controller.service.type=NodePort \
--set controller.enablePreviewPolicies=true \
--set controller.prometheus.create=true \
--set controller.prometheus.port=9113 \
--set controller.ingressClass=ingressclass1
```
```shell
$ helm list
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
my-ingress	default  	1       	2021-11-10 14:25:59.573764945 +0400 +04	deployed	nginx-ingress-0.11.3	2.0.3
```
```shell
$ kubectl get deployments.apps -l app.kubernetes.io/managed-by=Helm
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
my-ingress-nginx-ingress   1/1     1            1           113s
```

### Create Ingress Resources

#### Standard Kuberenets Ingress Resources

```shell
$ kubectl apply -f basic-ingress.yaml
ingress.networking.k8s.io/arcadia created
```
```shell
$ kubectl describe ingress arcadia
Name:             arcadia
Namespace:        default
Address:
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                    Path  Backends
  ----                    ----  --------
  www.arcadia-finance.io
                          /        main:80 (10.244.0.81:8080)
                          /files   backend:80 (10.244.0.83:8080)
                          /api/    app2:80 (10.244.0.82:8080)
                          /app3/   app3:80 (10.244.0.84:8080)
Annotations:              <none>
```

#### NGINX CRD (Custom Resource Definitions) Resources

```shell
$ kubectl apply -f advance-ingress.yaml
virtualserver.k8s.nginx.org/arcadia created
```
```shell
$ kubectl get virtualserver
NAME      STATE   HOST                     IP    PORTS   AGE
arcadia   Valid   www.arcadia-finance.io                 10s
```

## Deploying Prometheus and Grafana in Kubernetes

This is based on kube-prometheus stack that is meant for cluster monitoring, so it is pre-configured to collect metrics from all Kubernetes components. In addition to that it delivers a default set of dashboards and alerting rules.


```shell
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus
kubectl create -f manifests/setup
kubectl create -f manifests/
```

It should look like this when done:

```shell
$ kubectl get pods -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
alertmanager-main-0                   2/2     Running   0          58s
blackbox-exporter-6798fb5bb4-q9lgr    3/3     Running   0          58s
grafana-767fcb6796-cb8td              1/1     Running   0          57s
kube-state-metrics-6c699dfb8-wdhh4    3/3     Running   0          57s
node-exporter-fhc7r                   2/2     Running   0          56s
prometheus-adapter-7dc46dd46d-v2nng   1/1     Running   0          56s
prometheus-operator-5c875b748-tmjr2   2/2     Running   0          63s
```
```shell
$ kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.99.18.245     <none>        9093/TCP,8080/TCP            2m14s
alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   2m14s
blackbox-exporter       ClusterIP   10.109.162.204   <none>        9115/TCP,19115/TCP           2m14s
grafana                 ClusterIP   10.108.185.104   <none>        3000/TCP                     2m13s
kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            2m13s
node-exporter           ClusterIP   None             <none>        9100/TCP                     2m12s
prometheus-adapter      ClusterIP   10.98.191.114    <none>        443/TCP                      2m12s
prometheus-k8s          ClusterIP   10.103.58.31     <none>        9090/TCP,8080/TCP            2m12s
prometheus-operator     ClusterIP   None             <none>        8443/TCP                     2m19s
```

### Access Prometheus and Grafana dashboards

An easy way to access Prometheus and Grafana dashboards is by using kubectl proxy once all the services are running:
```shell
kubectl --namespace monitoring port-forward svc/grafana 3000
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
```
And web console is accessible through the URLs:
- http://localhost:3000
- http://localhost:9090

Grafana credentials:
- Username: admin
- Password: admin

Grafana dashboard for NGINX Ingress controller is available at:
https://grafana.com/grafana/dashboards/9614

