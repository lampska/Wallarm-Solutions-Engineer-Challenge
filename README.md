# Wallarm solution engineer challenge - solution

This README documents the steps taken to complete Wallarm technical solution architecture assignment. Additional troubleshooting.md is provided for further troubleshooting & debugging details.

# Summary

Local environment setup:
- Kubernetes (k8s), helm, nginx k8s ingress controller
- Backend vulnerable API https://github.com/erev0s/VAmPI
- Wallarm deployed as k8s nginx ingress controller chainer
- GoTestWaf deployed and run as docker container

Files:

- `troubleshooting.md` - Detailed troubleshooting of encountered issues
- `k8s/vampi-namespace.yaml` - k8s namespace for the app (`vampi-app`).
- `k8s/vampi-deployment.yaml` - k8s deployment for the VAmPI container.
- `k8s/vampi-service.yaml` - k8s clusterIP Service exposing the pod port.
- `k8s/vampi-ingress.yaml` - k8s nginx ingress resource

Decisions
- TLS not setup

# Detailed walkthrough

## K8s (kubernetes) environment setup

### MacOS

Easiest way:
- Install Docker Desktop for Mac https://docs.docker.com/desktop/setup/install/mac-install/
- Enable single node kubernetes cluster in Docker Desktop UI
- Install homebrew https://brew.sh/
- Install helm using terminal: ```brew install helm```


Verification & version information

Helm vetsion 3.19

```bash
helm version

version.BuildInfo{Version:"v3.19.0", GitCommit:"3d8990f0836691f0229297773f3524598f46bda6", GitTreeState:"clean", GoVersion:"go1.25.1"}
```

Docker version 28.5.1

```bash
docker -v

Docker version 28.5.1, build e180ab8
```

K8s version 1.34.1

```bash
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

## Deploy k8s nginx-ingress controller

Install nginx-ingress controller using helm:

```bash
# Add repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install into namespace ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace

# Check installed version; use --versions to see all available
helm search repo ingress-nginx/ingress-nginx
NAME                       	CHART VERSION	APP VERSION	DESCRIPTION                                       
ingress-nginx/ingress-nginx	4.13.3       	1.13.3     	Ingress controller for Kubernetes using NGINX a...
```

Verification -- please note nginx 404 error code is expected at this phase as there are no backend apps/apis running & routes yet. We are simply verifying the ingress is up and working.

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

If the test is not successful, check the following:
- firewall could prevent connection to local port 80
- another network service might be already bound to port 80
- security policy is preventing binding nginx-ingress to port 80

Full documentation, including how to change ports, is available at https://kubernetes.github.io/ingress-nginx. In case of changing ports, further steps need to be appropriately adjusted.

## Deploy target API app (VAmPI) and ingress routing to k8s

Target practice api/app is https://github.com/erev0s/VAmPI. VAmPI is intentionally vulnerable ala WebGoat, and it is meant for vulnerability scanning and security control efficay testing. It also comes with OpenAPI spec. For plain docker instructions, please refer to the VAmPI Github readme.

Steps:
- Create k8s namespace vampi-app.
- Deploy VAmPI pod & service (using docker image latest tag)
- Deploy nginx ingress rules to route host:80/ to VAmPI  

```bash
kubectl apply -f k8s/vampi-namespace.yaml
kubectl apply -f k8s/vampi-deployment.yaml
kubectl apply -f k8s/vampi-service.yaml
kubectl apply -f k8s/vampi-ingress.yaml
```

Verification (NOTE troubleshooting.md ISSUE #1)

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

Now we are able to reach VAmPI running on k8s thru nginx ingress.

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

Environment is now ready for Wallarm deployment and testing

## Wallarm deployment

The environment offers two primary options to deploy Wallarm in-line (with real-time blocking capability) https://docs.wallarm.com/installation/supported-deployment-options/
- nginx ingress controller
- pod/docker sidecar

Given the environment, nginx ingress controller chaining option was selected. It is a very likely real-world scenario with least deployment friction
- No need to replace existing nginx controller
- No need to deploy resource intensive sidecars
- Works with local setup

The downsides
- East-west (kubernetes service-to-service) communication is not secured.
- Chaining multiple controllers can introduce additional latency

Relevant documentation
- https://docs.wallarm.com/installation/inline/kubernetes/nginx-ingress-controller/
- https://docs.wallarm.com/admin-en/chaining-wallarm-and-other-ingress-controllers/
- https://docs.wallarm.com/admin-en/configure-kubernetes-en/#controllerwallarmexistingsecret

### Wallarm API token

Login and create a "node deployment token" at https://us1.my.wallarm.com/settings/api-tokens. Save the token securely.

![Wallarm api token](screenshots/wallarm-api-token.png "Wallarm api token")

### Wallarm k8s namespace, secret, nginx controller

1. Create Wallarm namespace.

```bash
kubectl create namespace wallarm
```

1. Create Wallarm token secret 'wallarm-api-token' (replace WALLARM_NODE_TOKEN with the actual token) 

```bash
kubectl -n wallarm create secret generic wallarm-api-token --from-literal=token=<WALLARM_NODE_TOKEN>
```

1. Add Wallarm helm charts repo

```bash
helm repo add wallarm https://charts.wallarm.com
helm repo update
```

1. Configure k8s/wallarm-values.yaml and install to wallarm namespace

```bash
helm install --version 6.6.2 internal-ingress wallarm/wallarm-ingress -n wallarm -f k8s/wallarm-values.yaml
```

1. Verification

```bash
kubectl get all -n wallarm 
NAME                                                               READY   STATUS    RESTARTS   AGE
pod/internal-ingress-wallarm-ingress-controller-86f7648cd5-hxvdl   1/1     Running   0          2m34s

