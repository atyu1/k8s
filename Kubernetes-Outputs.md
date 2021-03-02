# Whats this

Some handson kubernetes commands

### Version 

```
master $ kubectl version
Client Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.0", GitCommit:"641856db18352033a0d96dbc99153fa3b27298e5", GitTreeState:"clean", BuildDate:"2019-03-25T15:53:57Z", GoVersion:"go1.12.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.0", GitCommit:"641856db18352033a0d96dbc99153fa3b27298e5", GitTreeState:"clean", BuildDate:"2019-03-25T15:45:25Z", GoVersion:"go1.12.1", Compiler:"gc", Platform:"linux/amd64"}
```

```
master $ kubectl version --short
Client Version: v1.14.0
Server Version: v1.14.0
```

### Cluster info

```
master $ kubectl cluster-info
Kubernetes master is running at https://172.17.0.29:6443
dash-kubernetes-dashboard is running at https://172.17.0.29:6443/api/v1/namespaces/kube-system/services/dash-kubernetes-dashboard:http/proxy
KubeDNS is running at https://172.17.0.29:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### Components

```
master $ kubectl get componentstatus
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

### Nodes

``` 
master $ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   18m   v1.14.0
node01   Ready    <none>   18m   v1.14.0
```

### Deployment

Create yaml file
```
master $ cat echoserver.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: hello
  name: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      run: hello
  template:
    metadata:      labels:
        run: hello
    spec:
      containers:
      - image: k8s.gcr.io/echoserver:1.9
        name: hello
        ports:
        - containerPort: 8080
 ```

Deployy by yaml
```
master $ kubectl create -f myserver.yaml
deployment.extensions/myserver created
``` 

Or create Pod + Replica + Service
```
kubectl run myserver --image=k8s.gcr.io/echoserver:1.9 --generator=run-pod/v1 --port=8080

kubectl scale --replicas=3 deployment/myserver
```

```
master $ kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
myserver   1/1     1            1           12m
```

### Services

```
master $ kubectl get service myserver
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
myserver   NodePort   10.10.123.111   <none>        8080:32323/TCP   10s
```

Expose
```
master $ kubectl expose deployment myserver --type=NodePort
service/myserver exposed
```

Describe
```
master $ kubectl describe service myserver
Name:                     hello
Namespace:                default
Labels:                   run=myserver
Annotations:              <none>
Selector:                 run=myserver
Type:                     NodePort
IP:                       10.10.123.111
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  32323/TCP
Endpoints:                10.20.0.5:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
 
Testing (use curl if no browser is possible)

### PODS

Create by manifest
```
apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  containers:
    - image: gcr.io/kuar-demo/kuard-amd64:blue
      name: kuard
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

Apply Manifest
```
[root@master ~]# kubectl apply -f kuard.yaml 
pod/kuard created
```

Create by command
```
[root@master ~]# kubectl run kuard --image=gcr.io/kuar-demo/kuard-amd64:blue
```

Delete 
```
[root@master ~]# kubectl delete pods/kuard
pod "kuard" deleted
```

or
```
[root@master ~]# kubectl delete -f kuard.yaml 
pod "kuard" deleted
```

Get more details - Describe
```
[root@master ~]# kubectl describe pod/kuard
Name:         kuard
Namespace:    default
Priority:     0
Node:         node1/192.168.18.121
Start Time:   Sat, 04 Jul 2020 14:05:39 +0200
Labels:       <none>
Annotations:  Status:  Running
IP:           10.32.0.3
IPs:
  IP:  10.32.0.3
Containers:
  kuard:
    Container ID:   docker://ab1021d96527ef1b72af0b8185935c75545c5965a35d042f00f776ac61f0454c
    Image:          gcr.io/kuar-demo/kuard-amd64:blue
    Image ID:       docker-pullable://gcr.io/kuar-demo/kuard-amd64@sha256:1ecc9fb2c871302fdb57a25e0c076311b7b352b0a9246d442940ca8fb4efe229
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 04 Jul 2020 14:05:40 +0200
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-b4gkr (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-b4gkr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-b4gkr
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kuard to node1
  Normal  Pulled     9s         kubelet, node1     Container image "gcr.io/kuar-demo/kuard-amd64:blue" already present on machine
  Normal  Created    9s         kubelet, node1     Created container kuard
  Normal  Started    9s         kubelet, node1     Started container kuard
```

