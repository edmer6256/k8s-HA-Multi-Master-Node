# Set up a Highly Available Kubernetes Cluster using kubeadm
Follow this documentation to set up a highly available Kubernetes cluster using __Ubuntu 23.04 LTS__.

This documentation guides you in setting up a cluster with two master nodes, two worker nodes and a load balancer node using HAProxy.

## Environment
|Role|FQDN|IP|OS|RAM|CPU|API Version|
|----|----|----|----|----|----|----|
|Master|k8s-master1.jem-dev.net|192.168.20.xxx|Ubuntu 23.04|4G|2|1.19|
|Master|k8s-master2.jem-dev.net|192.168.20.xxx|Ubuntu 23.04|4G|2|1.19|
|Master|k8s-master1.jem-dev.net|192.168.20.xxx|Ubuntu 23.04|4G|2|1.19|
|Worker|k8s-worker1.jem-dev.net|192.168.20.xxx|Ubuntu 23.04|1G|1|1.19|
|Worker|k8s-worker2.jem-dev.net|192.168.20.xxx|Ubuntu 23.04|1G|1|1.19|
|Worker|k8s-worker1.jem-dev.net|192.168.20.xxx|Ubuntu 23.04|1G|1|1.19|
|HAproxy|k8s-lb1.jem-dev.net|192.168.20.xxx|Ubuntu 23.04|1G|1|
|HAproxy|k8s-lb2.jem-dev.net|192.168.20.xxx|Ubuntu 23.04|1G|1|


> * Perform all the commands as root user unless otherwise specified


## Set up load balancer node
##### Install Haproxy
```
apt update && apt install -y haproxy
```
##### Configure haproxy
Append the below lines to **/etc/haproxy/haproxy.cfg**
```
frontend kubernetes-frontend
    bind 192.168.20.84:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server k8s-master1 192.168.20.24:6443 check fall 3 rise 2
    server k8s-master2 192.168.20.100:6443 check fall 3 rise 2
```
##### Restart haproxy service
```
systemctl restart haproxy
```

## On all kubernetes nodes (k8s-master1, k8s-master2, k8s-worker1, k8s-worker2)
##### Disable Firewall
```
ufw disable
```
##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
##### Install docker engine
```
{
  sudo apt-get update
  # apt-transport-https may be a dummy package; if so, you can skip that package
  sudo apt-get install -y apt-transport-https ca-certificates curl gpg
  apt update && apt install -y docker.io containerd.io
}
```
### Kubernetes Setup
##### Add Apt repository
```
{
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
}
```
##### Install Kubernetes components
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
## On any one of the Kubernetes master node (Eg: k8s-master1)
##### Initialize Kubernetes Cluster
```
kubeadm init --control-plane-endpoint="192.168.20.84:6443" --upload-certs --apiserver-advertise-address=192.168.20.24 --pod-network-cidr=192.168.0.0/16
```
Copy the commands to join other master nodes and worker nodes.
##### Deploy Calico network
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

## Join other nodes to the cluster (k8s-master2 & k8s-worker1, k8s-worker2)
> Use the respective kubeadm join commands you copied from the output of kubeadm init command on the first master.

## Downloading kube config to your local machine
On your host machine
```
mkdir ~/.kube
scp root@192.168.20.24:/etc/kubernetes/admin.conf ~/.kube/config
```

## Verifying the cluster
```
kubectl cluster-info
kubectl get nodes
kubectl get cs
```

Have Fun!!
