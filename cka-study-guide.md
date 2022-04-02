# Workloads
## API resource types that run an application
- Deployments
- ReplicaSet
- StatefulSet
- DaemonSet
- Job
- CronJob
- Pod
The exam will only include the **Deployment, ReplicaSet, and Pod**. need to understand replication and rollout
features managed by a Deployment, and understand the API primitives for injecting configuration data into a Pod.
## Managing workloads with deployment
![Relationship between a deployment and replicaset](https://i.imgur.com/r6ZKMuI.png "Fig 1")

For label selection to work properly, the assignment of `spec.selector.matchLabels` and `spec.template.metadata` needs to match.
![Maps the Deployment to the Pod teplate used for replicas](https://i.imgur.com/Z4edShB.png "Deployment label selection")

## Autoscaling a deployment
![high-level architecutre diagram involving a HPA](https://i.imgur.com/qCO21n5.png "Autoscaling a deployment")
## Defining and consuming configuration data
If those environment variables become a common commodity across multiple Pod manifests within the same namespace, there’s no way around copy-pasting the definition. For that particular use case, Kubernetes introduced the concept of configuration data represented by dedicated API resources.

Those API resources are called ConfigMap and Secret. Both define a set of key-values pairs and can be injected into a container as environment variables or mounted as a Volume. 

![Configuration data in K8s](https://i.imgur.com/7SUFvYt.png "options")
# Scheduling and Tooling
## Understanding how resource limits affect pod scheduling
![the scheduling process based on the resource requests](https://i.imgur.com/6PuXsPL.png "pod scheduling based on resrouce requests")
## Common templating tools
### yq
- Reading values
use the command `eval` or the short from `e`
the path expression needs to start with a mandatory dot characher`.` to denote the root node of the YAML structure.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: spring-boot-app
spec:
  containers:
  - image: bmuschko/spring-boot-app:1.5.3
    name: spring-boot-app
    env:
    - name: SPRING_PROFILES_ACTIVE
      value: prod
    - name: VERSION
      value: '1.5.3'
```

``` shell
$ yq e '.metadata.name' pod.yaml
spring-boot-app
$ yq e '.spec.containers[0].env[1].value' pod.yaml
1.5.3
```
- Modifying values
`-i` flag `yq e -i '.spec.containers[0].env[1].value = "1.6.0"' pod.yaml`
### Helm
Helm is a templating engine for a set of k8s manifests.

# Service and networking
## Creating services
Creates both objects in one swoop while creating the proper label selection.
`$k run echoserver --image=k8s.gcr.io/echoserver:1.10 --restart=Never --port=8080 --expose`
service/echoserver created
pod/echoserver created
## Rendering serive details
``` shell
kubectl describe service echoserver
kubectl get endpoints echoserver
kubectl describe endpoint echoserver
```
## ClusterIP
![Service port mapping](https://i.imgur.com/fab5THP.png "a Service that accepts incoming traffic on port 80****
**ClusterIP** is the default type of Service. It exposes the service on a cluster-internal ip address.
The service can only be accessed from a Pod running inside of the cluster.
![ClusterIP](https://i.imgur.com/tvPUrRo.png "accessibility of a Service with the type ClusterIP")
## NodePort
can be resolved from outside of the k8s cluster, a port number in the range of 30000 and 32767.
![NodePort](https://i.imgur.com/Nf4v9U9.png)
## LoadBalancer
![LoadBalancer](https://i.imgur.com/EEYqiYw.png)
### Understand the difference between a Service and an Ingress
The Ingress is meant for routing cluster-external HTTP(S) traffic to one or many Services based on an optional hostname and
mandatory path. A Service routes traffic to a set of Pods.

# Storage
## Volume Types
- emptyDir: Only persisted for the lifespan of a Pod.
- hostPath: File or directory from the host node's filesystem.
- configMap, secret: Provides a way to inject configuration data.
- nfs: An existing NFS shared. Preserves data after Pod restart.
- persistentVolumeClaim: Claims a Persistent Volume.
![Claiming a PersistentVolume from a Pod](https://i.imgur.com/bBTCVyB.png "relationship between the Pod, the PersistentVolumeClaim, and the PersistentVolume")
# Troubleshooting
## Troubleshooting Pods
![Common pod error statuses](https://i.imgur.com/RYpDK2e.png "error")
`k get events`: lists the events across all Pods for a given namespace.
`k logs incorrect-cmd-pod`: two helpful options:
- `-f` streams the logs, you'll see new log entries as they're being produced in real time
- `--previous` gets the logs from the previous instantiation of a container, which is helpful if the container has been restarted.
### Opening an interactive shell
``` shell
$ kubectl exec failing-pod -it -- /bin/sh
# mkdir -p ~/tmp
# cd ~/tmp
# ls -l
total 4
-rw-r--r-- 1 root root 112 May  9 23:52 curr-date.txt
```
### Troubleshooting Services
In case you can’t reach the Pods that should map to the Service, start by ensuring that the label selector matches with the assigned labels of the Pods. You can query the information by describing the Service and then render the labels of the available Pods with the option --show-labels. The following example does not have matching labels and therefore wouldn’t apply to any of the Pods running in the namespace:
``` shell
$ kubectl describe service myservice
...
Selector:          app=myapp
...
$ kubectl get pods --show-labels
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
myapp-68bf896d89-qfhlv   1/1     Running   0          7m39s   app=hello
myapp-68bf896d89-tzt55   1/1     Running   0          7m37s   app=world
```
you can also query the endpoints of the Service instance. Say you expected three Pods to be selected by a matching label but only two have been exposed by the Service
``` shell
$ kubectl get endpoints myservice
NAME        ENDPOINTS                     AGE
myservice   172.17.0.5:80,172.17.0.6:80   9m31s
```
A common source of confusion is the type of a Service. By default, the Service type is ClusterIP, which means that a Pod can only be reached through the Service if queried from the same node inside of the cluster. First, check the Service type. If you think that ClusterIP is the proper type you wanted to assign, open an interactive shell from a temporary Pod inside the cluster and run a curl or wget command:
``` shell
$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
myservice    ClusterIP   10.99.155.165   <none>        80/TCP    15m
$ kubectl run tmp --image=busybox -it --rm -- wget -O- 10.99.155.165:80
...
```
Finally, check if the port mapping from the target port of the Service to the container port of the Pod is configured correctly. Both ports need to match or the network traffic wouldn’t be routed properly:
``` shell
$ kubectl get service myapp -o yaml | grep targetPort:
    targetPort: 80
$ kubectl get pods myapp-68bf896d89-qfhlv -o yaml | grep containerPort:
    - containerPort: 80
```
### Troubleshooting cluster failures
`k get nodes`
when identifying issues on a high-level:
- Is the health status for node anything else than "Ready"?
- Does the version of a node deviate from the version of other nodes?
#### troubleshooting control plane nodes
Diagnose issues: `k cluster-info`
Very detailed view of the cluster logs: `k cluster-info dump`
#### Inspecting control plane components
- kube-apiserver: Exposes the k8s api used by clients like kubectl for managing objects.
- etcd: A key-value store for storing the cluster data
- kube-scheduler: selects nodes for Pods that have been scheduled but not created yet.
- kube-controller-manager: Runs controller processes e.g. the job controller responsible for Job object execution.
- cloud-controller-manager: Links cloud provider-specific API to the k8s cluster. is not available in on-premise cluster installations of k8s.

Discover those components and their status: `k get pods -n kube-system`
retrieve the logs: `k logs kube-apiserver-minikube -n kube-system`

#### troubleshooting worker nodes
`k get nodes`
The “NotReady” state means that the node is unused and will accumulate operational costs without actually scheduling workload. There might be a variety of reasons why the node entered this state. The following list shows the most common reasons:

- Insufficent resources: The node may be low on memory or disk space.
- Issues with the kubelet process: The process may have crashed or stopped on the node. Therefore, it cannot communicate with the API server running on any of the control plane nodes anymore.
- Issues with kube-proxy: The Pod running kube-proxy is responsible for network communication from within the cluster and from the outside. The Pod transitioned into a non-functional state.

SSH into the relevant worker node(s) and start your investigation.

Checking available resources by `describe node`

Checking the kubelet process `systemctl status kubelet` `journalctl -u kubelet.service`

Checking the certificate: `openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -text`

Checking the kube-proxy pod:The kube-proxy components run in a set of dedicated Pods in the namespace kube-system. You can clearly identify the Pods by their naming prefix kube-proxy and the appended hash. Verify if any of the Pods states a different status than “Running”. Each of the kube-proxy Pods runs on a dedicated worker node. You can add the -o wide option to render the node the Pod is running on in a new column.
Have a look at the event log for kube-proxy Pods that seem to have an issue. The command below describes the Pod named kube-proxy-csrww. In addition, you might find more information in the event log of the corresponding DaemonSet.

$ kubectl describe pod kube-proxy-csrww -n kube-system
$ kubectl describe daemonset kube-proxy -n kube-system
The logs may come in handy as well. You will only be able to check the logs for the kube-proxy Pod that runs on the specific worker node.

$ kubectl describe pod kube-proxy-csrww -n kube-system | grep Node:
Node:                 worker-1/10.0.2.15
$ kubectl logs kube-proxy-csrww -n kube-system
