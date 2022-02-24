# Networking
## Networking pre-requisites
### Switching and Routing
- Switching
- Routing
A router helps connect two networks together.
- Default Gateway
If the network was a room, the gateway is a door to the outside world to other network or to the internet.A
``` bash
route
ip route add 192.168.2.0/24 via 192.168.1.1
vim /etc/network/interfaces
cat /proc/sys/net/ipv4/ip_forward # check if IP forwarding is enable on a host
1
```
### DNS
- DNS configurations on linux
order file: /etc/nsswitch.conf

apps.google.com
Org DNS-> Root DNS-> .com DNS-> google dns
- CoreDNS introduction
#### search domain
cat >> /etc/resolve.conf
nameserver 192.168.1.100
search mycompany.com

ping web = ping web.mycomany.com
**nslookup does not consider the entries in the local /etc/host file**
### Network namespaces
create network ns
``` bash
ip netns add red
ip netns add blue
# to list 
ip netns
```
exec in entork ns
``` bash
ip link
ip netns exec red ip link
ip -n red link
```
![how to](https://i.imgur.com/UTll0FW.png)
the arp table: `ip netns exec red arpi`

more than two network need a switch:
linux bridge, open vswitch
![linux bridge](https://i.imgur.com/xZsBB3G.png)
del the link:
`ip -n red link del veth-red`
### Docker Networking 
### Linux interfaces for virtual networking
- Bridge Network
1. none network
the docker container is not attached to any network. the container cannot reach the outside world and no one from the outside world can reach the container.
2. Host network
There is no network isolation between the host and the container.
3. Bridge
An internal private network is created which the docker host and containers attach to.
e.g. The network has an address 172.17.0.0 by default and each device connecting to this network get their own internal private network address on this network.
- CNI(Container newtwork interface)
* Container runtime must create network namespace
* Identify network the container must attach to
* Container runtime to invoke network plugin (bridge) when container is ADDed.
* Container runtime to invoke network plugin (bridge) when container is DELeted.
* JSON format of the network configuration

## Cluster Networking
### Ports
![master and work node ports](https://i.imgur.com/2tUajgm.png)
Master-01 to master-02 
etcd 2380 port need to open

## Pod networking
- Every POD should have an IP address
- Every POD should be able to communicate with every other POD in the same node.
- Every POD should be able to communicate with every other POD on other nodes without NAT.
run the script automatically:
# net-script.sh
``` bash
ADD)
 # create veth pair
 # Attach veth pair
 # Assign IP address
 # Bring Up interface
 ip -n <namespace> link set ...
DEL)
 # Delete veth pair
 ip link del ...
```
The **kubelet** looks at the CNI configuration passed as a command line argument when it was run and idetifies our scripts name:
`--cni-conf-dir=/etc/cni/net.d`
## CNI in kubernetes
### Configuring CNI
The CNI plugin is configured in the kubelet service on each node in the cluster.
![kubelet.service](https://i.imgur.com/88xHRbF.png**
**View kubelet options**
ps -aux|grep kubelet
ls /opt/cni/bin # has all the supported CNI plugins as executables, such as the bridges, dhcp,flannel etc.
ls /etc/cni/net.d # a set of configuration files
It will choose the one in alphabetical order.
## CNI weave
### Deploy weave
`k apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version|base64|tr -d '\n')"`
### Weave Peers
`k get pods -n kube-system`
`k logs weave-net-5gcmb weave -n kube-system`
## Service Networking
![kube-proxy](https://i.imgur.com/dG48Wv4.png "kube-proxy")
![iptables](https://i.imgur.com/UkbgN7o.png "iptables")
## DNS in kubernetes
![svc dns](https://i.imgur.com/AHAR8Qh.png)
![pod dns](https://i.imgur.com/uYdcD4O.png)
## CoreDNS in k8s
The coredns server is deployed as a POD in the kube-system namspace in the k8s cluster.
Two pods for redundancy,as part of a replicaset.
`cat /etc/coredns/Corefile`
Core file is passed into the pod has a confiMap object. `k get configmap -n kube-system`

When we deploy coredns solution, it also creates a service to make it available to other components within a cluster.
The service is named as kube-dns by default.
`k get service -n kube-system`

## Ingress
Ingress helps your users access your application using a single externally accessible URL, that you can configure to route to different services with your cluster 
based on the URL path, at the same time implement SSL security as well.
1. Deploy tools like: Nginx Haproxy traefik
Ingress controller: GCP https(s), Load balaner (GCE), nginx, contour, haproxy, traefik, Istio

Nginx:
``` yaml
apiVersion: extendsions/v1beat1
kind: Deployment
metada:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    sepec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
      arges:
        - /ginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
```
So we create a service of type NodePort with the gninx-ingress label selector to link the service to the deployment.
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
  selector:
    name: nginx-ingress
```

The ingress controllers have additional intelligence build-in to monitor the k8s cluster for ingress resources and configure the underlying nginx server 
when sth is changed but for the ingress controller to do this, it requres a service account with a right set of permissions for that we create a service account with the roles and roles bindings.
``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```
![ingress controller](https://i.imgur.com/FeG3kRG.png)
with a deployment of the nginx-ingress image, a service to expose it, a ConfigMap to feed nginx configuration data, and a service account with the right permissions to access all of these objects.

ConfigMap: nginx-configuration
``` yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
```

Not deployed by default

2. Configure
the set of rules you configure are called as ingress resources
can configure rules to simply forward all incoming traffic to a single application or route traffic to different applications.
Base on the URL or domain name itself.
``` yaml
# Ingress-wear.yaml
apiVersion: extensions/v1beata1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  backend:
    serviceName: wear-service
    servicePort: 80
```
k create -f Ingress-wear.yaml

k get ingress
![Ingress resource-rules](https://i.imgur.com/A5xc7IZ.png "rules")
1. Rule
  www.my-online-store.com
2. Paths
/wear  /watch

``` yaml
apiVersion: extensions/v1beata1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - http:
      paths:
      - path: /wear
        backend:
          serviceName: wear-service
          servicePort: 80
      - path: /watch
        backend:
          serviceName: watch-service
          servicePort: 80
```
k describe ingress ingress-wear-watch

if a user tries to access a URL that does not match any of these rules, then the user is directed to the service specified as the default backend.
![default backend](https://i.imgur.com/M3TBLUR.png)
in this case it happens to be a service named default-http-backend. So you must remember to deply such a service back in your application.
3. Domain name
weare.my-online-store.com
watch.my-online-store.com
``` yaml
apiVersion: extensions/v1beata1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - host: wear.my-online-store.com
    http:
      paths:
      - backend:
          serviceName: wear-service
          servicePort: 80
  - host: watch.my-online-store.com
    http:
      paths:
      - backend:
          serviceName: watch-service
          servicePort: 80
```

**if you don't specify the host field it will simply consider it as a star or accept all the incoming traffic through that particular rule without matching the hostname in this case**

## current version in **Ingress**
``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port: 
              number: 80
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port: 
              number: 80
```

Now, in k8s version 1.20+ we can create an Ingress resource from the imperative way like this:-
Format - kubectl create ingress <ingress-name> --rule="host/path=service:port"
Example - kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"
`k get ingress --all-namespaces`
``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-pay
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /pay
        pathType: Prefix
        backend:
          service:
            name: pay-service
            port:
              number: 8282
```
## Ingress-Annotations and rewrite-target
For example: replace(path, rewrite-target)
In our case: replace("/path","/")
``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /pay
        backend:
          serviceName: pay-service
          servicePort: 8282
```
In another example given here, this could also be:

replace("/something(/|$)(.*)", "/$2")

``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: rewrite
  namespace: default
spec:
  rules:
  - host: rewrite.bar.com
    http:
      paths:
      - backend:
          serviceName: http-svc
          servicePort: 80
        path: /something(/|$)(.*)
```

### Lab
What is the networking solution used by this cluster?
Hint: check the config file located at `/etc/cni/net.d/`

What is the default gateway configured on the PODs scheduled on node01?

Try scheduling a pod on node01 and check ip route output
