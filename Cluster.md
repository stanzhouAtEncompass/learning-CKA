# Cluster Maintenance
## OS Upgrades
The time it waits for a pod to come back online is known as the pod eviction timeout.
`kube-controller-manager --pod-eviction-timeout=5m0s`
After the nod goes offline, the master node waits for up to 5 minutes before considering the
node dead.
`k drain node-1 --ignore-daemonsets`
`k cordon node-1` # mark the pod unschedulable 
`k uncordon node-1`
## Software versions
The k8s release versions consists of 3 parts.
`v1.11.3` Major Minor Patch

Minor version are released very few months with new features and functionalities
Patches are relaesed more often with critical bug fixes.
![version](https://i.imgur.com/Ml5kJML.png)

## Cluster upgrade process
- kube-apiserver: X v1.10
- Controller-manager: X-1 v1.9 or v1.10
- kube-scheduler: X-1, v1.9 or v1.10
- kubelet: X-2, v1.8 or v1.9 or v1.10
- kube-proxy: X-2, v1.8 or v1.9 or v1.10

- kubectl: X+1 > X-1

at anytime k8s support only up to the recent three minor versions, when 1.13 is released, only version 1.13, v1.12 and v1.11 are supported before the relase.
v1.10 is un-supported.
**how do we upgrade**: upgrade one minor version at a time, v1.10 to v1.11, then 1.11 to 1.12 and then 1.12 to 1.13.
- if deployed on public cloud: just a few clicks `upgrade available`
- if deployed your cluster from kubeadm, then 
``` shell
k upgrade plan
k upgrade apply
```
- if deployed your cluster from scratch, then you manually upgrade yourself

Upgrading a cluster involves two major steps:
  * 1. upgrade master nodes
  * 2. upgrade the worker nodes while the master is being upgraded

Different strategies available to upgrade the worker nodes:
Strategy-1: upgrade all of them at once, but then your pods are down and users are no longer able to access the applications.
Once the upgrade is complete, the notes are back up, new pods are scheduled and users can resume access.
Require downtime.

Strategy-2: upgrade one node at time. our master upgraded and node waiting to be upgraded, we first upgrade the first node where the
workloads move to the second and third node and users are so far from there. Once the first node is upgraded and back up with an update, the second node 
where the worloads move to the first and third node. And finally, the third node where the workloads are shared between the first two,
until we have all nodes upgraded to a newer version, we then follow the same procedure to upgrade the nodes from 1.11 to 1.12 and then 1.13.

Strategy-3: add new nodes to the cluster nodes with newer software version. This is especially convenient if you're on a cloud environment where you can
easily provision new nodes and decommission old ones, nodes with the newer, softer version can be added to the cluster, move the workload over to the new.
And remove the old node. Until you finally have all new node with the new software version.
  
### kubeadm - upgrade
on master node:
``` shell
apt-get upgrade -y kubeadm=1.12.0-00
kubeadm upgrade apply v1.12.0
k get nodes
apt-get upgrade -y kubelet=1.12.0-00
systemctl restart kubelet
k get nodes
```
on worker node:
``` shell
# move the workload from node1 to other node
k drain node-1
apt-get upgrade -y kubeadm=1.12.0-00
apt-get upgrade -y kubelet=1.12.0-00
kubeadm upgrade node config --kubelet-version v1.12.0
systemctl restart kubelet
k uncordon node-1
```
### Lab
Q: What is the latest stable version available for upgrade?
A: `kubeadm upgrade plan`

Q: we will be upgrading the master node first. Drain the master node of workloads and mark it Unschedulable.
A: `k drain controlplane --ignore-daemonsets`

`k get deployments.apps`
`k version --short`

## Backup and restore methods
### Backup candiates
- Resource configuration
``` shell
k create namespace new-namespace
k create secret
k create configmap
```
- ETCD cluster
### backup
build in snashot
`ETCDCTL_API=3 etcdctl snapshot save snapshot.db`
view the status of the backup: `ETCDCTL_API=3 etcdctl snapshot status snapshot.db`
### restore
`service kube-api-server stop`
`ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup`
Then configure the existing configuration file to use the the new data directory.
![etcd.service](https://i.imgur.com/jzvv8CF.png)
``` shell
systemctl daemon-reload
service etcd restart
service kube-apiserver start
```
- Persistent volumes
#### declarative approach is the preferred approach
resource configuration -> Github
`k get all --all-namespaces -o yaml > all-deploy-services.yaml`
Tools:
- Velero
- HeptIO

If you're using a managed environment, then at times you may not even have access to the any cluster,
backup by acquiring the kube api server is probably the better way.

## Working with ETCDCTL
etcdctl is a command line client for etcd.


In all our Kubernetes Hands-on labs, the ETCD key-value database is deployed as a static pod on the master. The version used is v3.

To make use of etcdctl for tasks such as back up and restore, make sure that you set the ETCDCTL_API to 3.


You can do this by exporting the variable ETCDCTL_API prior to using the etcdctl client. This can be done as follows:
export ETCDCTL_API=3
On the Master Node:
`export ETCDCTL_API=3`
`etcdctl version`

To see all the options for a specific sub-command, make use of the -h or --help flag.
For example, if you want to take a snapshot of etcd, use:
etcdctl snapshot save -h and keep a note of the mandatory global options.
Since our ETCD database is TLS-Enabled, the following options are mandatory:
--cacert                                                verify certificates of TLS-enabled secure servers using this CA bundle
--cert                                                    identify secure client using this TLS certificate file
--endpoints=[127.0.0.1:2379]          This is the default as ETCD is running on master node and exposed on localhost 2379.
--key                                                      identify secure client using this TLS key file

Similarly use the help option for snapshot restore to see all available options for restoring the backup.
etcdctl snapshot restore -h
For a detailed explanation on how to make use of the etcdctl command line tool and work with the -h flags, check out the solution video for the Backup and Restore Lab.

### Lab
`etcdctl snapshot save --help`
`k get deployments.apps,svc,pod`
`etcdctl snapshot restore --help`
Q: Restore the original state of the cluster using the backup file.
A:
`etcdctl snapshot restore --help`
`ETCDCTL_API=3 etcdctl snapshot restore /opt/snapshot-pre-boot.db --data-dir=/var/lib/etcd-from-backup`
`cd /etc/kubernets/manifests/`
`vi etcd.yaml` update path under hostPath
check the old etcd was destoryed and recreated by: `docker ps -a| grep etcd`
`docker logs -f foobar`

### Certification Exam Tip
Here's a quick tip. In the exam, you won't know if what you did is correct or not as in the practice tests in this course. You must verify your work yourself. For example, if the question is to create a pod with a specific image, you must run the the `kubectl describe pod` command to verify the pod is created with the correct name and correct image.

