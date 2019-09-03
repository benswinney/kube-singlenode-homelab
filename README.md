# Home Lab Setup
Steps to configure homelab kubernetes cluster - specific to my set up, but could be easily replicated within other environments

### Operating System 
Bare Metal install of Ubuntu Server 18.04 LTS.

## Kubernetes Configuration
Single node deployment, suits my homelab environment. Will switch to multi-node deployment in the future.

### Configure Kubernetes & Docker Repositories
```shell
sudo apt update && sudo apt install -y apt-transport-https curl ca-certificates software-properties-common nfs-common
```

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```shell
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo pt-key add -
```

```shell
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable
```

```shell
sudo apt update
```

### Disable Swap
```shell
sudo swapoff -a
```
Check /etc/fstab file and comment out the swap mounting point

### Install Docker packages
```shell
sudo apt-get install docker-ce=18.06.2~ce~3-0~ubuntu
```

### Modify Docker to use systemd driver and overlay2 storage driver
```shell
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
```

```shell 
sudo mkdir -p /etc/systemd/system/docker.service.d
```

### Restart docker
```shell
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### Install Kubernetes Packages
```shell
sudo apt install -y kubelet kubeadm kubectl
```

### Initialize Kubernetes node on the master node
```shell 
sudo kubeadm init --config kubeinit.conf
```

### Allow non-root user to run kubectl and configure Kubernetes
```shell 
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Apply a network model (Flannel)
```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### Configure as a single node cluster
```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Set up helm
```shell
kubectl create -f helm/helm-rbac.yaml
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
sudo snap install helm --classic
helm init --upgrade
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
### Install metalLB for Bare-Metal LB
```shell
helm install --name=metallb --namespace=metallb-system -f metallb/metallb-values.yaml stable/metallb
```

### Install NFS Storage Provisioner & set as default storage class
```shell
kubectl create -f nfs-client/deploy/rbac.yaml
kubectl apply -f nfs-client/deploy/deployment.yaml
kubectl apply -f nfs-client/deploy/class.yaml
kubectl patch deployment nfs-client-provisioner -p '{"spec":{"template":{"spec":{"serviceAccount":"nfs-client-provisioner"}}}}'
kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Install Kubernetes Dashboard
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
```

### Edit the Kubernetes Dashboard service in order to expose it via MetalLB.
Change spec.type to LoadBalancer:
```shell
kubectl --namespace kubernetes-dashboard edit service kubernetes-dashboard
kubectl apply -f dashboard-admin.yaml
```

### Get Token for Kubernetes Dashboard
```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

#### Example
```shell
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXNtbTQyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3YjlkMzk4Ni1kYTQyLTQwMTUtOWI4ZC1mYjgzNzgxM2I1YTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.jjGAVDeJJBIXe7jzSbmC_azlT5MAnH3yemX81m9Bv9W_I5u2Nm9aezTPZyRnO46UN7Eb2piWH5fUeNCiVZylPQt-FI4L4BGLEl5RWJInckollrSRw2bhEBkdtmEdHWjqsKXNQLV2qbuTin6ZE4lpuMa0PbkCkX-wtdpf0ejnq_PIIEdkOAvrYOKzIO6LHAEkCtK4nFObwEGPUH1yDoIbGCbdlg_xbEx-6Uv7Xz8YfbZ3DBDljcL_tyk8LwmaUWmNryTNclWBXNPOKnqrfkx1DEdj6RXTrG9TIbaIJ8YW324PmYPkPt_MDGQNxDDwpWAgH7BsogOcb7XWRGuix16_pQ
```

### Get Kubernetes Dashboard ExternalIP
```shell
kubectl --namespace kubernetes-dashboard get service kubernetes-dashboard
```
Connect via https://<ExternalIP>

### Install Heapster for cluster metrics and health info
```shell
helm install stable/heapster --name heapster --set rbac.create=true
```

### Install Traefik for LoadBalancing (although single node is set up)
```shell
kubectl apply -f traefik/traefik-service-acc.yaml
kubectl apply -f traefik/traefik-cr.yaml
kubectl apply -f traefik/traefik-crb.yaml
kubectl apply -f traefik/traefik-deployment.yaml
kubectl apply -f traefik/traefik-svc.yaml
kubectl apply -f traefik/traefik-webui-svc.yaml
kubectl apply -f traefik/traefik-webui-ingress.yaml
```
