# Install Kubernetes the kubeadm way
## Introduction to deployment with kubeadm
### Steps
1. Have multiple systems or vm provisioned for configuring a cluster.
Once the systems are created, designate one node as master and other as worker nodes.
2. Install a container runtime on the hosts. We using Docker. Install docker on all the nodes.
3. Install kubeadm tool on all the nodes, the kubeadm tool helps us bootscrap the k8s solution by installing and confgiguring all the required
components in the right nodes in the right order.
4. Initialize the master server. During this process all the required components are installed and confgigured on the master server.
Once the master is initialized and before joining the worker nodes to the master, you must ensure that the network prerequisites are met and
normal network connectivity between the systems is not sufficient for this.
K8s requires a special networking solution between the master and worker nodes, which is called as the pod network.
5. join the worker node to the master node.
## Deploy with kubeadm-Provision VM with Vagrant
[repo](https://github.com/kodekloudhub/certified-kubernetes-administrator-course.git "repo")
vagrant ssh kubemaster
## Demo-deployment with kubeadm
[install kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
``` bash
lsmod | grep br_netfilter
modprobe br_netfilter
```
run on all the nodes
### letting iptables see bridged traffic
``` bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

### Install Docker Engine on ubuntu:
``` bash
apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
systemctl enable docker.service
systemctl enable containerd.service
systemctl edit docker.service
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://127.0.0.1:2375
systemctl daemon-reload
systemctl restart docker.service
systemctl status docker.service
```
### Installing kubeadm, kubelet and kubectl
- kubeadm: the command to bootstrap the cluster.
- kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
- kubectl: the command line util to talk to your cluster.

`kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.56.2`
`kubeadm join 10.26.82.9:6443 --token 5oze05.sgcn4a5j8or1ho3m \
        --discovery-token-ca-cert-hash sha256:cd674108921cedba78804f3d5632503f098a53ecc36b4c1217bcd9cac8d308da `
Check kubeadm version: `kubeadm version -o short`
