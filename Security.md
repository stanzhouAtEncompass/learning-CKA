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
### Cluster roles and role bindings
Namespaces
1. namesapaced: pods, replicasets, jobs, deployments, services, secrets, roles, rolebindings, configmaps, PVC
`k api-resources --namespaced=true`
2. cluster scoped: nodes, PV, clusterroles, clusterrolebindings, certificatesigningrequests, namespaces
`k api-resources --namspaced=false`
#### clusterroles
Cluster admin
- Can view nodes
- Can create nodes
- Can delete nodes
Storage admin
- Can view PVs
- Can create PVs
- Can delete PVs

cluster-admin-role.yaml
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","get","create","delete"]
```
`k create -f cluster-admin-role.yaml`
Create a user link to custer role
cluster-admin-role-binding.yaml
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
- kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```
`k create -f cluster-admin-role-binding.yaml`

``` yaml
#Solution manifest file to create a clusterrole and clusterrolebinding for michelle user:
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: node-admin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list", "create", "delete"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: michelle-binding
subjects:
- kind: User
  name: michelle
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-admin
  apiGroup: rbac.authorization.k8s.io
```
### Service Accounts
two types of accounts in k8s:
1. user account
used by humans
- Admin: accessing the cluster to perform administrative tasks
- Developer: accessing the cluster to deploy applications
2. service account
used by machines
account used by application to interact with k8s cluster

e.g. Prometheus uses a service account to pull the k8s api for performance metrices 
Jenkins deploy applications on the k8s cluster

create a service account
`k create serviceaccount dashboard-sa`
to view the service account: `k get serviceaccount`
When the service account is created, it also creates a token automatically, the service account token is what must be used by the exteneral application while authenticating to the k8s API. The token is stored as a secret object.
`k describe serviceaccount dashboard-sa`

