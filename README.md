
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
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd

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
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

### Creating a Service for Kubernetes dashboard

```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: kubernetes-dashboard
  name: kubernetes-dashboard-service-np
  labels:
    k8s-app: kubernetes-dashboard
spec:
  type: NodePort
  ports:
  - port: 8443
    nodePort: 30002
    targetPort: 8443
    protocol: TCP
  selector:
    k8s-app: kubernetes-dashboard
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

```bash
 firefox https://10.0.0.10:30002/#/login
```

![Kubernetes](https://raw.githubusercontent.com/matebence/kubernetes/master/kubernetes.png)

## Terminology

- Cluster - A set of Node machines which are running the Containrized Application(Worker Nodes) or control other nodes (Master node)
- Nodes - physical or virtual machine with a certain hardware capacity which hosts one or multiple pods and communicate with the cluster
  - Master node - Cluster Control Plane, managing the Prods across worker nodes
  - Worker node - Host, Pods running App Containers
- Pods - Pods hold the actual running app containers + their required resources
- Containers - Normal Docker containers
- Services - Exposes Pods to the cluster or externally to the outside word

## Kubernetes has the following components

API server
  Act as a frontend for the users
  Every one talks to the api server 
etcd
  Its a key value store
  For storing information about the nodes
kubelet
  Is an agent that run on each nodes
  the agent make sure the containers are running on nodes as excepted
Container Runtime
  It is used to run container in our case is docer
Controller
  It make desicition to scale container or restart them
Scheduler
  It distributes the work
  It assigned container to the nodes

## Worker node

- Pods are created and managed by Kubernetes (Master node) and placed on the Worker nodes
- A pod hosts one or more application containers and their resources (volumes, IP, run config)
- On the worker node we have to have installed
  - docker - for managing containers
  - kubelet - communication between Master and worker node
  - kube-proxy (core dns) - for network communication

## Master node

- The most importat service on the master node is the API Server
  - API for the kubelets to communicate
- Scheduler
  - Watches for new Pods, select Worker nodes to run them on
- Kube-Controller-Manager 
  - Watches and controls worker nodes, correct nubmer of pods and more
- Cloud-Controller-Manager
  - Like Kube-Controller-Manager but for speicifc Cloud Provider: Knows how to interact with cloud provider resources
- Etcd 
  - key value store

## Kubernetes Objects

Kubernetes is working with Objects like pods, deployments, services, replicasets etc ... The 3 most important are:

### Pod

Contains and runs one or more multiple containers
Has a cluster internal IP by default = for communication
Containers inside a Pod can communicate via localhost

### Deployment

- Controls multiple Pods
- Deployments can be paused, deleted and rolled back
- Deployment can be scaled dynamically (and automatically)

### Services (Networking)

- --type=ClusterIP - default accesible only inside from the cluster
- --type=NodePort - it will be exposed via the worker node ip, and it will be accesible from outside
- --type=LoadBalancer - the loadbalancer will generate a unique address and it will evenly distribute all the traffi

In pods where we have multiple containers we can use:
- http://localhost

For pod to pod communication we have two options
- env: [SERVICE_NAME]_SERVICE_HOST
- Code DNS: http//service-name.namespace

### Volumes

Kubernetes can mount Volumes into Containers. Broad variety of Volumes types (NFS, EFS, CSI):
- Local volumes (on worker nodes)
- Cloud-provider specific Volumes

**Normal Volumes**
- Volume is attached to Pod and Pod lifecycle
- If pod removed volumes removed too  (depending in type)
- Defined and created together with Pods
- Repetitive ad hard to administer on a global level

**Persistent volumes**
- Volume is a standalone Clister resource (NOT ATTACHACHED TO A POD)
- Created standalone claimed via a PVC
- Can be defined once and used multiple times

**Access modes**
- ReadWriteOnce - it can be used by multiple pods but the node must be the same
- ReadOnlyMany - it is read only by multiple pods on many nodes
- ReadWriteMany - its be used by multiple pods and many nodes

### Object creation

- Imperatively - command
- Declaratively - .yml files

## The basics - imperatively approach

```bash
##creating deployment
kubectl create deployment my-app --image=nginx:latest

## creating service
kubectl expose deployment/my-app --type=NodePort --port=80

## expose to host via minikube
minikube service my-app



## remove deployment
kubectl delete deployment/my-app

## remove service
kubectl delete service/my-app

## remove pod
kubectl delete pod/my-app-sd789qw3



## get informations
kubectl get pods
kubectl get services
kubectl get deployments
kubectl describe deployment/my-app
kubectl exec -it pod/my-app -- bash



## set replicas
kubectl scale deployment/firstapp --replicas=3

## changing image, no additional update needed
kubectl set image deployment/my-app nginx=nginx:perl

## show deployment history
kubectl rollout history deployment/my-app

## show detailed deployment history
kubectl rollout history deployment/my-app --revision=1

## rollback last deployment
kubectl rollout undo deployment/my-app

## rollback to specific deployment
kubectl rollout undo deployment/my-app --to-revision=2

## update if the tag is always latest
kubectl rollout restart deployment/my-app

## check the update status
kubectl rollout status deployment/my-app
```

## The basics - declaratively approach

```yaml
# YAML explained
key: value

propertyOne: propertyOneValue
propertyTwo: propertyTwoValue
propertyThree:
	anotherDictionary: anotherDictionaryValue

arrayOfElements:
	- first-element
	- second-element

arrayOfDictionaries
	- name: testName
	  value: testValue
	  descrption: testDescription
```

```bash
### for all opearions like CRUD
kubectl create -f deployment.yaml
kubectl apply -f deployment.yaml -f service.yaml
kubectl delete -f deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: my-app
  ports:
    - protocol: 'TCP'
      port: 123
      targetPort: 80
			nodePort: 32000
  type: NodePort
```

### Merging config files via '---'

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: my-app
  ports:
    - protocol: 'TCP'
      port: 123
      targetPort: 80
      nodePort: 30200
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

### Selectors via expressions

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchExpressions:
      - {key: app, operator: In, values: [my-app]}
      - {key: app, operator: NotIn, values: [first-app, second-app]}
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

### Additional configurations (LivenessProbe and ImagePull Policy)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchExpressions:
      - {key: app, operator: In, values: [my-app]}
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
		  imagePullPolicy: Always
		  livenessProbe:
		  	httpGet:
		  		path / 						
		  		port: 8080 					
		  		httpHeaders: Authorization 	
		  	periodSeconds: 10
		  	intialDelaySeconds: 5
```

### Setting environment variables

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          env:
            - name: test
              value: 'test'
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-map
data:
  myKey: myValue
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          env:
            - name: test
              valueFrom:
                configMapKeyRef:
                  name: app-config-map
                  key: myKey
```

### Setting normal pod volumes (emptyDir & hostPaths)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          volumeMounts:
            - mountPath: /app/data
              name: my-mount
      volumes:
        - name: my-mount
          emptyDir: {}
        - name: second-mount
          hostPath:
            path: /data
            type: DirectoryOrCreate
```

### Setting persistent volumes

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
   storage: 1Gi
  volumeMode: Filesystem
  storageClassName: standard
  accessModes:
   - ReadWriteOnce      
  hostPath:
    path: /data
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  volumeName: my-pv       
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          volumeMounts:
            - mountPath: /app/data
              name: my-mount
      volumes:
        - name: my-mount
          persistentVolumeClaim:
            claimName: my-pvc
```