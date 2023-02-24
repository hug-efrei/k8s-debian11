# k8s-debian11
k8s install on debian 11 with kubeadm, containerd et calico

# Node configuration
First, make sure you have the proper hostnames set up on all nodes:

## On master
```bash
sudo hostnamectl set-hostname "master"
```

## On worker1
```bash
sudo hostnamectl set-hostname "worker1"
```

## On worker2
```bash
sudo hostnamectl set-hostname "worker2"
```

## On all nodes
```bash
sudo hostnamectl set-hostname "master"

cat <<EOF | sudo tee /etc/hosts
192.168.1.11       master
192.168.1.12       worker1
192.168.1.13       worker2
EOF

sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system


wget https://github.com/containerd/containerd/releases/download/v1.6.16/containerd-1.6.16-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.6.16-linux-amd64.tar.gz
sudo mkdir -p /usr/local/lib/systemd/system/
sudo wget -P /usr/local/lib/systemd/system/ https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

sudo systemctl daemon-reload
sudo systemctl enable --now containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64

sudo install -m 755 runc.amd64 /usr/local/sbin/runc

wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz
`

## cluster install

### master
Edit the file ‘/etc/containerd/config.toml’ and look for the section ‘[plugins.”io.containerd.grpc.v1.cri”.containerd.runtimes.runc.options]’ and add SystemdCgroup = true
'''
#Enable systemd for runc
sudo vi /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

#Enable k8s repo
sudo apt install apt-transport-https ca-certificates gnupg gnupg2 curl software-properties-common -y
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/cgoogle.gpg
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

#install k8s
sudo apt update
sudo apt install kubelet kubeadm kubectl -y
sudo apt-mark hold kubelet kubeadm kubectl

#initiate cluster
sudo kubeadm init --control-plane-endpoint=master

#enable admin via current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
'''

kubectl apply -f https://projectcalico.docs.tigera.io/v3.25/manifests/calico.yaml
