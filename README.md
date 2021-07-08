
# Run a single-node kubernetes cluster for 5€/month

## What this tutorial expects you to know.
You need basic understanding of 
- Linux environment
- SSH

## Contabo
[Contabo](https://contabo.com/de/) is a standard cloud hosting provider. 
As the headline advertises Contabo offer a VPS server with solid hardware for 5€.  

We will use this server to host our single-node k8s cluster.
The setup via their website is easy and quick. You can pay with credit card or paypal.
Once the server is provisioned you can access it via SSH (the passwords is sent via E-Mail).
```bash
ssh root@<vps-ip>
```
First, we should create a new user, change the password and add him to the sudoers group.
````bash
useradd -m <user-name>
passwd <user-name>
usermod -aG sudo <user-name>
````
Now, you can log into the server with the new username and password.

### Side Note
I can only recommend to install a more advanced terminal. It will speed up your productivity.
Personally, I like `zsh` but choose by your own preference ([zsh install guide](https://blog.ssdnodes.com/blog/perk-vps-better-terminal-zsh/))

## KUBERNETES  
This section details how we install a healthy kubernetes cluster on our VPS server.

#### Prerequisites:
Docker needs to be installed. Follow the instructions on the [docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) website.

### Kubeadm
[Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is a tool built to provide `kubeadm init` and `kubeadm join` as best-practice "fast paths" for creating Kubernetes clusters.
```
sudo usermod -aG docker $USER
newgrp docker

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

sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo bash -c "cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF"

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Disable Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo reboot
```

## Setup

The cluster setup requires some steps.
Start the cluster with flannel as network:
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

# Add this to roots bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```

Taint the master node to be able to run pods:
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```
Change back to your user
```
su <user-name>
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=$HOME/.kube/config
```

Let's test our cluster with a simple application. 
We will deploy a [hello-world frontend](https://hub.docker.com/r/tutum/hello-world/) using 
the Kubernetes resource Deployment.
We use a minimal deployment file to run our docker image.
It already contains a lot of fields for running just one container but they are all relevant
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
        image: tutum/hello-world
        imagePullPolicy: Always
        ports:
          - name: http
            containerPort: 80
```
The Deployment will take care of the running the Pod containing the docker image `tutum/hello-world`.
Store the YAML config in `frontend-deployment.yml` and create the resource on your k8s cluster
```shell
kubectl -f apply frontend-deployment.yml
```
You can check the status of your Deployment and corresponding Pod 
```shell
kubectl get pods
kubectl get deployments
```
To access the docker container inside the Pod, inside the Deployment, inside the Cluster we will use 
port forwarding. From your local machine run
```shell
kubectl port-forward deployment/frontend 8080:80
```
Open your browser http://localhost:8080.