<p align="center">
<img src=".images/kubernetes.png" alt="Kubernetes Logo" width="300px">
</p>

# ☁️ Setup A Hobbyist Kubernetes Cluster In The Cloud For 5€/Month
This tutorial will guide you through the setup of a **Kubernetes cluster** on a VPS server.
In the end you will have:
- ☸️  &nbsp; fully-functional single-node **cluster** running on a VPS
- 🌐  &nbsp; hello-world frontend **deployed** and **accessible** from your local browser

## 🌱 Skill Prerequisites
You need **basic** understanding of 
- 💻  &nbsp; **Linux Environment**: ssh, shell, users, network stack
- ☸️  &nbsp; **Kubernetes**: high-level understanding of resources like Nodes, Pods, Deployments, etc.
- 🐳. &nbsp;**Docker**: ideally, you have used a Docker image before
 

## ☁️ VPS Hosting On Contabo
[Contabo](https://contabo.com/de/) is a standard cloud hosting provider.  
As the headline advertises, Contabo offers a VPS server with solid hardware (4vCPU, 8GB Ram, 200GB SSD) for 5€.  
The setup via their website is easy and quick. You can pay with a credit card or Paypal.
Once the server is provisioned you can access it via SSH (the password is sent via E-Mail).
```bash
ssh root@<vps-ip>
```
First, we should create a new user, change the password and add the user to the sudoers group.
````bash
useradd -m <user-name>
passwd <user-name>
usermod -aG sudo <user-name>
````
Now, you can log into the server with the new username and password.

### 🗒 Side Note
I can only recommend installing a more advanced terminal. It will speed up your productivity.
Personally, I like `zsh`: [zsh install guide](https://blog.ssdnodes.com/blog/perk-vps-better-terminal-zsh/))

## ☸️ KUBERNETES  
This section details how to install a healthy Kubernetes cluster on our VPS server.

### 🌱 Prerequisites:
Docker needs to be installed. Follow the instructions on the [docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) website.

### Kubeadm
[Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is a tool built to provide `kubeadm init` and `kubeadm join` as best-practice "fast paths" for creating Kubernetes clusters.

For convenience, make Docker executable without sudo:
```
sudo usermod -aG docker $USER
newgrp docker
```

Install Kubeadm:
```
# Adjust docker config
sudo bash -c 'cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF'

sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl restart docker

# Prepare installation
sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo bash -c "cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF"

# Install kubeadm package
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Disable Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo reboot
```

## 🕸 Setup A Network

Start the cluster with flannel as a network:
```
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```
<details>
  <summary>In case you get: sysctl: cannot stat </summary>
    $ modprobe br_netfilter && sudo sysctl -p /etc/sysctl.conf
</details>

```
su - root
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all

# tell kubectl where to find the config
export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```

Taint the master node to be able to run pods.
By default, Kubernetes is configured to run no pods on the master node for good reasons. If you plan to deploy resource-intensive applications or have security concerns
it is better to provision a multi-node cluster.
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Change back to your user:
```
su <user-name>
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
# add this to your .bashrc to make permanent
export KUBECONFIG=$HOME/.kube/config
```
Hint: Making a copy of the `.kube/config` to your local machine to access the cluster from there.

## 🤖 Test With Hello World
We will deploy a [hello-world frontend](https://hub.docker.com/r/tutum/hello-world/) using the Kubernetes resource Deployment.
We use a minimal deployment file to run our docker image.
It already contains a lot of fields but they are all relevant
and explained in more detail [here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: tutum/hello-world  # <-- Our image name
        imagePullPolicy: Always
        ports:
          - name: http
            containerPort: 80  # <-- Our container port 
```
The Deployment will take care of running the Pod containing the Docker image `tutum/hello-world`.
Store the YAML config in `frontend-deployment.yml` and create the resource on your k8s cluster
```shell
kubectl -f apply frontend-deployment.yml
```
You can check the status of your Deployment and corresponding Pod 
```shell
kubectl get pods
kubectl get deployments
```
To access the Docker container inside the Pod, inside the Deployment, inside the Cluster we will use 
port forwarding. From your local machine run
```shell
kubectl port-forward deployment/frontend 8080:80
```
Open your browser [here](http://localhost:8080).
