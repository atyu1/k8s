# CKA prep

## Installation
- Disable swap!

`All Nodes prepare:`
- Disable Firewall
- Disable Swap
- Edit /etc/hosts
- Set selinux to permisive
- Install, Enable and Start: kubeadm, kubelet kubectl
- Add iptables bridging:

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

`On Workers:`
- Install, enable and start Docker-CE

`On Control:`
- kubeadm init (run as root)
- Copy the config file based on output after init

`On Workers:`
- kubeadm join based on output after init (run as root)

## Network Addons
- Install addons to have Pod-to-Pod connectivity

## RBAC checks
- kubectl auth can-i <what> <where>
- kubectl auth can-i create pod (answer yes no)

## Structure

`General Notes`
Pod = Container + IP + Volume
Deployment = Pod + Replica + Upgrades

Service = Connection between Deployments + LB
End User connects to service

PV = Persistent Volume
PVC = Persistent Volume Control

Pod = Can connect to PVC -> Connects to PV

`API`
etcd — communnicates —> api-server (TLS) 
curl directly -------> api- server (requires to have TLS on curl - complicated)
kube-proxy — create secure connection —-> api-server
curl ——communicates to ——> kube-proxy

## Accessing the API
- kubectl api-resources = shows api groups and resources
- kubectl api-versions = find versions to create for yaml files
- kubectl explain <subject> = to see, how to specify the yaml file

## Bash completion
- Install bash-completion
- Enable for kubectl and redirect to either:
        - kubectl completion bash > /etc/completion.d/kubectl. (root needed, for all users)
        - kubectl completion bash >> ~/.bashrc. (per user)

## Curl
- Curl wit TLS example: curl —cert test.pem —key test-key.pem —cacert /root/testca.pem https://control:6443/api/v1
- Curl via proxy:
    - kubectl proxy —port=8001 &
    - curl http://localhost:8001/api/v1/namespaces/default/pods/busybox2

## etcdctl
- manage etcd database
- etcdctl2 (for v2) or etcdctl (for v1)

## Namespaces
- Dedicated container - for multi user deployments
- Apply quotas and restrictions
- RBAC
- kubectl get ns
- kubectl describe ns

## Deployments 
- Keep track of pods and replicas
- Create by: kubectl create deployment —image=<name>
- Kubectl describe with -o yaml, can help to create base yaml

## Replicas
- Create replicaSet to specify the number of pods to be schedulled
- kubectl replica —replication=3 <deployment>

## Tags
- Important to identify pods, deployments, services
- use —show-labels to see in output for get all
- filter by tag using —selector

## Rolling upgrades
- Specify in deployment as rolling update type
- Creates new replicasets
- It will run the same amount of Pods in new replica and it will shift active to new upgraded
- two options are importatn:
    - maxSurge - how many we can have temporary more then replicaSet - 3 pods, max surge 1 -> during upgrade we can see 4 PODs active for shifting
    - maxUnavailable - how much we can have unavailable
- check explain deployments.spec.strategy

## Rolout upgrades
- Deployments can be reverted
- kubectl rollout histor deployment
- kubectl edit deployments <name> - change version of image
- kubectl get all, there will be new and all versions and slowly, old replicasets will be removed and new replicaset will fillup
- After check: kubectl rollout history deployment rolling <name> (2 revisions appear)
- To revert to 1st version run: kubectl rollout undo deployments —revision=1

## Init Containers
Used in deployments, for first spinning up, if all good then it will deploy container from containers section (some pre-check if all is good)
```
spec:
  containers:
  …
  initContainers:
  - name: init-service
     image: myservice
     ….
```

## StatefullSets
- aka deployments but ensure uniqness and order of Pods
- used if it has unique network identifiers or stable persistent storage
- also used in: ordered deployment, scaling and rolling updates
- storage must be Persistent Volume
- Delete will not delete the storage
- We define first Headless Service to guarantee identity
https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

