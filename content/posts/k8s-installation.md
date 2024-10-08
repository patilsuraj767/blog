+++
title = 'Kubernetes Installation Bash/Kickstart Script'
date = 2024-08-24T08:35:28+05:30
draft = false
+++

Many times we as developers come across a scenario where we require a Kubernetes cluster for testing or for hosting some application. Minikube and kind clusters are good for the local setup, and I use them in my day to day development work on local systems, but when it comes to installing Kubernetes on cloud instances I usually prefer to have a stock/vanilla Kubernetes cluster installed using Kubeadm, it gives me a sense of having more control over the cluster. Routing, adding firewall rules and testing new CNCF projects get simpler.

I know we can directly leverage cloud services like EKS and AKS to get the Kubernetes cluster, but it comes with the extra cost. Also, somewhere in the back of mind, I also think what other cloud configuration might be getting changed by deploying EKS and AKS cluster.
Also, while testing new projects I blindly copy and paste the Kubectl commands, and suppose if by mistake we created a k8s service using type loadbalancer than cloud provider will deploy an external load balancer and charge us separately.

I have created the below script, which can be run on any instance (local VM or cloud instance) of Ubuntu to install the stock/vanilla kubernetes cluster in just a few minutes.

```
#!/bin/bash

# Installing docker

sudo apt-get update -y
sudo apt-get install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y


# Installing kubectl, kubelet and kubeadm

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -rf kubectl

sudo swapoff -a
sudo apt-get install -y apt-transport-https ca-certificates curl gpg -y
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet

# Installing cri-dockerd

VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4)
echo $VER
wget https://github.com/Mirantis/cri-dockerd/releases/download/${VER}/cri-dockerd-$(echo $VER | cut -c2- ).amd64.tgz
tar xvf cri-dockerd-$(echo $VER | cut -c2- ).amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
cri-dockerd --version
rm -rf cri-dockerd*
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service -O /etc/systemd/system/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket -O /etc/systemd/system/cri-docker.socket
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl start cri-docker.socket


# Installing cluster

kubeadm init --pod-network-cidr=192.168.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Installing calico

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml
kubectl taint nodes --all node-role.kubernetes.io/control-plane-


# Creating default storage class

mkdir -p /var/k8s-storage/pv-{0..99} ; chmod -R 777 /var/k8s-storage

for i in {0..99}; do
  kubectl create -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-$i
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  hostPath:
    path: "/var/k8s-storage/pv-$i"
EOF
done

kubectl create -f - <<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: manual
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF

```

Above script can also be pasted in the user-data/cloud-init section while creating a cloud instance.
If we use the script in the cloud-init, than to configure kubectl we will need to explicitly run the below commmands after instance it spinned up.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
