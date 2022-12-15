---
title: "Clustering Kubernetes using Kubeadm"
date: 2022-03-18T06:00:20+06:00
hero: /images/posts/writing-posts/kubernetes.png
menu:
  sidebar:
    name: Clustering Kubernetes using Kubeadm
    identifier: kubernetes-post-test-id
    parent: kubernetes-category
    weight: 2
---
## 1. Eksekusi di semua nodes

1.  Update repo & packages

```bash
sudo apt-get update -y && sudo apt upgrade -y --with-new-pkgs
```

2.  Install dependencies

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

3.  Setup localtime

```bash
sudo timedatectl set-timezone Asia/Jakarta
```

4.  Setup hosts file

```bash
sudo nano /etc/hosts
...
xxx.xxx.xxx.xxx hostname
... 
```

## 2. Install Kubernetes dan Containerd

1.  Disable swap

```bash
sudo swapoff -a
```

2. Disable swap on startup in /etc/fstab:

```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

3. Create configuration file for containerd:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

4.  Load modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

5. Set system configurations for Kubernetes networking:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

6. Apply new settings:

```bash
sudo sysctl --system
```

7. Install Containerd

```bash
sudo apt-get update && sudo apt-get install -y containerd
```

8. Create default configuration file for containerd:

```bash
sudo mkdir -p /etc/containerd
```

9. Generate default containerd configuration and save to the newly created default file:

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

10. Restart & check status containerd to ensure new configuration file usage:

```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```

11. Add kubernetes repository

```bash
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
```

12. Install Kubelet, kubeadm dan kubectl

```bash
sudo apt-get -y install kubelet=1.23.0-00 kubeadm=1.23.0-00 kubectl=1.23.0-00
```

## 3. Clustering Kubernetes

### Master Nodes

1. Init kubernetes 

> Jika menggunakan flannel

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint "<vip-atau-hostname-master:6443>" --upload-certs
```

> Jika menggunakan calico

```bash
sudo kubeadm init --control-plane-endpoint "<vip-atau-hostname-master:6443>" --upload-certs
```

2. Set Kubectl access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

3. Alternatively, if you are the root user, you can run:

```bash
export KUBECONFIG=$HOME/.kube/config
```

4. Install CNI 

> Jika menggunakan Flannel

```bash
kubectl apply -f [https://raw.githubusercontent.com/coreos/**flan**nel/master/Documentation/kube-**flan**nel.yml](https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml)
```

> jika menggunakan Calico

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### Init kubernetes jika menggunakan versi dibawah 1.20.x

1. Bikin file kubeadm-config

```yaml
sudo nano kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
networking:
podSubnet: "10.244.0.0/16"
kubernetesVersion: "v1.15.1"  ## *sesuaikan dengan versi cluster*
controlPlaneEndpoint: “<vip-atau-hostname-master:6443>"
apiServer:
  certSANs:
  - "IP-Master1"
  - "hostname-master1"
  - "IP-Master2"
  - "hostname-master2"
  - "IP-Master3"
  - "hostname-master3"
  - "IP-VIP"
  - "hostname-VIP"
```

2. Init Cluster

```bash
kubeadm init --config=kubeadm-config.yaml
```

3. Sebelum join cluster, copy file ke semua master

```bash
scp -r /etc/kubernetes/pki master-node:~/
cd pki
mkdir -p /etc/kubernetes/pki/etcd
cp ca.crt ca.key sa.key sa.pub front-proxy-ca.crt front-proxy-ca.key /etc/kubernetes/pki
cp etcd/ca.crt etcd/ca.key /etc/kubernetes/pki/etcd/
```

### Join cluster

1. Join Master node ke Cluster

> In the Control Plane Node/Master node, create the token and copy the kubeadm join command (NOTE:The join command can also be found in the output from kubeadm init command):
> 

```bash
sudo kubeadm token create --print-join-command

sudo kubeadm join <join command from the previous command> --control-plane —apiserver-advertise-address=<ip-node>
```

2. Join Worker node ke Cluster

```bash
sudo kubeadm join <join command from the previous command>
kubectl label nodes <worker-hostname> type=app
```

3. Add worker label

```bash
kubectl label nodes <worker-hostname> [node-role.kubernetes.io/worker=](http://node-role.kubernetes.io/worker=)
```

## 4. Rejoin master/worker node baru ke existing cluster

```bash
master$ sudo KUBECERT=$(kubeadm init phase upload-certs --upload-certs)
master# sudo kubeadm token create --certificate-key $KUBECERT --print-join-command
worker-new$ sudo kubeadm join <join command from the previous command>
  
master-new$ sudo kubeadm join --control-plane <join command from the previous command>
```