Expose Pod via port-forward: (this will block the cursor, while running you can open browser http://localhost:8080)
```
[root@master ~]# kubectl port-forward kuard 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

```

See logs (same as docker logs):
-f  will do like in tail, ongoing log provide
```
[root@master ~]# kubectl logs kuard
2020/07/04 12:09:43 Starting kuard version: v0.10.0-blue
2020/07/04 12:09:43 **********************************************************************
2020/07/04 12:09:43 * WARNING: This server may expose sensitive
2020/07/04 12:09:43 * and secret information. Be careful.
2020/07/04 12:09:43 **********************************************************************
2020/07/04 12:09:43 Config: 
{
  "address": ":8080",
  "debug": false,
  "debug-sitedata-dir": "./sitedata",
  "keygen": {
    "enable": false,
    "exit-code": 0,
    "exit-on-complete": false,
    "memq-queue": "",
    "memq-server": "",
    "num-to-gen": 0,
    "time-to-run": 0
  },
  "liveness": {
    "fail-next": 0
  },
  "readiness": {
    "fail-next": 0
  },
  "tls-address": ":8443",
  "tls-dir": "/tls"
}
2020/07/04 12:09:43 Could not find certificates to serve TLS
2020/07/04 12:09:43 Serving on HTTP on :8080
2020/07/04 12:11:32 127.0.0.1:35212 GET /
2020/07/04 12:11:32 Loading template for index.html
...
```

Get into the container/pod (same as in docker):
```
[root@master ~]# kubectl exec kuard -- date
Sat Jul  4 12:18:38 UTC 2020
```

```
[root@master ~]# kubectl exec kuard -- date
Sat Jul  4 12:18:38 UTC 2020
[root@master ~]# kubectl exec -it kuard -- ash
~ $ hostname
kuard
~ $ exit
```

Deeper health check, add to manifest file:
```
spec:  
   containers:    
   - image: gcr.io/kuar-demo/kuard-amd64:blue      
      name: kuard      
      livenessProbe:        
        httpGet:          
          path: /healthy          
           port: 8080        
          initialDelaySeconds: 5        
        timeoutSeconds: 1        
        periodSeconds: 10        
        failureThreshold: 3
```

###  Volumes

Add volumes via manifest:
```
apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  volumes:
    - name: "kuard-data"
      hostPath:
        path: "/var/lib/kuard"
  containers:
    - image: gcr.io/kuar-demo/kuard-amd64:blue
      name: kuard
      volumeMounts:
        - mountPath: "/data"
          name: "kuard-data"
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

### Labels
Create and Get:
```
[root@master ~]# kubectl run learn-prod \  --image=gcr.io/kuar-demo/kuard-amd64:blue \  --replicas=2 \  --labels="ver=1,app=kubernetes-learning,env=prod"
Flag --replicas has been deprecated, has no effect and will be removed in the future.
pod/learn-prod created
[root@master ~]# kubectl run learn-test \  --image=gcr.io/kuar-demo/kuard-amd64:blue \  --replicas=2 \  --labels="ver=1,app=kubernetes-learning,env=test"
Flag --replicas has been deprecated, has no effect and will be removed in the future.
pod/learn-test created
[root@master ~]# 
[root@master ~]# kubectl get pods --show-labels
NAME                            READY   STATUS              RESTARTS   AGE   LABELS
learn-prod                      0/1     CrashLoopBackOff    2          49s   app=kubernetes-learning,env=prod,ver=1
learn-test                      0/1     RunContainerError   2          38s   app=kubernetes-learning,env=test,ver=1
[root@master ~]# 
```

Show label as column
```
[root@master ~]# kubectl get pods -L app -L env
NAME         READY   STATUS              RESTARTS   AGE     APP                   ENV
learn-prod   1/1     Running             0          6m43s   kubernetes-learning   prod
learn-test   0/1     ContainerCreating   0          7m10s   kubernetes-learning   test

```

Get only based on labels:
```
[root@master ~]# kubectl get pods --selector="env=prod"
NAME         READY   STATUS    RESTARTS   AGE
learn-prod   1/1     Running   0          7m39s
```
