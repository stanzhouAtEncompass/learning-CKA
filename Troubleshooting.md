# Troubleshooting
## Application Failure
### Check accessibility
Two tier application: a web and a database server.
- database pod: hosting a database and serving the web servers through a database service
- web pod: serves users through the web service.
Write down or draw a map or chart of how your application is configured before you start.

Check every object and link in this map until you find the root cause of the issue.

1. If it's a web application, check if the web server is accessible on the ip of the node port using curl.
`curl http://web-service-ip:node-port`
2. Check the service.
Has it discovered endpoints for the web pod?
`k describe service web-service`
Endpoints: 10.32.0.6:8080
If not, you might want to check the service to the pod discovery. Compare the selectors configured on the
service to the ones on the pod. Make sure they match.
`Selector: name=webapp-mysql`
`name:webapp-mysql`
3. Check the pod itself and make sure it is in a running state.
`k get pod`
Check the events related to the pod using `k describe pod web`
Check the logs of the application using `k logs web`
If the pod is restarting due to a failure, then the logs in the current version of the pod that's running
the current version of the container may not reflect why it failed the last time. So you either have to watch
those logs using the -f option and wait for the application to fail again or use the previous options to view the logs of a
previous pod.
4. Check the status of the db-service as before.
5. Check the DB pod itself.

### Lab
`k -n alpha get svc mysql -o yaml> mysql-service.yaml`

mysql-service don't have endpoint
``` shell
k get svc mysql-service -n -o yaml
k -n gamma delete svc mysql-service
k -n gamma expose pod mysql --name=mysql-service
k -n gamma get ep (endpoint)
k -n delta get deployments. webapp-mysql -o yaml > web.yaml
k delete deployment. webapp-mysql -n delta
```
## Control Plane Failure
### Check Controlplane Pod
`k get pods -n kube-system`
### Check Controlplane Services
Check the status of the services
`service kube-apiserver status`
`service kube-controller-manager status`
and schedule on the master node
`service kube-scheduler status`
and kubelet
`service kubelet status`
on the worker nodes: `service kube-proxy status`
### Check service logs
`k logs kube-apise`
`k logs kube-apiserver-master -n kube-system`
`sudo journalctl -u kube-apiserver`

more tip ref to k8s troubleshoot clusters

### Lab
1. The cluster is broken again. We tried deploying an application but it's not working. Troubleshoot and fix the issue.
start looking at the deployments.

First step check the deployment in the cluster:
```
k get all
k describe pod app-foobar
Stats: Pending
don't have any other issue on this pod
k get pods -n kube-system
k -n kube-system describe pod kube-scheduler-master
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
find KUBELET_CONFIG_ARGS
grep -i staticPodPath /var/lib/kubelet/config.yaml
watch k get all
```
2. Scale the deployment app to 2 pods
k scale deployment app --replicas=2

### Worker node failure
Check Node status
```
k get nodes
k describe node worker-1
```
Check Node
```
top
df -h
```
Check kubelet status
```
ps -ef|grep kubelet
systemctl status kubelet.service -l
journalctl -u kubelet
kubelet config file: /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
#check the default kubelet port
kubectl cluster-info
#check kubelet certificates
openssl x509 -in /var/lib/kubelet/worker-1.crt -text
"", ReportingInstance:""}': 'Post "https://controlplane:6553/api/v1/namespaces/default/events": dial tcp 10.22.183.5:6553: conn
ect: connection refused'(may retry after sleeping)
/etc/kubernetes/kubelet.conf
```
## Network troubleshooting
Network Plugin in kubernetes

--------------------

Kubernetes uses CNI plugins to setup network. The kubelet is responsible for executing plugins as we mention the following parameters in kubelet configuration.

- cni-bin-dir:   Kubelet probes this directory for plugins on startup

- network-plugin: The network plugin to use from cni-bin-dir. It must match the name reported by a plugin probed from the plugin directory.


There are several plugins available and these are some.


1. Weave Net:


  These is the only plugin mentioned in the kubernetes documentation. To install,

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"


You can find this in following documentation :

                  https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/


2. Flannel :


   To install,

  kubectl apply -f               https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml

   

Note: As of now flannel does not support kubernetes network policies.


3. Calico :

   

   To install,

   curl https://docs.projectcalico.org/manifests/calico.yaml -O

  Apply the manifest using the following command.

      kubectl apply -f calico.yaml

   Calico is said to have most advanced cni network plugin.


In CKA and CKAD exam, you won't be asked to install the cni plugin. But if asked you will be provided with the exact url to install it. If not, you can install weave net from the documentation 

      https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/


Note: If there are multiple CNI configuration files in the directory, the kubelet uses the configuration file that comes first by name in lexicographic order.