## DeamonSet
- Ensure that all or some nodes run a copy of a Pod
- Mostly for services, which should run everywhere

## Volumes
- Keep data out of container so we don't loose it when Pod is destroyed
- Type of volumes:
    - PVC (persistent volume claim) -> PV (NFS, CFS, …)
    - ConfigMap (used to replace variables in configs)
    - Secret (Special configMap for passwords)

- PV = it has info about volume, how to connect, where it is …
- PVC = its just have requirements how big volume and access mode (like ReadWrite) we need and it will search through PVC to find that

## Config:
- We create 2 sections in spec: part
- 1. volumes (same layer as container)
    - define name and how to mount
    - easy test for learning emptyDir: {}
- 2. containers -> name -> volumeMounts

## ConfigMap

- Create CM for file:
``` 
ASOVAK-M-J0S8:Downloads asovak$ kubectl create cm --from-file /tmp/ntp.conf ntp-conf
configmap/ntp-conf created
ASOVAK-M-J0S8:Downloads asovak$ kubectl get cm ntp-conf
NAME       DATA      AGE
ntp-conf   1         21s
ASOVAK-M-J0S8:Downloads asovak$ kubectl get cm ntp-conf -o yaml
apiVersion: v1
data:
  ntp.conf: |
    server ntp.esl.cisco.com
    server time.apple.com
    server time.asia.apple.com
    server time.euro.apple.com
kind: ConfigMap
metadata:
  creationTimestamp: 2020-07-18T16:05:30Z
  name: ntp-conf
  namespace: default
  resourceVersion: "2539996"
  selfLink: /api/v1/namespaces/default/configmaps/ntp-conf
  uid: f1aa3573-7789-4907-92f3-ec82c8a16af5
```

- Create POD with NTP reference:
```
spec:
  containers:
  - name: my-pod
     image: busybox
     volumeMounts:
     - name: ntpConf
       mountPath: /etc/
  volumes:
  - name: ntpConf
     configMap:
       name: my-pod
       items:
       - key: ntp.conf
          path: ntp.conf
```

- Create CM for key/val:
``` ASOVAK-M-J0S8:Downloads asovak$ kubectl create cm myconf --from-literal=game=chess
configmap/myconf created
ASOVAK-M-J0S8:Downloads asovak$ kubectl get cm myconf -o yaml
apiVersion: v1
data:
  game: chess
kind: ConfigMap
metadata:
  creationTimestamp: 2020-07-18T16:19:26Z
  name: myconf
  namespace: default
  resourceVersion: "2542401"
  selfLink: /api/v1/namespaces/default/configmaps/myconf
  uid: eb3e9daa-2c91-476a-bd91-1303f4b3d84a
```

- Define in POD:
```
apiVersion: v1
kind: Pod
metadata:
  nname: gamePod
spec:
  containers:
  - name: freeGames
     image: gameGenerator
     env:
     - name: GAME
       valueFroM:
         configMapRef:
           name: myconf
           key: game
  restartPolicy: never
```

## Sercrets
- Store sensitive data, like passwords, auth keys, ssh keys
- reduce the risk of exposure
- 3 tyopes:
    - docker-registry - used to connect to registry
    - TLS
    - generic - create from local file
- Base secrets must have base64 encided values
- Withing the POD the password is clear text, just outside is base64

```
ASOVAK-M-J0S8:Downloads asovak$ kubectl create secret generic testingsecret --from-literal=user=admin --from-literal=password=secret123
secret/testingsecret created
ASOVAK-M-J0S8:Downloads asovak$ 
ASOVAK-M-J0S8:Downloads asovak$ kubectl get secret testingsecret -o yaml
apiVersion: v1
data:
  password: c2VjcmV0MTIz
  user: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: 2020-07-18T16:29:22Z
  name: testingsecret
  namespace: default
  resourceVersion: "2544117"
  selfLink: /api/v1/namespaces/default/secrets/testingsecret
  uid: cf273475-d139-4f5e-92ea-57b585384920
type: Opaque
ASOVAK-M-J0S8:Downloads asovak$ 
```