NAME                                                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/internal-ingress-wallarm-ingress-controller             ClusterIP   10.102.108.224   <none>        80/TCP,443/TCP   2m34s
service/internal-ingress-wallarm-ingress-controller-admission   ClusterIP   10.96.42.132     <none>        443/TCP          2m34s

NAME                                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/internal-ingress-wallarm-ingress-controller   1/1     1            1           2m34s

NAME                                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/internal-ingress-wallarm-ingress-controller-86f7648cd5   1         1         1       2m34s
```

### Routing traffic via wallarm nginx ingress

This step enables wallarm in-line monitoring and blocking. The ingress routing is modified from

```
nginx-ingress -> vampi-service
```

to

```
nginx-ingress -> wallarm-ingress -> vampi-service
```

1. Configure and deploy k8s/vampi-wallarm-ingress-object.yaml

```bash
kubectl apply -f k8s/vampi-wallarm-ingress-object.yaml
```

(NOTE: Issue #2 namespace bug in troubleshooting.md)

1. Get the wallarm controller service name (internal-ingress-wallarm-ingress-controller)

```bash
kubectl get svc -l "app.kubernetes.io/component=controller" -n wallarm -o=jsonpath='{.items[0].metadata.name}'
```

1. Configure 'k8s/vampi-wallarm-ingress-forward.yaml' 

Notably, make sure in 'k8s/vampi-wallarm-ingress-object.yaml'
- 'spec.ingressClassName' must match the ingressClassName in 'k8s/vampi-ingress.yaml'
- 'spec.rules[0].http.paths[0].backend.service.name' must match the wallarm controller service name

```yaml
spec:
  # must match nginx ingress controller class name
  ingressClassName: nginx
  rules:
    - host: localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                # must match wallarm ingress controller name installed via helm
                name: internal-ingress-wallarm-ingress-controller
                port:
                  number: 80
```

1. Delete the original nginx-ingress (NOTE Issue #3 in troubleshooting.md)

```
kubectl delete ingress vampi-ingress -n vampi-app
```

1. Deploy modified nginx-ingress

```bash
kubectl apply -f k8s/vampi-wallarm-ingress-forward.yaml
```

### Enable monitoring

(NOTE Issue #4 in troubleshooting.md)

```bash
kubectl annotate ingress <YOUR_INGRESS_NAME> -n <YOUR_INGRESS_NAMESPACE> nginx.ingress.kubernetes.io/wallarm-mode=monitoring
kubectl annotate ingress <YOUR_INGRESS_NAME> -n <YOUR_INGRESS_NAMESPACE> nginx.ingress.kubernetes.io/wallarm-application="<APPLICATION_ID>"
```