# Security
## Secure Kubernetes
kube-apiserver: the first line of defense, controlling access to the API server itself.
### Authentication
who can access?
- Files - Username and passwords
- Files - Username and tokens
- Certificates
- External authentication providers - LDAP
- Service accounts
what can they do?
- RBAC authorization
- ABAC authorization
- Node authorization
- Webhook node
### TLS certificates
All communication with the cluster, between the various components such as the ETCD cluster, kube controller manager,
scheduler, api server, as well as those running on the worker nodes such as the kubelet and kubeproxy is secured using
TLS encryption.
### Network policies
can restrict acess pod access all other pods within the cluster.
### Accounts
- User: managed by API server
  - Admins
  `kubectl`
  - Developers
  `curl https://kube-server-ip:6443/`
  The kube api server authenticates the request before processing it.
### How does the kube api server authenticate
Auth mechanisms:
- static password file
- staic token file
- certificates
- identity services
#### Basic
``` shell
user-details.csv
password123,user1,u0001
password123,user2,u0002

kube-apiserver
kube-apiservicer.services
ExecStart=...
...
--basic-auth-file=user-details.csv

#/etc/kubernetes/mainifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-apiserver
  namspace: kube-system
spec:
  containers:
  - commands:
    - kube-api-server
    - --authentication-mode=Node,RBAC
    - --advertise-address=172.17.0.107
    - --allow-previleged=true
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --basic-auth-file=user-details.csv
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
```
#### authenticate user
`curl -v -k https://master-node-ip:6443/api/v1/pods -u "user1:password123"`
- Service Accounts
  - Bots
  
  CA: certificate authority (CA)
  PKI: public key infrastructre
  Certificate (public key): *.crt *.pem
  server.crt server.pem client.crt client.pem
  
  Private key: *.key *-key.pem
  server.key server-key.pem client.key client-key.pem
  
#### TLS in Kubernetes
CA: ca.crt ca.key

Server certificates for servers
Kube-API server   apiserver.crt apiserver.key
ETCD SERVER etcdserver.crt etcdserver.key
KUBELET SERVER kubelet.crt kubelet.key

Client certificates for clients
admin.crt admin.key  admin
scheduler.crt scheduler.key KUBE-SCHEDULER
controller-manager.crt controller-manager.key  KUBE-CONTROLLER-MANAGER
kube-proxy.crt kube-proxy.key  KUBE-PROXY
apiserver.crt apiserver.key  KUBE-API SERVER
[diagram](https://i.imgur.com/d9bTo3j.png "crt")  

#### certificate creation
Certificate authority (CA)
1. Generate keys ca.key
`openssl genrsa -out ca.key 2048`
2. Certificate signing request ca.csr
`openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr`

kube api server
`openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr`

openssl.cnf
``` shell
[req]
req_extensions = v3_request
distinguished_name = req_distinguished_name
[ v3_req ]
baseConstraints = CA:FALSE
keyUsage = nonRepudiation,
subjectAltName = @atl_names
[atl_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```

#### View certificate details
`cat /etc/kubernetes/mainifests/kube-apiserver.yaml`
decode the cert for details:
`openssl x509 -in /etc/kubernetes/pki/apiserver.cert -txt -noout`
check for Subject.

[kubeadm](https://i.imgur.com/G4iHeDv.png "kubeadm")

##### Inspect service logs
`journalctl -u etcd.service -l`
##### View logs
`kubectl logs etcd-master`
`docker ps -a`
`docker log 87fc`

### kubeconfig
`kubectl get pods --kubeconfig config`
Lookin into the path $HOME/.kube/config
``` shell
#config file content
--server my-kube-playground:64443
--client-key admin.key
--client-certificate admin.crt
--certificate-authority ca.crt
```
##### kubeconfig file
Clusters:
- Development
- Production
- Google
- MyKubePlayground
Contexts:
- Admin@Production
- Dev@Google
- MyKubeAdmin@MyKubePlayground
Users:
- Admin
- Dev User
- Prod User
- MyKubeAdmin

``` fish
apiVersion: v1
kind: Config

current-context: dev-user@google

clusters:
- name: my-kube-playground
  cluster:
    certficate-authority: ca.crt
    server: https://my-kube-playground:6443
- name: development
- name: production
- name: google

contexts:
- name: my-kube-admin@my-kube-playground
  context:
    cluster: my-kube-playground
    user: my-kube-admin
- name: dev-user@google
- name: prod-user@production


users:
- name: my-kube-admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
- name: admin
- name: dev-user
- name: prod-user
```
view context by: 
``` shell
kubectl config view
k config view --kubeconfig=my-custom-config
k config use-context prod-user@production
k config --kubeconfig=/root/my-kube-config use-context foobar
# check by
k config --kubeconfig=/root/my-kube-config current-context
k config -h #help
```
#### Namespaces
automatically switch
#### Certificates in kubeconfig
two ways:
1. use the cert file

``` shell
apiVersion: v1
kind: Config

clusters:
- name: production
  cluster:
    certficate-authority: /etc/kubernetes/pki/ca.crt
```
ca.crt file:
-----BEGIN CET....
-----END CET...
2. use content:

``` shell
apiVersion: v1
kind: Config

clusters:
- name: production
  cluster:
    certficate-authority-data:
     # convert the ca.crt file by: cat ca.crt|base64
     # if want to decode by: echo "LS0...bnj"| base64 --decode
```

#### API Groups
/metrics /healthz /version /api(core) /apis(named) /logs

##### core
/api
/v1
namespaces pods pr events endpoints nodes bindings PV PVC configmaps secrets services
##### named
/apis 
/apps -> /v1 -> Resources: /deployments /replicasets /statefulsets
Vebs: list, get, create, delete, update, watch
`curl http://localhost:6443 -k`
`curl http://localhost:6443/apis -k|grep "name"`
/extensions
/networking.k8s/io -> /v1 -> /networkpolicies
/stoarge.k8s.io
/authentication.k8s.io
/certificates.k8s.io-> /v1 -> /certificatesigningrequests

peope->kubectl proxy->kube apiserver
`kubectl proxy`
`curl http://localhost:8001 -k`
kube proxy not = kubectl proxy
### Authorization
#### Mechanisms
1. Node
kubelet access the kube apiVersion
- Read
  - Services
  - Endpoints
  - Nodes
  - Pods
- Write
  - Node status
  - Pod status
  - events
  Node authorizer
2. ABAC
dev-user
- can view PODs
- can create PODs
- Can delte PODs
`{"kind":"Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup":"**"}}`
3. RBAC
[RBAC](https://i.imgur.com/s7ZTcP5.png)
developer-role.yaml
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list","get","create","update","delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```
create a role by `kubectl create -f developer-role.yaml`
devuser-developer-binding.yaml
[role-binding](https://i.imgur.com/Vhb7lld.png)
View RBAC
`k get roles`
`k get rolebindings`
`k describe role developer`
`k describe rolebinding devuser-developer-binding`
Edit RBAC
`k edit role developer -n blue`
Check Access
`k auth can-i create deployments`
`k auth can-i delete nodes`
`k auth can-i create deployments --as dev-user`
`k auth can-i create pods --as dev-user`
`k auth can-i create pods --as dev-user --namespace test`
Resources names
[resources names](https://i.imgur.com/NoO7wj8.png)


4. Webhook
tool: open policy agent
5. AlwaysAllow
6. AlwaysDeny
