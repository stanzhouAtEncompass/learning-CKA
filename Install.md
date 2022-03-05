# Design and Install a Kubernetes Cluster
## Ask
- Purpose
  - Education
     * Minikube
     * Single node cluster with kubeadm/GCP/AWS
  - Development & Testing
    * Multi-node cluster with a Single Master and Multiple workers
    * Setup using kubeadmin tool or quick provision on GCP or AWS or AKS
  - Hosting Production Applications
    * High availability multi node cluster with multiple master nodes
    * Kubeadm or GCP or Kops on AWS or other supported platforms
    * Upto 5000 nodes
    * Upto 150,000 PODs in the cluster
    * Upto 300,000 Total Containers
    * Upto 100 PODs per Node
|   nodes | gcp            |                | aws        |                |
|---------+----------------+----------------+------------+----------------|
|     1-5 | n1-standard-1  | 1 vcpu 3.75 gb | m3.medium  | 1 vcpu 3.75 gb |
|    6-10 | n1-standard-2  | 2 vcpu 7.5 gb  | m3.large   | 2 vcpu 7.5 gb  |
|  11-100 | n1-standard-4  | 4 vcpu 15 gb   | m3.xlarge  | 4 vcpu 15 gb   |
| 101-250 | n1-standard-8  | 8 vcpu 30 gb   | m3.2xlarge | 8 vcpu 30 gb   |
| 251-500 | n1-standard-16 | 16 vcpu 60 gb  | c4.4xlarge | 16 vcpu 30 gb  |
|   > 500 | N1-standard-32 | 32 vCPU 120 GB | C4.8xlarge | 36 vCPU 60 GB  |

- Cloud or OnPrem?
  * Use Kubeadm for op-prem
  * GKE for GCP
  * Kops for AWS
  * Azure Kubernetes Service(AKS) for Azure
- Storage
  * High Performance – SSD Backed Storage
  * Multiple Concurrent connections – Network based storage
  * Persistent shared volumes for shared access across multiple PODs
  * Label nodes with specific disk types
  * Use Node Selectors to assign applications to nodes with specific disk types
- Nodes
  * Virtual or Physical Machines
  * Minimum of 4 Node Cluster (Size based on workload)
  * Master vs Worker Nodes
  * Linux X86_64 Architectur
  
  * Master nodes can host workloads
  * Best practice is to not host workloads on Master nodes
## Choosing k8s infrastructure
- Minikube
Deploys VMs
Single Node Cluster
- Kubeadm
Requires VMs to be ready
Single/multi node cluster
### Turnkey solutions
* You provision VMs
* You configure VMs
* You use scripts to deploy cluster
* You maintain VMs yourself
* Eg: k8s on AWS using KOPS

OpenShift, Cloud Foundry container runtime, VMware Cloud PKS, Vagrant

### Hosted solutions (managed solutions)
* Kubernetes-As-A-Service
* Provider provisions VMs
* Provider install K8s
* Provider maintains VMs
* Eg: Google Container Engine (GKE)

GKE, OpenShift Online, AKS, EKS

## Configure high availability
## ETCD in HA
ETCD is a distributed reliable key-value store that is Simple, Secure & Fast.
Read is easy.
One of the nodes process the write.
LEader election-RAFT
Quorum = N/2+1
Quorum of 3 = 3/2+1=2.5 ~= 2
![Quorum](https://i.imgur.com/xSU8Lvp.png)
### ETCDCTL
``` yaml
export ETCDCTL_API=3
etcdctl put name joh
etcdctl get name
etcdctl get / --prefix --keys-only
```

### Number of Nodes
![NumOfETCDnode](https://i.imgur.com/R9PUUAW.png)
3 is a good start but if you prefer a higher level of full tolerance, then 5 is better,
anything beyond that is just unnecessary.
![Our design](https://i.imgur.com/SfhXDcQ.png)

- Workloads
  - How many?
  - What kind?
    - Web
    - Big Data/Analytics
  - Application resource requirements
    - CPU intensive
    - Memory intensive
  - Traffic
    - Heavy traffic
    - Burst traffic