#### When a service account is created
1. craetes the service account object
2. generates a token for the service account
3. creates a secret object and stores that token inside the secret object
4. the secret object is then linked to the service account 
to view the secret object by `k describe secret dashboard-sa-token-kbbdm`
[service account](https://i.imgur.com/F9HjCWq.png)

can using curl: `curl https://ipaddress:6443/api -insecure --header "Authorization: Bearer [token content]"`

#### default service account
A service account named default is automatically created.
Each namespace has its own default service account whenever a part is created.
The default SA and its token are automatically mounted to that part as a volume mount.

`k exec -it my-kubernetes-dashboard ls /var/run/secrets/kubernetes.io/serviceaccount`
The one with the actual token is the file named token.

ps. the default service account is very much restricted. It only has permission to run basic common API queries.

**you cannot edit the service account of an existing part, must delete and recreate the pod**

k8s automatically mounts the default service account if you haven't explicitly specified any. 
not to mount a service account automatically by setting the `automountServiceAccountToken: false`.

using RBAC
#pod-reader-role.yaml
``` yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups:
  - ''
  resources:
  - pods
  verbs:
  - get
  - watch
  - list
```

#dashboard-sa-role-binding.yaml
``` yaml
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: dashboard-sa # Name is case sensitive
  namespace: default
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

#### Lab notes
You shouldn't have to copy and paste the token each time. The Dashboard application is programmed to read token from the secret mount location. However currently, the 'default' service account is mounted. Update the deployment to use the newly created ServiceAccount

Edit the deployment to change ServiceAccount from 'default' to 'dashboard-sa'

**Use the serviceAccountName field inside the pod spec.**
``` yaml
template:
    metadata:
      creationTimestamp: null
      labels:
        name: web-dashboard
    spec:
      serviceAccountName: dashboard-sa
      containers:
      - image: gcr.io/kodekloud/customimage/my-kubernetes-dashboard
        imagePullPolicy: Always
        name: web-dashboard
        ports:
        - containerPort: 8080
          protocol: TCP           
```
### Image security
Private repository
`docker login private-registry.io`
`docker run private-registry.io/apps/internal-app`

How do you pass the credentials to the docker on times, on the worker node?
create a secret object
`k create secret docker-registry regcred --docker-server= --docker-username= --docker-password= --docker-email=`
# nginx-pod.yaml
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: private-registory.io/apps/internal-app
  imagePullSecrets:
  - name: regcred
```
### Security Contexts
configure the security settings at a container level or at a pod level
if configured it at a pod level the settings will carry over to all the containers within the pod.
if configure it at both the pod and the container the settings on the container will override the settings on the pod.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
 containers:
   - name: ubuntu
     image: ubuntu
     command: ["sleep", "3600"]
     securityContext:
       runAsUser: 1000
       capabilities:
          add: ["MAC_ADMIN", "NET_ADMIN"]
```
**Capabilities are only supported at the container level and not at the POD level**

Lab:
What is the user used to execute the sleep process within the ubuntu-sleeper pod?
`k exec ubuntu-sleeper -- whoami`
### Network Policies
When you define ingress and egress remember you're only looking at the direction in which the traffic originated.
one of the pre-requisite for networking:
the pods should be able to communicate with each other without having to configure any additional settings like routes.

k8s is configured by default with an "All Allow" rule that allows traffic from any pod to any other pod or services.
A network policy is another object in the k8s namespace. You link a network policy to one or more pods.

How do you apply or link a network policy to a pod.
Use the same technique that was used before to link ReplicasSets or Servies to Pods. Labels and Selectors.
Label the pod and use the same labels on the pod selector field in the network policy. And then build our rule uder policy types specify whether the rule is allow ingress or egress traffic or both.

``` yaml
labels:
  role: db
  
podSelector:
  matchLabels:
    role: db
# rules
policyTypes:
- Ingress
ingress:
- from:
  - podSelector:
     matchLabels:
       name: api-pod
  ports:
  - protocol: TCP
    port: 3306
```
put it together:

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
   matchLabels:
     role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
       matchLabels:
         name: api-pod
    ports:
    - protocol: TCP
      port: 3306
```
`k create -f policy-definition.yaml`
Note:
Solutions that support network policies:
- Kube-router
- Calico
- Romana
- Weave-network
Solutions that DO NOT support network policies:
- Flannel
### Developing network policies
do we need ingress or egress here or both?
always look at this from the DB pod perspective,
to allow incoming traffic from the API part, so that is incoming=> ingress.
[network policy](https://i.imgur.com/oJJYURQ.png)
The API pod makes database queries and the database pod returns the results.
Do you need a separate rule for the results to go back to the api pod?
No, once you allow incoming traffic, the response or reply to that traffic is allowed back automatically. **we don't need a separate rule for that**
Using the namespace selector defining from what namespace traffic is allowed.
``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
   matchLabels:
     role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
         matchLabels:
           name: api-pod
      namespaceSelector:
         matchLabels:
           name: prod
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - ipBlock:
      cidr: 192.168.5.10/32
    ports:
    - portocol: TCP
      port: 80
```
#### configure a network policy to allow traffic originating from certain ip addresses
IP block: allows you to sepecify a range of IP addresses from which you could allow traffic to hit the database pod.
[ipBlock](https://i.imgur.com/GjeCrtC.png)

Two separate rules:[separate rules](https://i.imgur.com/Mxh2YUL.png) 
- means that traffic matching the first rule is allowed, that is from any pod matching the label api-pod in any namespace and traffic matching.
- the 2nd rule is allowed, which from any pod within the namespace, and that is either from the pod web, along with the backup server as we have the IP block specification as well.

**lab**
Q:Create a network policy to allow traffic from the Internal application only to the payroll-service and db-service.

Use the spec given on the below. You might want to enable ingress traffic to the pod to test your rules in the UI.
[task](https://i.imgur.com/OoVu48x.png)
A: **Note:** We have also allowed Egress traffic to TCP and UDP port. This has been added to ensure that the internal DNS resolution works from the internal pod. Remember: The kube-dns service is exposed on port 53:

``` yaml
root@controlplane:~# k get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   19m
root@controlplane:~# cat do.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
spec:
  podSelector:
   matchLabels:
     name: internal 
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
         matchLabels:
           name: mysql
    ports:
    - protocol: TCP
      port: 3306
  - to:
    - podSelector:
         matchLabels:
           name: payroll
    ports:
    - protocol: TCP
      port: 8080
  - ports:
    - port: 53
      protocol: UDP 
    - port: 53
      protocol: TCP
```
Q: how many network policies do you see in the environment?
A: `k get netpol` # netpo is short for network policy

`k get pods -l name=payroll` # l specific label
