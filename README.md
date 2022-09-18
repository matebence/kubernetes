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

- API server
	- Act as a frontend for the users
 	- Everyone talks to the api server 
 	- curl -X POST /api/v1/namespaces/default/pods
- etcd
	- Its a key value store
	- For storing information about the nodes
- kubelet
	- The agent make sure the containers are running on nodes as excepted
	- They listen for action and send back information to the master node
- Container Runtime
	- It is used to run container in our case is docer
	- docker, contaierd, rocket
- Controller
	- Replication controller
		- It make desicition to scale container or restart them
	- Node controller
		- Watches the nodes (heartbeat)
- Scheduler
	- It distributes the work
	- It assignes container to the nodes
	- Which pods goes on which node
- Kube-proxy
	- Container to container communication
	- Its job is to look for new services and everytime a new  service is created it creates the appropriate rules on each node to forward the traffic to services


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

Strategies:
- recreate - A deployment defined with a strategy of type Recreate will terminate all the running instances then recreate them with the newer version
- ramped (rolling update) - A ramped deployment updates pods in a rolling update fashion, a secondary ReplicaSet is created with the new version of the application, then the number of replicas of the old version is decreased and the new version is increased until the correct number of replicas is reached.

### Kubernetes controllers

- They monitor kubernetes objects
- Replication controller supports the high avaibility
  - For scaling our pods and restarting on failure

There are two terms:
- Replication Controller 
  - Its the older technology
  - And replaced by 'Replica set'
- Replica set
  - Its the new recommended way to setup replication

**DaemonSets**

- Its similar to replication set
- It helps to deploy multiple instances of pods 
- But it runs one copy on each node on the cluster 
- If new node added then the pod gets created there
- If the node is removed then the pod is removed
- It good for Monitoring and logs solution 

- Lets say you use ReplicaSet-A for controlling your pods, then You wish to update your pods to a newer version, now you should create Replicaset-B, scale down ReplicaSet-A and scale up ReplicaSet-B by one step repeatedly (This process is known as rolling update). Although this does the job, but it's not a good practice and it's better to let K8S do the job.

- A Deployment resource does this automatically without any human interaction and increases the abstraction by one level.

### Services (Networking)

- **--type=ClusterIP** 
  - default, accesible only inside from the cluster
- **--type=NodePort** 
  - it will be exposed via the worker node ip, and it will be accesible from outside 
  - this is also loadbalanced
  - 30000-32767
- **--type=LoadBalancer** 
  - the loadbalancer will generate a unique address and it will evenly distribute all the traffic

