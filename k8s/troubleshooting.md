
## Issue #1 - target api pods not running (wrong port)

While setting up the target VAmPI api with copilot generated yaml, noticed that pods were not ready.


```bash
% kubectl get all -n vampi-app

NAME                                   READY   STATUS    RESTARTS      AGE
pod/vampi-deployment-f7975f895-wpt6r   0/1     Running   2 (30s ago)   2m30s
pod/vampi-deployment-f7975f895-z56sq   0/1     Running   2 (10s ago)   2m30s

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/vampi-service   ClusterIP   10.104.46.79   <none>        80/TCP    2m30s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vampi-deployment   0/2     2            0           2m30s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/vampi-deployment-f7975f895   2         2         0       2m30s
lampska@Mac k8s % curl -v http://localhost:80          
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
< HTTP/1.1 503 Service Temporarily Unavailable
< Date: Wed, 22 Oct 2025 02:21:22 GMT
< Content-Type: text/html
< Content-Length: 190
< Connection: keep-alive
< 
<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body>
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>nginx</center>
</body>
</html>
* Connection #0 to host localhost left intact
```

Basic troubleshooting shows pod is running ok and listening on port 5000

```
kubectl logs -n vampi-app pod/vampi-deployment-f7975f895-wpt6r 

 * Serving Flask app 'config'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://10.1.0.10:5000
Press CTRL+C to quit
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 485-631-255

```

Generated vampi-deployment.yaml had wrong port number. Changed it from 8080 to 5000; then redeploy & verify

```
 <          - containerPort: 5000
 >          - containerPort: 8080
```

Success

```bash
kubectl apply -f vampi-deployment.yaml

kubectl get all -n vampi-app                                   

NAME                                    READY   STATUS    RESTARTS   AGE
pod/vampi-deployment-77f96fff48-7qq9m   1/1     Running   0          69s
pod/vampi-deployment-77f96fff48-shpx6   1/1     Running   0          56s

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/vampi-service   ClusterIP   10.104.46.79   <none>        80/TCP    14m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vampi-deployment   2/2     2            2           14m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/vampi-deployment-77f96fff48   2         2         2       69s
replicaset.apps/vampi-deployment-f7975f895    0         0         0       14m

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
