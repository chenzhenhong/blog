---
title: Kubernetes集群搭建
date: 2024-03-29 22:01:16
category:
- 实践
tags: 
- Kubernetes
- 容器
---

> [安装kubeadm](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

> [阿里云镜像站](https://developer.aliyun.com/mirror/?spm=a2c6h.13651102.0.0.2ebd1b11PuNEF0&serviceType=mirror)

> [containerd getting-started](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

### 1. 开启IPv4数据包转发
```shell
# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system

# 用于确认
sudo sysctl net.ipv4.ip_forward
```

### 2.确保集群节点MAC和product_id唯一
```shell
# 查看product_id
sudo cat /sys/class/dmi/id/product_uuid
```

### 3.开发端口
> [端口与协议 | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/networking/ports-and-protocols/)

### 4.关闭swap、关闭SELinux
```shell
sudo swapoff -a # 临时关闭
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab # 永久关闭


sudo setenforce 0 # 临时关闭
sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config # 永久关闭
```
 
### 5.安装容器运行时-containerd
```shell
repo_file="/etc/yum.repos.d/docker-ce.repo"
if [ -f ${repo_file} ]; then
    sudo cp ${repo_file} ${repo_file}.$(date +%Y%m%d%H%M%S).bak
fi

# 使用阿里云提供的镜像源
sudo curl -fsSL -o ${repo_file} https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

sudo dnf install -y containerd
sudo systemctl enable --now containerd
```

### 6.修改containerd的cgroup驱动
当系统使用systemd时，推荐使用systemd cgroup驱动；修改kubelet和containerd使用一样的驱动，但是kubelet现在默认是systemd cgroup。
```shell
# 生成配置文件，dnf安装后自动生成的配置文件是简化版的
containerd config default > /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
```

### 7.安装kubeadm、kubelet、kubectl
```shell
# 使用阿里云提供的镜像源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.33/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet && systemctl start kubelet
```

### 8.在master节点初始化集群 - 请先查看第10点
> [kubeadm init参数](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-init/)
```shell
# 设置配置文件
cat <<EOF | sudo tee /etc/kubernetes-init.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
imageRepository: "registry.cn-hangzhou.aliyuncs.com/google_containers"
clusterName: "example-cluster"
EOF

kubeadm --config /etc/kubernetes-init.yaml init
```

### 9.使用kubectl
运行完kubeadm init后会在/ect/kubernetes中生成admin.conf和super-admin.conf俩个配置文件（内包含证书）
```shell
# 非root用户
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# root用户
export KUBECONFIG=/etc/kubernetes/admin.conf

# 使用kubectl查看节点
kubectl get nodes

# 查看当前核心组件pod, 在命令空间kube-system
kubectl get pods -n kube-system

# 查看集群信息
kubectl cluster-info
```

### 10.安装CNI插件Calico
> [Calico本地安装](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
```shell
# 还没有安装CNI插件时会看到coreDNS pod未处于ready状态
# Calico有俩种安装方式Operator和Manifest,这里使用Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/custom-resources.yaml -O
kubectl create -f custom-resources.yaml

# 验证
kubectl get pods -n calico-system
```

```shell
# 安装Calico需要在kubeadm init时设置参数pod网络段
# 这里的192.168.0.0/16从上一步Calico的custom-resources.ymal文件中获取
cat <<EOF | sudo tee /etc/kubernetes-init.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
imageRepository: "registry.cn-hangzhou.aliyuncs.com/google_containers"
clusterName: "example-cluster"
networking:
  podSubnet: "192.168.0.0/16"
EOF

# 重新创建节点
kubeadm reset -f 
kubeadm --config /etc/kubernetes-init.yaml init
```

### 10.允许控制平面调度pod
```shell
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 11.添加工作节点
```shell
# 如果忘记了join命令可以通过以下命令获取--默认24h有效
kubeadm token create --print-join-command

# 然后在工作节点运行join命令就像
kubeadm join 192.168.122.2:6443 --token f0xopg.yj3os9n757dx8ym6 --discovery-token-ca-cert-hash sha256:3fe08a85f4031bff54fb366feda87d52ae1b6872ac8188453ff1733f507c2ddb
```

### 12.设置工作节点的kubectl
```shell
scp root@192.168.122.2:/etc/kubernetes/admin.conf /etc/kubernetes/admin.conf

# 非root用户
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# root用户
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 13.加入工作节点后重新平衡coreDNS pod
```shell
kubectl -n kube-system rollout restart deployment coredns
```


-- -----------------------------------
附录：
### 安装脚本 - 不包含init join
```shell
#/bin/env bash
set -Euo
set -o pipefail

#---------------------------------------------
# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
# 应用 sysctl 参数而不重新启动
sudo sysctl --system
#---------------------------------------------

#---------------------------------------------
sudo swapoff -a # 临时关闭
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab # 永久关闭
sudo setenforce 0 # 临时关闭
sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config # 永久关闭
#---------------------------------------------

#---------------------------------------------
repo_file="/etc/yum.repos.d/docker-ce.repo"
if [ -f ${repo_file} ]; then
    sudo cp ${repo_file} ${repo_file}.$(date +%Y%m%d%H%M%S).bak
fi

sudo curl -fsSL -o ${repo_file} https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo dnf install -y containerd
sudo systemctl enable --now containerd

containerd config default > /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

grep -q '\[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"\]' /etc/containerd/config.toml || \
sed -i '/\[plugins."io.containerd.grpc.v1.cri".registry.mirrors\]/a\
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]\n  endpoint = ["https://registry.aliyuncs.com"]
' /etc/containerd/config.toml

sudo systemctl restart containerd
#---------------------------------------------

#---------------------------------------------
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.33/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet && systemctl start kubelet
#---------------------------------------------
```

### 主节点init脚本
```shell
#---------------------------------------------
cat <<EOF | sudo tee /etc/kubernetes-init.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
imageRepository: "registry.cn-hangzhou.aliyuncs.com/google_containers"
clusterName: "example-cluster"
networking:
  podSubnet: "192.168.0.0/16"
EOF

kubeadm --config /etc/kubernetes-init.yaml init

# 使用source运行，或者写入~/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
#---------------------------------------------

#---------------------------------------------
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/custom-resources.yaml -O
kubectl create -f custom-resources.yaml
#---------------------------------------------

#---------------------------------------------
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
#---------------------------------------------
```

### 工作节点join脚本
```shell
kubeadm join 192.168.122.2:6443 --token f0xopg.yj3os9n757dx8ym6 --discovery-token-ca-cert-hash sha256:3fe08a85f4031bff54fb366feda87d52ae1b6872ac8188453ff1733f507c2ddb

scp root@192.168.122.2:/etc/kubernetes/admin.conf /etc/kubernetes/admin.conf

# 使用source运行，或者写入~/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### containerd设置镜像源
注意：kubeadm配置的镜像源只用于拉取kubernetes组建，如kube-apiserver、kube-controller-manager、kube-scheduler、kube-proxy 等。普通pod如果通过containerd运行，需要配置containerd的。
```shell
grep -q '\[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"\]' /etc/containerd/config.toml || \
sed -i '/\[plugins."io.containerd.grpc.v1.cri".registry.mirrors\]/a\
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]\n  endpoint = ["https://registry.aliyuncs.com"]
' /etc/containerd/config.toml

sudo systemctl restart containerd
```


### kubeadm init输出
```shell
[root@rocky9-vm-1 ~]# kubeadm --config /etc/kubernetes-init.yaml init
[init] Using Kubernetes version: v1.33.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
W0618 14:14:27.507966   15503 checks.go:846] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.10" as the CRI sandbox image.
^C
[root@rocky9-vm-1 ~]# kubeadm --config /etc/kubernetes-init.yaml init
[init] Using Kubernetes version: v1.33.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
W0618 14:14:47.236950   15536 checks.go:846] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.10" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local rocky9-vm-1] and IPs [10.96.0.1 192.168.122.2]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost rocky9-vm-1] and IPs [192.168.122.2 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost rocky9-vm-1] and IPs [192.168.122.2 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 502.278478ms
[control-plane-check] Waiting for healthy control plane components. This can take up to 4m0s
[control-plane-check] Checking kube-apiserver at https://192.168.122.2:6443/livez
[control-plane-check] Checking kube-controller-manager at https://127.0.0.1:10257/healthz
[control-plane-check] Checking kube-scheduler at https://127.0.0.1:10259/livez
[control-plane-check] kube-controller-manager is healthy after 10.002751337s
[control-plane-check] kube-scheduler is healthy after 10.766272408s
[control-plane-check] kube-apiserver is healthy after 12.503088837s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node rocky9-vm-1 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node rocky9-vm-1 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: j4lxv0.xp6zdr0vp80jx673
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.122.2:6443 --token j4lxv0.xp6zdr0vp80jx673 \
	--discovery-token-ca-cert-hash sha256:3b445f11df5beb1f6a76ec6bdf1bf0777189a66aecaac2bc66df73bfc3910ffa 
```