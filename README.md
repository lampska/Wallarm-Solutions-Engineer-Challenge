# Wallarm solution engineer challenge - solution

### Dependencies and files

Environment Setup:
- docker, kubernetes (k8s), helm, nginx k8s ingress controller
- https://github.com/erev0s/VAmPI

Files:

- `troubleshooting.md` - Detailed troubleshooting of encountered issues
- `k8s/vampi-namespace.yaml` - k8s namespace for the app (`vampi-app`).
- `k8s/vampi-deployment.yaml` - k8s deployment for the VAmPI container.
- `k8s/vampi-service.yaml` - k8s clusterIP Service exposing the pod port.
- `k8s/vampi-ingress.yaml` - k8s nginx ingress resource

# Detailed walkthru

## Environment setup

### MacOS

- Install Docker Desktop for Mac https://docs.docker.com/desktop/setup/install/mac-install/
- Enable single node kubernetes cluster in Docker Desktop
- Install homebrew https://brew.sh/
- Install helm: ```brew install helm```


Verification & version information

```bash
helm version

version.BuildInfo{Version:"v3.19.0", GitCommit:"3d8990f0836691f0229297773f3524598f46bda6", GitTreeState:"clean", GoVersion:"go1.25.1"}

docker -v

Docker version 28.5.1, build e180ab8

kubetcl version

Client Version: v1.34.1
Kustomize Version: v5.7.1
Server Version: v1.34.1

kubectl get all -A

NAMESPACE     NAME                                         READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-66bc5c9577-fl25c                 1/1     Running   0          13m
kube-system   pod/coredns-66bc5c9577-gx6gm                 1/1     Running   0          13m
kube-system   pod/etcd-docker-desktop                      1/1     Running   0          13m
kube-system   pod/kube-apiserver-docker-desktop            1/1     Running   0          13m
kube-system   pod/kube-controller-manager-docker-desktop   1/1     Running   0          13m
kube-system   pod/kube-proxy-62m8x                         1/1     Running   0          13m
kube-system   pod/kube-scheduler-docker-desktop            1/1     Running   0          13m
kube-system   pod/storage-provisioner                      1/1     Running   0          13m
kube-system   pod/vpnkit-controller                        1/1     Running   0          13m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  13m
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   13m

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   13m

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   2/2     2            2           13m

NAMESPACE     NAME                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-66bc5c9577   2         2         2       13m


```

### Windows WSL / Ubuntu 24

FIXME: Ubuntu 24 WSL setup

### nginx-ingress controller

Install an nginx-ingress controller (example using Helm):

```bash
# Add repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install into namespace ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

Verification -- please note nginx 404 error code is expected at this phase

```bash
kubectl get service --namespace ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.99.146.215    localhost     80:31993/TCP,443:30535/TCP   119s
ingress-nginx-controller-admission   ClusterIP      10.102.154.113   <none>        443/TCP                      119s

curl -v http://localhost:80

* Host localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to localhost (::1) port 80
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 404 Not Found
< Date: Wed, 22 Oct 2025 02:13:56 GMT
< Content-Type: text/html
< Content-Length: 146
< Connection: keep-alive
< 
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

### Target API app (VAmPI) and routing

For plain docker instructions, please refer to https://github.com/erev0s/VAmPI

This will deploy VAmPI and nginx ingress rules to kubernetes namespace vampi-app

```bash
kubectl apply -f k8s/vampi-namespace.yaml
kubectl apply -f k8s/vampi-deployment.yaml
kubectl apply -f k8s/vampi-service.yaml
kubectl apply -f k8s/vampi-ingress.yaml
```

Verification (NOTE troubleshooting ISSUE #1)

```bash
kubectl get all -n vampi-app
kubectl get svc -n ingress-nginx
kubectl describe ingress -n vampi-app


NAME                                    READY   STATUS    RESTARTS   AGE
pod/vampi-deployment-77f96fff48-7qq9m   1/1     Running   0          7m23s
pod/vampi-deployment-77f96fff48-shpx6   1/1     Running   0          7m10s

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/vampi-service   ClusterIP   10.104.46.79   <none>        80/TCP    20m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vampi-deployment   2/2     2            2           20m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/vampi-deployment-77f96fff48   2         2         2       7m23s
replicaset.apps/vampi-deployment-f7975f895    0         0         0       20m
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.99.146.215    localhost     80:31993/TCP,443:30535/TCP   42m
ingress-nginx-controller-admission   ClusterIP      10.102.154.113   <none>        443/TCP                      42m
Name:             vampi-ingress
Labels:           <none>
Namespace:        vampi-app
Address:          localhost
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  localhost   
              /   vampi-service:80 (10.1.0.11:5000,10.1.0.12:5000)
Annotations:  kubernetes.io/ingress.class: nginx
              nginx.ingress.kubernetes.io/proxy-body-size: 8m
              nginx.ingress.kubernetes.io/ssl-redirect: false
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    20m (x2 over 20m)  nginx-ingress-controller  Scheduled for sync
```

Now we are able to reach VAmPI running on k8s thru nginx ingress


```bash
curl -v http://localhost:80

* Host localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to localhost (::1) port 80
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Wed, 22 Oct 2025 02:32:05 GMT
< Content-Type: application/json
< Content-Length: 271
< Connection: keep-alive
< 
* Connection #0 to host localhost left intact
{ "message": "VAmPI the Vulnerable API", "help": "VAmPI is a vulnerable on purpose API. It was created in order to evaluate the efficiency of third party tools in identifying vulnerabilities in APIs but it can also be used in learning/teaching purposes.", "vulnerable":1}

```
