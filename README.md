# homelab
Home Lab Setup

# OS
Ubuntu Bare Metal - 18.04

# Kube

For each node run the following:

Configure repo:

sudo apt update && sudo apt install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt update

Disable swap:

swapoff -a
Check /etc/fstab file and comment out the swap mounting point
Install packages:

apt install -y  docker.io kubelet kubeadm kubectl
systemctl enable docker
Make sure that the br_netfilter module is loaded before this step. This can be done by running:

lsmod | grep br_netfilter
In order to load it, explicitly call:

modprobe br_netfilter

Then initialize Kubernetes node on the master node:

kubeadm init -f kubeinit.conf 