Create POD:
```
apiVersion: v1
kind: Pod
metadata:
  name: secretPod
  namespace: default
spec:
  containers:
  - name: secretBox
     image: busybox
     command:
       - sleep 
       - "6000"
     volumeMounts:
     - mountPath: /etc/passwordReader
        name: secretMount
  volumes:
  - name: secretMount
     secret:
       secretName: testingsecret
```

## Service Networking
- Allows to communicate with PODs which have the same tag/label
- We create Endpoint for every POD on service side to ensure communication with all replicas
- Service function as a load balancer and its a front door for outer world (or behind Ingress)
- Types:
    - ClusterIP - service reachable only within cluster
    - NodePort - service is exposed on each node's IP add as a specific port. (node ip + node port)
    - LoadBalancer - cloud offers load balancer that routes traffic to nodePort or ClusterIP
    - ExternalName - service is mapped as implemented as DNS CNAME

Command to create service that expose Pod or Deployment
```
# kubectl expose pod testpod —type=NodePort —port=112233
```

## Custom Resource Definition

```
apiVersion: apiextension.k8s.io/v1
kind: CustomResourceDefintion
metadata:
  name: os.backups.com
spec:
  group: backups.com
  version: v1
  scope: Namespaced
  names:
    plural: backups
    singular:  backup
    shortNames: [ bk ]
    kind: BackUp
```

After We can use it:
```
apiVersion: "stable.linux.com/v1"
kind: BackUp
metadata:
  name: mylinux
spec:
  timeSpec: "* *  * * * */5"
  image: linux
  replicas: 5
```

## Taints and Tolerations:
- Taints applied to Node for some specific purpose
- Tolerations ignore taints, otherwise cannot spin up POD on tainted node
- POD without toleration cannot run on Node with taint
- Taints are key=value:effect
- 3 types of effects:
    - NoSchedule - does not schedule new pods
    - PreferNoSchedule - if possible not create new pods
    - NoExecute - migrate all pods away

Create key value with no schedule for node
```
kubectl taint nodes node1 test-key=test-value:NoSchedule
```

Here if you spin up POD with test-key as test-value, you can run on this node (Toleration)
```
spec:
  tolerations:
    - key: test-key
       operator: "Exists"
       effect: "NoSchedule"
```

## Cluster Management
### Add Nodes
- Adding worker:
    - setup docker
    - install kubernetes
    - kubeadm token create
    - export TOKEN=<token>
    - opensssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2> /dev/null | openssl dgst -sha256 | sed 's/ˆ.* //'
    - kubeadm join --discovery-token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:1234..cdef 1.2.3.4:6443

### Reboot of the node
- kubectl cordon Node (stops spawning new PODs)
- kubectl drain Node (migrate old PODs)
- reboot
- After it joins back automatically

## Security (new user)
1. Create custom ns
	```kubectl create ns test```

2. See the client config
	```kubectl config get contexts```

3. Create custom user (if needed)
	```
	# sudo useradd -G wheel john
        # sudo passwd john
        # openssl genrsa -out john.key 2048
        # openssl req -new -key john.key -out john.csr -subj"/CN=john/O=dev"
	# sude openssl x509 -req -in john.csr -CA /etc/kubernetes/pki/ca.crt —CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out john.crt -days 180

4. Connfig context
	# kubectl config set-credentials john —client-certificate=./john.crt —client-key=./john.key
	# kubectl config set-context john-context —cluster=kubernetes —namespace staff —user=john
	# kubectl —context=john-context get pods
	# kubectl config get-contexts

5. RBAC
	# kubectl create -f staff-role.yaml

```
staff-role.yaml

kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: staff
  name: staff
