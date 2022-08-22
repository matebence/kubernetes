
# Kubernetes

## Installing Google Chrome

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo apt install ./google-chrome-stable_current_amd64.deb
```


## Installing Kubernetes via minikube

### Update system

```bash
sudo apt update
sudo apt install apt-transport-https
```

### If a reboot is required after the upgrade then perform the process

```bash
[ -f /var/run/reboot-required ] && sudo reboot -f
```

### Install KVM or VirtualBox Hypervisor

```bash
sudo apt install virtualbox virtualbox-ext-pack
```

### Download minikube

```bash
wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

### Install kubectl

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### Check version

```bash
kubectl version -o json  --client
```

### Creating docker group and assign currently logged user

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
sudo shutdown -r now
```

### Remove temp data

```bash
sudo rm -rf /tmp/juju-mk*
sudo rm -rf /tmp/minikube.*
```

### Assigning permissions

```bash
sudo chown -R $USER $HOME/.minikube
chmod -R u+wrx $HOME/.minikube
mkdir $HOME/.docker
sudo chown -R $USER $HOME/.docker
chmod -R u+wrx $HOME/.docker
```

### Starting minikube

```bash
minikube start
```

### Minikube Basic operations

```bash
kubectl cluster-info
kubectl config view
kubectl get nodes
```

```bash
minikube ssh
minikube stop
minikube delete
```

### Enable Kubernetes Dashboard

```bash
minikube addons list
minikube dashboard
minikube dashboard --url
```

### Put in background

```bash
CTRL + Z
bg %1
```

## Installing Kubernetes via kubeadm (Docker)

### Installing VirtualBox

```bash
sudo apt update
sudo apt install virtualbox
```

### Installing Vagrant

```bash
curl -O https://releases.hashicorp.com/vagrant/2.2.9/vagrant_2.2.9_x86_64.deb
sudo apt install ./vagrant_2.2.9_x86_64.deb
vagrant --version
```

### Vagrantfile, Kubeadm Scripts & Manifests

```bash
git clone https://github.com/matebence/kubeadm-scripts.git
```

```bash
vagrant status
vagrant up

vagrant destroy
vagrant halt

vagrant ssh master
vagrant ssh node01
vagrant ssh node02
```

Following high-level steps are required to setup Kubernetes via kubeadm:
- Install container runtime on all nodes- We will be using Docker.
- Install Kubeadm, Kubelet, and kubectl on all the nodes.
- Initiate Kubeadm control plane configuration on the master node.
- Save the node join command with the token.
- Install the Calico network plugin.
- Join worker node to the master node (control plane) using the join command.
- Validate all cluster components and nodes.
- Install Kubernetes Metrics Server
- Deploy a sample app and validate the app

### Enable iptables Bridged Traffic on all the Nodes

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

### Disable swap on all the Nodes

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Install Docker Container Runtime On All The Nodes

### Install the required packages for Docker

```bash
sudo apt-get update -y
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

### Add the Docker GPG key and apt repository

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install the Docker community edition

```bash
sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

### Add the docker daemon configurations to use systemd as the cgroup driver

```bash
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### Start and enable the docker service.

```bash
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Creating docker group and assign currently logged user

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
sudo shutdown -r now
```

## Install Kubeadm & Kubelet & Kubectl on all Nodes

### Install the required dependencies

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

### Add the GPG key and apt repository

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Update apt and install kubelet, kubeadm, and kubectl

```bash
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
```

## Initialize Kubeadm On Master Node To Setup Control Plane

```bash
IPADDR="10.0.0.10"
NODENAME=$(hostname -s)

sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd

sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=192.168.0.0/16 --node-name $NODENAME --ignore-preflight-errors Swap
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Install Calico Network Plugin for Pod Networking

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## Join Worker Nodes To Kubernetes Master Node (example)

```bash
sudo kubeadm join 10.0.0.10:6443 --token 8s6mcn.vvau2di3348o7j1n --discovery-token-ca-cert-hash sha256:41a332b1bed6bf02a3b05ab037a64db74715c0e254489549f38e11e2926d8189 
```

### Check, until status is ready (5m)

```bash
watch kubectl get nodes
```

## Setup Kubernetes Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
### If Local env
```bash
kubectl edit deploy -n kube-system metrics-server
```

#### Set insecure tls

```yaml
- --kubelet-insecure-tls
```

## Setup Kubernetes dashboard

```bash
kubectl apply -f 
https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

### Creating a Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

### Creating a ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

### Getting a Bearer Token

```yaml
kubectl -n kubernetes-dashboard create token admin-user
```
