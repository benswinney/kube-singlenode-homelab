# Home Lab Setup
Steps to configure homelab kubernetes cluster - specific to my set up, but could be easily replicated within other environments

### Operating System 
Bare Metal install of Ubuntu Server 18.04 LTS on both Master and Worker nodes

## Pre-Reqs
```shell
sudo apt install libnss3-tools
```

```shell
wget https://github.com/FiloSottile/mkcert/releases/download/v1.4.0/mkcert-v1.4.0-linux-amd64
mv mkcert-v1.4.0-linux-amd64 mkcert
chmod +x mkcert
sudo mv mkcert /usr/local/bin/mkcert
```

Create x509 Certificates
```shell
# install local CA
mkcert -install
mkcert '*.home.swinney.io'
mkdir -p cert && mv *-key.pem cert/key.pem && mv *.pem cert/cert.pem
```

Install GoLANG
```shell
wget https://dl.google.com/go/go1.13.linux-amd64.tar.gz
sudo tar -xvf go1.13.linux-amd64.tar.gz
sudo mv go /usr/local
```

Set GOROOT and GOPATH in .profile
```shell
GOROOT=/usr/local/go
GOPATH=$HOME/go-projects
PATH=$GOPATH/bin:$GOROOT/bin:$PATH
```

Install GCC
```shell
sudo apt update
sudo apt install -y gcc
```

Install CFSSL
```shell
sudo apt install golang-cfssl
```

Install Istioctl
```shell
```

## Kubernetes Configuration
Multi node deployment, with Pods deployable on Master node, suits my homelab environment.
Deep Learning Worker node is marked for ML/DL deployments only. No other Pods will run on it.

### Master Node

### Configure Kubernetes & Docker Repositories
```shell
sudo apt update && sudo apt install -y apt-transport-https curl ca-certificates software-properties-common nfs-common
```

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```shell
sudo add-apt-repository \
  "deb https://apt.kubernetes.io/ kubernetes-xenial main"
```

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo pt-key add -
```

```shell
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
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

### Hold Docker Version
```shell
sudo apt-mark hold docker-ce=18.06.2~ce~3-0~ubuntu
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

### Apply a network model (Flannel or Secure Weave)
```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```shell
kubectl create secret -n kube-system generic weave-passwd --from-literal=weave-passwd=$(hexdump -n 16 -e '4/4 "%08x" 1 "\n"' /dev/random)
kubectl apply -n kube-system -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&password-secret=weave-passwd"
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
kubectl apply -f dashboard/dashboard-admin.yaml
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
Connect via https://<b><ExternalIP></b>

### Install Heapster for cluster metrics and health info
```shell
helm install stable/heapster --name heapster --set rbac.create=true
```

### Install Traefik for LoadBalancing/Ingress - We'll use Node-labels to prevent pods being deployed on Deep Learning Worker 
```shell
kubectl apply -f traefik/traefik-service-acc.yaml
kubectl apply -f traefik/traefik-cr.yaml
kubectl apply -f traefik/traefik-crb.yaml
kubectl apply -f traefik/traefik-deployment.yaml
kubectl apply -f traefik/traefik-svc.yaml
kubectl apply -f traefik/traefik-webui-svc.yaml
kubectl apply -f traefik/traefik-webui-ingress.yaml
```

## Metric Server
```shell
kubectl create -f 1.8-metricserver/*.yaml
```

## Add DeepLearning Worker Node 
A lot of this could be done during the initial Master/Worker build

### Install Nvidia GPU Drivers

```shell
TBC
```

REBOOT!

```shell
sudo apt update && sudo apt install -y apt-transport-https curl ca-certificates software-properties-common nfs-common
```

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```shell
sudo add-apt-repository \
  "deb https://apt.kubernetes.io/ kubernetes-xenial main"  
```

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo pt-key add -
```

```shell
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

### Add nvidia-docker2 repository for passing GPU's into docker containers
```shell
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -

distribution=$(. /etc/os-release;echo $ID$VERSION_ID)

curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
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

### Hold Docker Version
```shell
sudo apt-mark hold docker-ce=18.06.2~ce~3-0~ubuntu
```

### Install nvidia-docker2
```shell
sudo apt-get install nvidia-docker2
```

### Modify Docker to use systemd driver, overlay2 storage driver and default nvidia-docker2 runtime
```shell
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
          "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "default-runtime": "nvidia",
  "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
  }
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

RBEOOT!

### Add Deep Learning Worker node to Cluster

Create token for joining cluster and list it out
```shell
kubeadm token create && kubeadm token list 
```

List token-ca-cert-hash
```shell
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

On Worker Node
```shell
sudo kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

Example:
```shell
sudo kubeadm join --token zo9ju3.ezaz85c7oha3x9jp kube:6443 --discovery-token-ca-cert-hash sha256:6877abd8a6f646680ad1fd8ef0373d128890715decfbb1724d75431dd8bbdd80
```

Confirm on master node that the worker node has joined
```shell
kubectl get nodes -o wide
```

### Add NodeSelector Label to Deep Learning Worker Node
```shell
kubectl label nodes deeplab.home.swinney.io workload=mldl
```

Confirm label has been applied
```shell
kubectl get nodes --show-labels
```

### Enable GPU Support within Kubernetes
On Master Node, install Nvidia DevicePlugin DaemonSet
```shell
kubectl create -f deviceplugins/nvidia-device-plugin.yml
```


## Install homelab utilities

### PiHole Ad Blocker for Kube

### UniFi

### Transmission

## Media Server

## Plex Media Server




## KubeFlow for ML/DL

### Install kfctl
```shell
opsys=linux

curl -s https://api.github.com/repos/kubeflow/kubeflow/releases/latest |\
    grep browser_download |\
    grep $opsys |\
    cut -d '"' -f 4 |\
    xargs curl -O -L && \
    tar -zvxf kfctl_*_${opsys}.tar.gz
```

### Install KubeFlow
```shell
# Add kfctl to PATH, to make the kfctl binary easier to use.
export PATH=$PATH:"<path to kfctl>"
export KFAPP="<your choice of application directory name>"

# Installs Istio by default. Comment out Istio components in the config file to skip Istio installation. See https://github.com/kubeflow/kubeflow/pull/3663
export CONFIG="https://raw.githubusercontent.com/kubeflow/kubeflow/v0.6-branch/bootstrap/config/kfctl_k8s_istio.0.6.2.yaml"

kfctl init ${KFAPP} --config=${CONFIG} -V
cd ${KFAPP}
kfctl generate all -V
kfctl apply all -V
```
