# homelab
Home Lab Setup

# OS
Ubuntu Bare Metal - 18.04

# Kube

For each node run the following:

Configure repo:

sudo apt update && sudo apt install -y apt-transport-https curl ca-certificates software-properties-common nfs-common
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt update

Disable swap:

swapoff -a
Check /etc/fstab file and comment out the swap mounting point
Install packages:

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
apt-get update && apt-get install docker-ce=18.06.2~ce~3-0~ubuntu
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d
# Restart docker.
systemctl daemon-reload
systemctl restart docker

apt install -y kubelet kubeadm kubectl

Make sure that the br_netfilter module is loaded before this step. This can be done by running:

lsmod | grep br_netfilter
In order to load it, explicitly call:

modprobe br_netfilter

Then initialize Kubernetes node on the master node:

kubeadm init --config kubeinit.conf 

# As non-root
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Apply network model
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

make this node a standalone one:

kubectl taint nodes --all node-role.kubernetes.io/master-

# Install Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml

Edit the service in order to expose it. Change ClusterIP to NodePort:

kubectl --namespace kubernetes-dashboard edit service kubernetes-dashboard

kubectl apply -f dashboard-admin.yaml

# Get Token for dashboard
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

# Example:

eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXNtbTQyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3YjlkMzk4Ni1kYTQyLTQwMTUtOWI4ZC1mYjgzNzgxM2I1YTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.jjGAVDeJJBIXe7jzSbmC_azlT5MAnH3yemX81m9Bv9W_I5u2Nm9aezTPZyRnO46UN7Eb2piWH5fUeNCiVZylPQt-FI4L4BGLEl5RWJInckollrSRw2bhEBkdtmEdHWjqsKXNQLV2qbuTin6ZE4lpuMa0PbkCkX-wtdpf0ejnq_PIIEdkOAvrYOKzIO6LHAEkCtK4nFObwEGPUH1yDoIbGCbdlg_xbEx-6Uv7Xz8YfbZ3DBDljcL_tyk8LwmaUWmNryTNclWBXNPOKnqrfkx1DEdj6RXTrG9TIbaIJ8YW324PmYPkPt_MDGQNxDDwpWAgH7BsogOcb7XWRGuix16_pQ

Start kube proxy in a screen:

kubectl proxy &

# Get dashboard NodePort
kubectl --namespace kubernetes-dashboard get service kubernetes-dashboard

# NFS Storage Provisioner

kubectl create -f nfs-client/deploy/rbac.yaml
kubectl apply -f nfs-client/deploy/deployment.yaml
kubectl apply -f nfs-client/deploy/class.yaml

# Set as default storage class
kubectl patch deployment nfs-client-provisioner -p '{"spec":{"template":{"spec":{"serviceAccount":"nfs-client-provisioner"}}}}'
kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Set up helm

kubectl create -f helm/helm-rbac.yaml
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
sudo snap install helm --classic
helm init --upgrade
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

# Install Heapster for cluster metrics and health info
helm install stable/heapster --name heapster --set rbac.create=true