rules:
- apiGroups: ["", "extension", "apps"]
       resources: ["deployments", "replicasets", "pods"]
       verbs: ["list", "get", "watch", "create", "update", "patch", "delete"]
```

6. Bind user to the new role
	# kubectl create -f bindit.yaml

```
bindit.yaml

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata: 
  name: staff-role-bind
  namespace: staff
subjects:
- kind: User
       name: anna
       apiGroup: ""
roleRef:
  kind: Role
  name: staff
  apiGroup: ""
``` 

7. Testing
	# kubectl —context=john-context gett pods
	# kubectl create deployment nginx —image=nginx
	# kubectl -n staff describe role staff

## Node Manipulation
### Add node post install
1. Create token:
	# kubectl token create
2. Use openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2> /dev/null | openssl dgst -sha256 -hex | sed 's/^.*//'
3. on new node run: kubeadm join —token <token create output> <ip of controller><port=6443> —discovery-token-ca-cert-hash sha256:<openssl output
4. Add new node to /etc/hosts

### Rebooting Nodes in a proper way
1. #kubectl cordon <NODE>
2. #kubectl drain <Nodename>
3. reboot

### Removing a node
1. #kubectl drain <Node> —delete-local-data —force —ignore-deamonsets
2. #kubectl delete node <Node>

### Rebuild the cluster
1.  #kubeadm reset 
2. rm -rf ~/.kube/config

### Plan Maintanance Windows
1. Stop any new POD deployments on node by: kubectl cordon <node>
2. Remove all existings pods from node by: kubectl drain <node>

### Static Pods 
1. Pods which are associated with Node and always running there
2. Create standard POD yaml file
3. Put it to /etc/kubernetes/manifests (or whatever is configured in /var/lib/kuberntetes/config.yaml)

## Manage ETCD database
Create a backup by:

sudo ETCDCTL_API=3 etcdctl snapshot save snapshot.db —endpoints https://127.0.0.1:2379 —cacert=/etc/kubernetes/pki/etcd/ca.crt —key=/etc/kubernetes/pki/etcd/server.key —cert=/etc/kubernetes/pki/etcd/server.cert

Verify:
sudo ETCDCTL_API=3 etcdctl —write-out=table snapshots status snapshot.db

## Monitor
1. # kubectl top
2. Install metric server
3. Check the logs of pods by: kubectl logs <pod>

## Yaml Examples
`Pod`
```
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  namespace: default
  labels:
    app: busybox
spec:
  containers:
  - name: busy
     image: busybox
     command:
       - hostname
  - name: nginx
     image: nginx
```

`Volume without PVC`
```
spec:
  containers:
    - name: test
       image: centos:7
       command: 
         - hostname
       volumeMounts:
       - mountPath: /test-dir
          name: testMount
  volumes:
  - name: testMount
     emptyDir: {}
```

### ConfMap + Pod with Volume or Env

```ASOVAK-M-J0S8:kind asovak$ cat confMap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-conf
data:
  username: "admin"
  hostname: "localhost"
  app: |
    connnection_type=secure
    max_connection=3
```

```
apiVersion: v1
kind: Pod
metadata:
  name: confpod
spec:
  containers:
  - name: confpod-demo
    image: nginx
    env:
    - name: DB_HOSTNAME
      valueFrom:
        configMapKeyRef:
          name: database-conf
          key: hostname
    - name: DB_USERNAME
      valueFrom:
        configMapKeyRef:
          name: database-conf
          key: username
    volumeMounts:
    - name: appdbconfig
      mountPath: "/app_config"
      readOnly: true
  volumes:
  - name: appdbconfig
    configMap:
      name: database-conf
      items:
      - key: "app"
        path: "app"
```

`inside view`

```
root@confpod:/# cat /app_config/app 
connnection_type=secure
max_connection=3
root@confpod:/# 
```

```
root@confpod:/# env | grep DB_
DB_HOSTNAME=localhost
DB_USERNAME=admin
```