DNS in Kubernetes
-----------------

Kubernetes uses CoreDNS. CoreDNS is a flexible, extensible DNS server that can serve as the Kubernetes cluster DNS.


Memory and Pods

In large scale Kubernetes clusters, CoreDNS's memory usage is predominantly affected by the number of Pods and Services in the cluster. Other factors include the size of the filled DNS answer cache, and the rate of queries received (QPS) per CoreDNS instance.


Kubernetes resources for coreDNS are:   

    a service account named coredns,

    cluster-roles named coredns and kube-dns

    clusterrolebindings named coredns and kube-dns, 

    a deployment named coredns,

    a configmap named coredns and a

    service named kube-dns.


While analyzing the coreDNS deployment you can see that the the Corefile plugin consists of important configuration which is defined as a configmap.


Port 53 is used for for DNS resolution.


        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }


This is the backend to k8s for cluster.local and reverse domains.


proxy . /etc/resolv.conf


Forward out of cluster domains directly to right authoritative DNS server.



Troubleshooting issues related to coreDNS

1. If you find CoreDNS pods in pending state first check network plugin is installed.

2. coredns pods have CrashLoopBackOff or Error state

If you have nodes that are running SELinux with an older version of Docker you might experience a scenario where the coredns pods are not starting. To solve that you can try one of the following options:

a)Upgrade to a newer version of Docker.

b)Disable SELinux.

c)Modify the coredns deployment to set allowPrivilegeEscalation to true:


    kubectl -n kube-system get deployment coredns -o yaml | \
      sed 's/allowPrivilegeEscalation: false/allowPrivilegeEscalation: true/g' | \
      kubectl apply -f -

d)Another cause for CoreDNS to have CrashLoopBackOff is when a CoreDNS Pod deployed in Kubernetes detects a loop.


  There are many ways to work around this issue, some are listed here:


    Add the following to your kubelet config yaml: resolvConf: <path-to-your-real-resolv-conf-file> This flag tells kubelet to pass an alternate resolv.conf to Pods. For systems using systemd-resolved, /run/systemd/resolve/resolv.conf is typically the location of the "real" resolv.conf, although this can be different depending on your distribution.

    Disable the local DNS cache on host nodes, and restore /etc/resolv.conf to the original.

    A quick fix is to edit your Corefile, replacing forward . /etc/resolv.conf with the IP address of your upstream DNS, for example forward . 8.8.8.8. But this only fixes the issue for CoreDNS, kubelet will continue to forward the invalid resolv.conf to all default dnsPolicy Pods, leaving them unable to resolve DNS.


3. If CoreDNS pods and the kube-dns service is working fine, check the kube-dns service has valid endpoints.

              kubectl -n kube-system get ep kube-dns

If there are no endpoints for the service, inspect the service and make sure it uses the correct selectors and ports.



Kube-Proxy
---------

kube-proxy is a network proxy that runs on each node in the cluster. kube-proxy maintains network rules on nodes. These network rules allow network communication to the Pods from network sessions inside or outside of the cluster.


In a cluster configured with kubeadm, you can find kube-proxy as a daemonset.


kubeproxy is responsible for watching services and endpoint associated with each service. When the client is going to connect to the service using the virtual IP the kubeproxy is responsible for sending traffic to actual pods.


If you run a kubectl describe ds kube-proxy -n kube-system you can see that the kube-proxy binary runs with following command inside the kube-proxy container.


        Command:
          /usr/local/bin/kube-proxy
          --config=/var/lib/kube-proxy/config.conf
          --hostname-override=$(NODE_NAME)

 

    So it fetches the configuration from a configuration file ie, /var/lib/kube-proxy/config.conf and we can override the hostname with the node name of at which the pod is running.

 

  In the config file we define the clusterCIDR, kubeproxy mode, ipvs, iptables, bindaddress, kube-config etc.

 
Troubleshooting issues related to kube-proxy

1. Check kube-proxy pod in the kube-system namespace is running.

2. Check kube-proxy logs.

3. Check configmap is correctly defined and the config file for running kube-proxy binary is correct.

4. kube-config is defined in the config map.

5. check kube-proxy is running inside the container

    # netstat -plan | grep kube-proxy
    tcp        0      0 0.0.0.0:30081           0.0.0.0:*               LISTEN      1/kube-proxy
    tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      1/kube-proxy
    tcp        0      0 172.17.0.12:33706       172.17.0.12:6443        ESTABLISHED 1/kube-proxy
    tcp6       0      0 :::10256                :::*                    LISTEN      1/kube-proxy



References:

Debug Service issues:

                     https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/

DNS Troubleshooting:

                     https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/