![Services](https://raw.githubusercontent.com/matebence/kubernetes/cert/services.png)

In pods where we have multiple containers we can use:
- http://localhost

For pod to pod communication we have two options
- env: [SERVICE_NAME]_SERVICE_HOST
- Code DNS: http//service-name.namespace

### Network policies

- Ingress means that traffic flows into a pod (not request/response logic)
- Egress means that traffic flows out of a pod (not request/response logic)

![Network policies](https://raw.githubusercontent.com/matebence/kubernetes/cert/ingress_egress.png)

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

### Taints & Tolerations

Tolerations does not tell the pod to go to a particular node. Instead it tells the node to accept pod with certain tolerations.

- **NoeSchedule** - The pod will not be scheduled on the node
- **PreferNoSchedule** - It will try not to schedule a pod on the node, but its not guaranteed
- **NoExecute** - New pods will not be scheduled on the node and the existing pods will be evicted(removed and not recreated on other nodes) if they dont tolerait the taint

### Affinity
	
Two types of affinities:
- During scheduling means it is created the first time
- During execution means means that if the label is removed from the node the pods are going to ignore it

K8S options:
- **requiredDuringSchedulingIgnoredDuringExecution** - If there is no label on the node what could match then the pod is not created
- **preferredDuringSchedulingIgnoredDuringExecution** - If there is no label on the node what could match then the pod is created, but on diff node
- **requiredDuringSchedulingRequiredDuringExecution** - If there is no label on the node what could match then the pod is not created and if the label is removed from the node the pods gets destroyed

### Static pods

```bash
ps -ef | grep kubelet
grep -i static /var/lib/kubelet/config.yaml
cd /etc/kubernetes/manifests
```

if we places pod.yaml files in this direcotry pods are going to be created by the kubelet without kube-apisever and when we delete the file it gets automatically deleted (only pods)

### [Multi-Container PODs](https://betterprogramming.pub/understanding-kubernetes-multi-container-pod-patterns-577f74690aee)

Patterns for multi container pods:
- [InitContainers](https://medium.com/bb-tutorials-and-thoughts/kubernetes-learn-init-container-pattern-7a757742de6b) - In Kubernetes, an init container is the one that starts and executes before other containers in the same Pod. It's meant to perform initialization logic for the main application hosted on the Pod. For example, create the necessary user accounts, perform database migrations, create database schemas and so on.
- **Ambassador** - The ambassador pattern derives its name from an Ambassador, who is an envoy and a person a country chooses to represent their country and connect with the rest of the world. Similarly, in the Kubernetes perspective, an Ambassador pattern implements a proxy to the external world
- **Adapter** - The Adapter is another pattern that you can implement with multiple containers. The adapter pattern helps you standardise something heterogeneous in nature. For example, you’re running multiple applications within separate containers, but every application has a different way of outputting log files.
- **Sidecar** - Sidecars derive their name from motorcycle sidecars. While your motorcycle can work fine without the sidecar, having one enhances or extends the functionality of your bike, by giving it an extra seat. Similarly, in Kubernetes, a sidecar pattern is used to enhance or extend the existing functionality of the container.

### Object creation

- Imperatively - command
- Declaratively - .yml files

## The basics - imperatively approach

```bash
##creating deployment
kubectl create deployment my-app --image=nginx:latest

##creating pod
kubectl run my-app --image=nginx:latest

## creating namespace
kubectl create namespace dev

## creating service
kubectl expose deployment my-app --type=NodePort --port=80

## expose to host via minikube
minikube service my-app


## remove pod
kubectl delete pod my-app

## remove deployment
kubectl delete deployment my-app

## remove namespace
kubectl delete namespace dev

## remove service
kubectl delete service my-app

## remove pod
kubectl delete pod my-app-sd789qw3



## converting imperative to declarative (deployment, pod etc ...)
kubectl create deployment my-app --image=nginx:latest --dry-run=client -o yaml > my-app-deployment.yaml



## get informations
kubectl get all
kubectl get pods
kubectl get pods -o wide
kubectl get services
kubectl get deployments
kubectl get configmaps
kubectl get persistentVolumes
kubectl get persistentVolumeClaims
kubectl get replicasets
kubectl get namespaces
kubectl get resourcequotas
kubectl get ingress
kubectl describe deployment my-app
kubectl exec -it my-app -- bash
kubectl exec my-app -- env



## get logs (for more information we install metrics server or prometheus)
kubectl logs -f myapp
kubectl logs myapp



## filter based on namespaces
kubectl -n your-namespace get pods
kubectl get pods --all-namespaces



## filter based on labels
kubectl get pods --show-labels
kubectl get pods -l env=dev
kubectl get pods -l env=dev,bu=finance



## set default namespace
kubectl config set-context $(kubectl config current-context) --namespace=dev



## editing existing resources (deployment, pod etc ...)
## if we change the image it will be pulled
kubectl edit deployment my-app



## set replicas
kubectl scale deployment firstapp --replicas=3

## changing image, no additional update needed
## it will automatically pull the image
kubectl set image deployment my-app nginx=nginx:perl

## show deployment history
kubectl rollout history deployment my-app

## show detailed deployment history
kubectl rollout history deployment my-app --revision=1

## rollback last deployment
kubectl rollout undo deployment my-app

## rollback to specific deployment
kubectl rollout undo deployment my-app --to-revision=2

## update if the tag is always latest
kubectl rollout restart deployment my-app

## check the update status
kubectl rollout status deployment my-app



## backup .yml files
kubectl get all --all-namespaces -o yaml > all-deploy-services
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

## Most used Objects

### Pod creation

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

### Pod creation - Security Context (Pod Level - all containers)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  securityContext:
    runAsUser: 1000
  containers:
    - name: nginx
      image: nginx:latest
```

### Pod creation - Security Context (Container Level - one containers)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx
      image: nginx:latest
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```

### Deployment creation

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

**Recreate**

```yaml
spec:
  replicas: 3
  strategy:
    type: Recreate
```

**Ramped**
```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # how many pods we can add at a time
      maxUnavailable: 0  # maxUnavailable define how many pods can be unavailable
                         # during the rolling update
```

## Networking

### Service creation

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: rest-api
  ports:
    - port: 40
      targetPort: 40
  type: ClusterIP
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: spa
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30200
  type: NodePort
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: pwa
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

**API versioning with service selectors**

```yaml
apiVersion: v1
kind: Service
metadata:
 name: my-app
 labels:
   app: my-app
spec:
 type: NodePort
 ports:
 - name: http
   port: 8080
   targetPort: 8080

 # Note here that we match both the app and the version.
 # When switching traffic, we update the label “version” with
 # the appropriate value, ie: v2.0.0
 selector:
   app: my-app
   version: v1.0.0
```

### Network policies

- (matchLabels && namespaceSelector) || (ipBlock[outside k8s]) -> if criterias are met then incoming traffic is allowed
- (matchLabels && namespaceSelector) || (ipBlock[outside k8s] ) -> if criterias are met then outcoming traffic is allowed

```yaml
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
      - ipBlock:
          cidr: 192.168.99.10/32
      ports:
        - protocol: TCP
          port: 3306
  egress:
    - to: 										
      - ipBlock:
          cidr: 192.168.99.10/32
      ports:
        - protocol: TCP
          port: 80
```

### Ingress (Load balancer)

- https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /pay
        pathType: Prefix
        backend:
          service:
            name: pay-service
            port:
              number: 8282
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  rules:
  - host: pay.foo.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: pay-service
            port:
              number: 8282
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
    service:
      name: test
      port:
        number: 80
```

## ENV CMD ARGS

### environment variables

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

### Using CMD(ARGS) and ENTRYPOINTS(command)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myos-pod
  labels:
    app: myos
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      args: ["10"]
      command: ["sleep2.0"]
```

## Storage

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

## Listeners

### Creating replicaset

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
spec: 
  selector: 
    matchLabels:
      app: my-app
  replicas: 3
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

### Creating DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

## Namespaces

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    name: dev
```

## Resource limits

When several users or teams share a cluster with a fixed number of nodes, there is a concern that one team could use more than its fair share of resources. Resource quotas are a tool for administrators to address this concern.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quata
  namespace: dev
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.nvidia.com/gpu: 4
```

If a container requests a resource, Kubernetes will only schedule it on a node that can give it that resource. Limits, on the other hand, make sure a container never goes above a certain value.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx
      image: nginx:latest
      resources:
        requests:
          memory: "1Gi"
          cpu: 1
        limits: 						
          memory: "2Gi"
          cpu: 2
```

## Selectors via expressions

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

## Scheduling

### Manual scheduling on new Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx
      image: nginx:latest
  nodeName: worker-node02
```

### Creating Tolerations and taints

```bash
kubectl taint nodes worker-node02 app=myapp:NoSchedule
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx
      image: nginx:latest
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "myapp"
      effect: "NoSchedule"
```

### Creating Node selector

```bash
kubectl label nodes worker-node01 type=SSD
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx
      image: nginx:latest
  nodeSelector:
    type: SSD
```yaml

### Creating Affinities

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx
      image: nginx:latest
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: type
              operator: In
              values:
                - SSD
                - RAM
```

## Security

```bash
openssl genrsa -out bence.key 2048
openssl req -new -key bence.key -out bence.csr

export BASE64_CSR=$(cat ./bence.csr | base64 | tr -d '\n')
```

### Create K8S CertificateSigningRequest

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: bence
spec:
  request: ${BASE64_CSR}
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

```bash
cat cert.yaml | envsubst | kubectl apply -f -
```

```bash
kubectl get csr
kubectl certificate approve bence
kubectl get csr bence -o yaml

kubectl get csr bence -o jsonpath='{.status.certificate}'| base64 -d > bence.crt

kubectl create ns github
```

### Create Role

```bash
kubectl api-resources --api-group apps -o wide
kubectl api-resources --api-group "" -o wide
kubectl api-resources -o wide
kubectl api-resources
```

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 namespace: github
 name: git
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["create", "get", "update", "list", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "get", "update", "list", "delete"]
```

### Create RoleBinding based on user

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 namespace: github
 name: git
subjects:
- kind: User
  name: bence
  apiGroup: rbac.authorization.k8s.io
roleRef:
 kind: Role
 name: git
 apiGroup: rbac.authorization.k8s.io
```

### Create RoleBinding based on group

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: git
 namespace: github
subjects:
- kind: Group
  name: github
  apiGroup: rbac.authorization.k8s.io
roleRef:
 kind: Role
 name: git
 apiGroup: rbac.authorization.k8s.io
```

### Adjust KubeConfig

```yaml
apiVersion: v1
kind: Config
preferences: {}
current-context: bence@local-kubernetes
clusters:
- cluster:
    certificate-authority-data: cat ./ca.crt | base64 | tr -d '\n'
    server: https://10.0.0.10:6443
  name: local-kubernetes
contexts:
- context:
    cluster: local-kubernetes
    namespace: github
    user: bence
  name: bence@local-kubernetes
users:
- name: bence
  user:
    client-certificate-data: cat ./bence.crt | base64 | tr -d '\n'
    client-key-data: cat ./bence.key | base64 | tr -d '\n'
```

```bash
kubectl config view
kubectl config use-context bence@local-kubernetes
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

## Additional configurations (Liveness Probe and Image Pull Policy)

- on any change the image will be repulled

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
