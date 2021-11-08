## Hardware Requirements

Kubernetes is a large project, and compiling it can use a lot of
resources. We recommend the following for any physical or virtual
machine being used for building Kubernetes.

    - 8GB of RAM
    - 50GB of free disk space
    
## Installing Required Software

#### GNU Development Tools 

Kubernetes development helper scripts require an up-to-date GNU
development tools environment. The method for installing these tools
varies from system to system.

##### Installing on Ubuntu

```sh
sudo apt update
sudo apt install build-essential
```
Once you have finished, confirm that `gcc` and `make` are installed.

#### Install Docker 

```sh
intel_http_proxy='http://child-prc.intel.com:913'
proxy_flag='on'
#--------------------- Setup proxy for Docker ----------------------------------
function set_proxy_for_docker() {
  sudo mkdir /etc/systemd/system/docker.service.d
  cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=$intel_http_proxy"
Environment="HTTPS_PROXY=$intel_http_proxy"
Environment="NO_PROXY=localhost,127.0.0.0/8,10.293.154.0/16"
EOF
  sudo systemctl daemon-reload
  sudo systemctl restart docker
  echo ">>>>>>>>>>>>>>>>>>>>>>> Set Proxy for Docker <<<<<<<<<<<<<<<<<<<<<<<<<<"
}
#---------------------- Install Container Runtime ------------------------------
function docker_install() {

  # sudo apt remover docker docker-engine docker.io containerd runc
  sudo apt update
  sudo apt install -y docker.io
  sudo usermod -aG docker $USER

  # runtime config
  sudo mkdir /etc/docker
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
  sudo systemctl enable docker
  sudo systemctl daemon-reload
  sudo systemctl restart docker

  if [ $proxy_flag == "on" ]; then
    set_proxy_for_docker
  fi
}
docker_install
```

#### Install Rsync and jq 

```sh
sudo apt install -y rsync jq
```

#### Install Go

**Note:** Building and developing Kubernetes requires a very recent
version of Go. Please install the newest stable version available for
your system. The table below lists the required Go versions for
different versions of Kubernetes.

| Kubernetes     | requires Go |
|----------------|-------------|
| 1.0 - 1.2      | 1.4.2       |
| 1.3, 1.4       | 1.6         |
| 1.5, 1.6       | 1.7 - 1.7.5 |
| 1.7            | 1.8.1       |
| 1.8            | 1.8.3       |
| 1.9            | 1.9.1       |
| 1.10           | 1.9.1       |
| 1.11           | 1.10.2      |
| 1.12           | 1.10.4      |
| 1.13           | 1.11.13     |
| 1.14 - 1.16    | 1.12.9      |
| 1.17 - 1.18    | 1.13.15     |
| 1.19 - 1.20    | 1.15.5      |
| 1.21 - 1.22    | 1.16.7      |
| 1.23+          | 1.17        |

