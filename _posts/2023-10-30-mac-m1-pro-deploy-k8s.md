---
layout: post
title:  "Mac M1 Pro上搭建k8s集群"
date: 2023-10-30 21:59:24 +0800
categories: [K8S]
tags: [k8s, flannel, m1, arm64]     # TAG names should always be lowercase1
---

## 简介

学习k8s想自己搭一套集群，首先考虑租云主机，算下来vultr或者阿里云按需付费，租三台一个月下来也得600元大洋，有点肉疼，而且最终销毁掉再想玩还得重新搭建太麻烦。所以还是决定用自己的M1 pro来搞，Apple Silicon芯片会有些小坑，已经有一定的心理准备了，下面记录搭建过程。

### 环境要求

1. M1 pro 10C16G 其他Apple Silicon芯片的电脑都可以
2. 科学上网（后面有介绍）
3. 100G上磁盘空间

### 搭建目标

1. 1台master节点，2台worker节点
2. proxy mode使用ipvs
3. 用flannel组件搭建网络层
4. cgroup驱动使用systemd

## 准备工作

### 安装虚拟机

1. 清理足够空间，预留100G以上磁盘空间
2. 虚拟机
   1. 支持silicon chip的虚拟机只有UTM（开源免费）和vmware fusion player（个人license免费） 
   2. 我这里使用的UTM [https://mac.getutm.app/](https://mac.getutm.app/)

### 安装RockyLinux 9

centos7、8不支持apple silicon芯片，简单说是apple silicon不支持64k分页，详见[vmware社区帖子](https://communities.vmware.com/t5/Fusion-22H2-Tech-Preview/Unable-to-install-CentOS-7-on-VMWare-fusion-for-Macbook-M1/td-p/2908072)。所以这里我使用的Rocky Linux 9（centos不维护后的社区替代版本，目前社区比较活跃），5.4版本的linux kernel。下载Rocky 9，要选择ARM64版本 minimal就可以。[下载地址](https://rockylinux.org/download)

UTM安装RockyLinux过程不赘述，可以参考视频 [https://www.youtube.com/watch?v=NTOcxlHm_u8](https://www.youtube.com/watch?v=NTOcxlHm_u8)。

我这里启动了三台虚拟机，master01、node01、node02，配置都是4c4g 100G硬盘。

![image](/assets/img/2023-10-30-mac-m1-pro-deploy-k8s/utm.png){: width="500" }

**注意：**

1. network选共享模式
2. 启用root账户，并允许ssh（后续操作方便一些）

### 科学上网

虽然有一些下载环节可以替换国内镜像源，但是作为程序员标配，还是方便很多。我后面的操作也依赖科学上网。没有梯子的同学推荐[这款](https://portal.shadowsocks.au/aff.php?aff=4434)，用了7、8年了还挺稳。

**注意：**

1. 在虚拟机内，查看master01、node01、node02的ip，记录下来。
2. 从宿主机ssh到虚拟机，在虚拟机内 `who` 看下宿主机的ip，记录下来。
3. 把梯子软件的“允许局域网连接”的选项开启

## 安装k8s

### 修改主机名

三台虚拟机分别修改为

- `hostnamectl set-hostname k8s-master01`
- `hostnamectl set-hostname k8s-node01`
- `hostnamectl set-hostname k8s-node02`

### 修改/etc/hosts

在三台机器上绑一下host，在/etc/hosts追加。这里注意，我写的ip是我本地情况，你应该根据实际情况修改。

```bash
192.168.64.2 k8s-master01
192.168.64.3 k8s-node01
192.168.64.4 k8s-node02
```

### 安装依赖包

```bash
yum install -y conntrack ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git
```

### 关闭防火墙

```bash
systemctl stop firewalld && systemctl disable firewalld
```

### 清空iptables规则

```bash
yum -y install iptables-services && systemctl start iptables && systemctl enable iptables && iptables -F && service iptables save
```

### 关闭swap

```bash
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 关闭SELinux

```bash
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 内核参数优化

```bash
cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
## net.ipv4.tcp_tw_recycle=0
net.ipv4.tcp_tw_reuse=1
vm.swappiness=0 ## 禁止使用 swap 空间，只有当系统 00M 时才允许使用它
vm.overcommit_memory=1 ## 不检查物理内存是否够用
vm.panic_on_oom=0 #开启 00M
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
```

net.ipv4.tcp_tw_recycle=0 从4.12内核开始已经被移除了，所以我这里注释掉了详见[https://djangocas.dev/blog/troubleshooting-tcp_tw_recycle-no-such-file-or-directory/](https://djangocas.dev/blog/troubleshooting-tcp_tw_recycle-no-such-file-or-directory/)

**拷贝配置**

```bash
cp kubernetes.conf /etc/sysctl.d/kubernetes.conf
```

**加载两个网络相关的模块**

```bash
modprobe br_netfilter
modprobe ip_conntrack
```

**生效配置**

```bash
sysctl -p /etc/sysctl.d/kubernetes.conf
```

### 配置IPVS

```bash
mkdir -p /etc/sysconfig/modules/

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
```

**验证**

```bash
lsmod|grep -e ip_vs -e nf_conntrack_ipv4
```

### 配置dns

```bash
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
```

### 安装docker

```bash
## 导入阿里云的镜像仓库
sudo dnf config-manager --add-repo=https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

## 安装docker
sudo dnf install -y docker-ce docker-ce-cli docker-compose-plugin

## 启动docker engine
systemctl start docker

## 设置开机自启
systemctl enable docker
```

**修改docker配置**

```bash
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver":"json-file",
  "log-opts": {
    "max-size":"100m"
  }
}
EOF
```

**docker科学上网**

参考 [https://note.qidong.name/2020/05/docker-proxy/](https://note.qidong.name/2020/05/docker-proxy/)

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d

## 根据自己的梯子实际情况配置下面的内容，192.168.64.1是我宿主机的ip，
cat > /etc/systemd/system/docker.service.d/proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://192.168.64.1:7890/"
Environment="HTTPS_PROXY=http://192.168.64.1:7890/"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
EOF
```

重启docker

```bash
systemctl daemon-reload
systemctl restart docker
```

**重启docker服务**

```bash
systemctl daemon-reload && systemctl restart docker
```

### 安装containerd

```bash
sudo dnf install -y containerd.io
```

生成containerd配置

```bash
containerd config default > /etc/containerd/config.toml
```

修改containerd配置

```bash
vim /etc/containerd/config.toml

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"  ## 这个应该默认就是，可以检查一下
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true  ## 这里要改成true，我们使用cgroup驱动是systemd
```

给containerd配置科学上网

```bash
mkdir -p /etc/systemd/system/containerd.service.d/

cat > /etc/systemd/system/containerd.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://192.168.64.1:7890"
Environment="HTTPS_PROXY=http://192.168.64.1:7890"
EOF
```

重启containerd服务

```bash
systemctl daemon-reload && systemctl restart containerd
```

### 安装kubelet kubeadm kubectl

**Add repo**

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-aarch64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

**Install**

```bash
sudo yum install -y kubelet kubeadm kubectl
```

**kubelet开机启动**

```bash
systemctl enable kubelet.service
```

### 初始化master01

以下操作在master01进行

```bash
kubeadm config print init-defaults > kubeadm-config.yaml
```

**修改配置**

注意：注释部分需要在修改时删掉

```bash
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.64.2    ## 这里改成当前机器的ip
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock ## 这里修改containerd.sock路径
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.28.3 ## 版本号
networking:
  dnsDomain: cluster.local
  podSubnet: "10.244.0.0/16"  ## Flannel默认pod网段
  serviceSubnet: 10.96.0.0/12
scheduler: {}
--- #下面是新增内容，配置ipvs
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```

初始化

```bash
kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.log
```

### 拷贝kubectl配置

```bash
mkdir -p $HOME/.kube/
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

## check
kubectl get node
kubectl get pod -n kube-system
```

### 部署网络

下载flannel资源清单文件，并部署

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl create -f kube-flannel.yml
```

查看下flannel的状态

```bash
kubectl get pod -n kube-flannel
```

### 加入worker node节点

在node01 和 node02上执行如下命令，这个命令是在kubeadm init成功后输出在控制台的

```bash
kubeadm join 192.168.64.2:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:28829c87136e5bdf1e22f4fcab6636c4d8c9172efe7548b092c31eb99a9e2d47
```

## 完成部署

到这里就完成了k8s集群在m1 pro上的搭建，看下最终的状态。

**pod状态**

```bash
[root@k8s-master01 ~]## kubectl get pod -A -o wide
NAMESPACE      NAME                                   READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
kube-flannel   kube-flannel-ds-hccq4                  1/1     Running   0          3h47m   192.168.64.3   k8s-node01     <none>           <none>
kube-flannel   kube-flannel-ds-lw2cf                  1/1     Running   0          3h46m   192.168.64.4   k8s-node02     <none>           <none>
kube-flannel   kube-flannel-ds-mnxs5                  1/1     Running   0          3h54m   192.168.64.2   k8s-master01   <none>           <none>
kube-system    coredns-5dd5756b68-7z4gg               1/1     Running   0          3h54m   10.244.0.2     k8s-master01   <none>           <none>
kube-system    coredns-5dd5756b68-ssb4z               1/1     Running   0          3h54m   10.244.0.3     k8s-master01   <none>           <none>
kube-system    etcd-k8s-master01                      1/1     Running   19         3h55m   192.168.64.2   k8s-master01   <none>           <none>
kube-system    kube-apiserver-k8s-master01            1/1     Running   19         3h55m   192.168.64.2   k8s-master01   <none>           <none>
kube-system    kube-controller-manager-k8s-master01   1/1     Running   27         3h55m   192.168.64.2   k8s-master01   <none>           <none>
kube-system    kube-proxy-9jjh5                       1/1     Running   0          3h47m   192.168.64.3   k8s-node01     <none>           <none>
kube-system    kube-proxy-dt6v5                       1/1     Running   0          3h46m   192.168.64.4   k8s-node02     <none>           <none>
kube-system    kube-proxy-qk7jb                       1/1     Running   0          3h54m   192.168.64.2   k8s-master01   <none>           <none>
kube-system    kube-scheduler-k8s-master01            1/1     Running   25         3h55m   192.168.64.2   k8s-master01   <none>           <none>
```

**node状态**

```bash
[root@k8s-master01 ~]## kubectl get node
NAME           STATUS   ROLES           AGE     VERSION
k8s-master01   Ready    control-plane   3h56m   v1.28.3
k8s-node01     Ready    <none>          3h48m   v1.28.2
k8s-node02     Ready    <none>          3h47m   v1.28.2
```