- Download Go from [here](https://golang.org/doc/install). 
- Extract the archive you downloaded into /usr/local, creating a Go tree in /usr/local/go.
```sh
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.1.linux-amd64.tar.gz
```
- Add /usr/local/go/bin to the PATH environment variable. In `/etc/profile`
```sh
export PATH=$PATH:/usr/local/go/bin
```
- Setup GOBIN and GOPATH environment variables
```sh
export GOPATH=$HOME/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOPATH/bin
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
```
- Verify
'''sh
go version
'''

#### Cloning the Kubernetes Git Repository
```sh
mkdir -p $GOPATH/src/k8s.io
cd $GOPATH/src/k8s.io
git clone https://github.com/kubernetes/kubernetes.git
git fetch origin pull/102884/head:vertial_scale
git checkout vertial_scale
cd kubernetes
```

## Building Kubernetes

#### Build K8s Binaries
```sh
KUBE_BUILD_PLATFORMS=linux/amd64 make all GOFLAGS=-v GOGCFLAGS="-N -l"
```
    - KUBE_BUILD_PLATFORMS=linux/amd64: To build binaries for a specific platform, add `KUBE_BUILD_PLATFORMS=<os>/<arch>`.
    - make all: builds all the components
    - GOFLAGS=-v: enable verbose log
    - GOGCFLAGS="-N -l": disable compiler optimizations
To build a specific part of Kubernetes use the `WHAT` environment variable. In `$GOPATH/src/k8s.io/kubernetes/`, the Kubernetes project directory, run the following command:
```sh
make WHAT=cmd/<subsystem>
```
Alternatively, you can also build the independent module under their project folder. For instance, to build api-server
```sh
cd kubernetes/cmd/kube-apiserver
go build -v
ls 
apiserver.go  app  kube-apiserver  BUILD  OWNERS
```
After the build, you can find all the binarines under `$GOPATH/src/k8s.io/kubernetes/_output/bin/`

#### Build the docker images

```sh
KUBE_BUILD_PLATFORMS=linux/amd64 KUBE_BUILD_CONFORMANCE=n KUBE_BUILD_HYPERKUBE=n make release-images GOFLAGS=-v GOGCFLAGS="-N -l"
```
    - KUBE_BUILD_CONFORMANCE=n KUBE_BUILD_HYPERKUBE=n: disable the building of hyperkube-amd64 and conformance-amd64 images
    - make release-images: Build all the component's docker images under kubernetes/_output/release-tars/amd64 directory

#### Load and tag the docker images

```sh
cd _output/release-tars/amd64
docker load -i kube-api-server.tar
docker load -i kube-controller-manager.tar
docker load -i kube-scheduler.tar 
docker load -i kube-proxy.tar
```
Tag the docker images to replace downloaded images with the specified the kubernetes version. (we are using v1.23.0-alpha.1 for the time being) 
```sh
docker tag k8s.gcr.io/kube-api-server-amd64:v1.23.0-alpha.1.33_231427f6c9dca9 k8s.gcr.io/kube-api-server:v1.23.0-alpha.1
docker tag k8s.gcr.io/kube-controller-manager-amd64:v1.23.0-alpha.1.33_231427f6c9dca9 k8s.gcr.io/kube-controller-manager:v1.23.0-alpha.1
docker tag k8s.gcr.io/kube-proxy-amd64:v1.23.0-alpha.1.33_231427f6c9dca9 k8s.gcr.io/kube-proxy:v1.23.0-alpha.1
docker tag k8s.gcr.io/kube-scheduler-amd64:v1.23.0-alpha.1.33_231427f6c9dca9 k8s.gcr.io/kube-scheduler:v1.23.0-alpha.1
```

## Run the In-place pod vertical scaling PR in a K8S cluster

#### Setup the k8s cluster 

```sh
intel_http_proxy='http://child-prc.intel.com:913'
proxy_flag='on'
k8s_version='1.23.0-alpha.1'
function set_proxy_for_docker() {
  sudo mkdir /etc/systemd/system/docker.service.d
  cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=$intel_http_proxy"
Environment="HTTPS_PROXY=$intel_http_proxy"
Environment="NO_PROXY=localhost,127.0.0.0/8,10.293.154.0/16"
EOF
  sudo systemctl daemon-reload
  sudo systemctl restart docker
  echo ">>>>>>>>>>>>>>>>>>>>>>> Set Proxy for Docker <<<<<<<<<<<<<<<<<<<<<<<<<<"
}

#-------------------------------------------------------------------------------
#                          Pre Configure of K8s
#-------------------------------------------------------------------------------
#----------------- Let iptables see bridged traffic ----------------------------
function iptables_conf() {
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

  sudo sysctl --system
  sudo modprobe br_netfilter
  # if the after command has response mean availavle
  lsmod | grep br_netfilter
}
function kube_install() {
  sudo apt-get update
  sudo apt-get install -y apt-transport-https ca-certificates curl \
    net-tools ipvsadm

  if [ $proxy_flag == 'on' ]; then
    sudo curl -x $intel_http_proxy \
      -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg \
      https://packages.cloud.google.com/apt/doc/apt-key.gpg

  else
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg \
      https://packages.cloud.google.com/apt/doc/apt-key.gpg
  fi

  echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
  https://apt.kubernetes.io/ kubernetes-xenial main" |
    sudo tee /etc/apt/sources.list.d/kubernetes.list

  sudo apt-get update
  sudo apt-get install -y \
    kubelet=${k8s_version}-00 kubeadm=${k8s_version}-00 kubectl=${k8s_version}-00
  sudo apt-mark hold kubelet kubeadm kubectl
}
iptables_conf
load_local_images
kube_install
```

#### Deploy the PR binary and check the version

```sh
cd $GOPATH/src/k8s.io/kubernetes/_output/bin/
sudo cp kubelet /usr/bin/kubelet
sudo cp kubeadm /usr/bin/kubeadm
sudo cp kubectl /usr/bin/kubectl
kubelet --version
kubeadm version
kubectl version
```

#### Init the kubernetes cluster

```sh
function k8s_init() {
  ifconfig | awk '/inet/{print $2}' | cut -f2 -d ":"
  ipaddr=$(ifconfig | awk '/inet/{print $2}' | cut -f2 -d ":" |
    awk 'NR==2 {print $1}')

  echo "Using Server IP:" $ipaddr
  read -p "K8s default IP is[$ipaddr]:" newIP
  if [ $newIP ]; then
    echo "specfic an IP "
    ipaddr=$newIP
  fi
  echo "---------------K8s API Ip is $ipaddr------------------------"
  echo "---------------K8s API Ip is $ipaddr------------------------"
  echo "---------------K8s API Ip is $ipaddr------------------------"

  cat << EOF | sudo tee kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: ${ipaddr}
  bindPort: 6443
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.23.0-alpha.1
networking:
  podSubnet: "10.244.0.0/16"
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF

  sudo kubeadm init -v=5 --config=kubeadm-config.yaml

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  # let master also be a work node
  kubectl taint nodes --all node-role.kubernetes.io/master-
  # nodename=$(kubectl get node -o jsonpath='{.items[0].metadata.name}')
  # kubectl taint node $nodename node-role.kubernetes.io/master:NoSchedule-
  # kubectl label --overwrite node $nodename ovn4nfv-k8s-plugin=ovn-control-plane
  # kubectl create namespace sdewan-system

  # kubectl autocomplete
  cat <<EOF | sudo tee ~/.alias
source <(kubectl completion bash)
alias k="kubectl"
complete -o default -F __start_kubectl k
EOF
  echo 'source ~/.alias' >>~/.bashrc

  # helm install
  # sudo snap install helm --classic
}
k8s_init
```

## Enable Feature gates for InPlacePodVerticalScaling

#### Add the additional argument to api-server by adding the line below to /etc/kubernetes/manifests/kube-apiserver.yaml. Api server will detect the change and restart itself automatically.
```sh
--feature-gates=InPlacePodVerticalScaling=true
```

#### Add the feature gates parameter to kubelet by adding it to /var/lib/kubelet/kubeadm-flags.env. The config file will be like below. Then restart kubelet. 
```sh
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.5 --feature-gates=InPlacePodVerticalScaling=true"
```

## Reference


1. 国内环境下 Kubernetes 源码编译及运行: https://cloud.tencent.com/developer/article/1433219
2. Development Guide: https://github.com/kubernetes/community/blob/master/contributors/devel/development.md

