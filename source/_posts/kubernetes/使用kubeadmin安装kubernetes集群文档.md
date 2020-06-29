---
title: 使用kubeadmin安装kubernetes集群文档
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## 使用kubeadmin安装kubernetes集群文档



### 集群环境

| 主机名  | IP地址      | 节点角色 | 操作系统  | Service网段   | pod网段      |
| ------- | ----------- | -------- | --------- | ------------- | ------------ |
| k8s-m01 | 10.111.2.58 | master   | CentOS7.6 | 10.100.0.0/20 | 10.96.0.0/12 |
| k8s-n01 | 10.111.2.56 | node     | CentOS7.6 | 10.100.0.0/20 | 10.96.0.0/12 |
| k8s-n02 | 10.111.2.55 | node     | CentOS7.6 | 10.100.0.0/20 | 10.96.0.0/12 |

---

### 服务器,网络要求

* service和Pod网段可以任意指定,但是不能和物理机网段,或者当前其他物理网段有冲突

* 服务器配置至少是2核2G

<!--more-->

### 软件版本

* kubernetes v1.15.3

* docker 18.09.7



### 安装文档参考:

大部分文档资料来源: https://www.kuboard.cn/install/install-k8s.html#introduction

但是我没有参考这个教程安装calico网络插件..而是安装flannal网络插件



### 安装过程



>  以下所有步骤都是用root账户执行



**一.在master节点执行以下安装步骤**



1.设置三台服务器的主机名:

```
#master节点:
hostnamectl --static set-hostname  k8s-m01

#node1节点
hostnamectl --static set-hostname  k8s-n01

#node2节点
hostnamectl --static set-hostname  k8s-02
```



2.添加host解析

```
#在3台服务器上执行

echo "10.111.2.58   k8s-m01
10.111.2.56  k8s-n01
10.111.2.55 k8s-n02
" >> /etc/hosts
```



3.安装以下软件:

* docker
* nfs-utils
* kubectl
* kubeadm
* kubelet

一键安装:

```
curl -sSL https://kuboard.cn/install-script/v1.15.3/install-kubelet.sh | sh
```



也可以采用手动安装的方式.(以下手动安装步骤实际上就是上面的Install-kubelet.sh文件内容.):

```
#!/bin/bash

# 在 master 节点和 worker 节点都要执行

# 安装 docker
# 参考文档如下
# https://docs.docker.com/install/linux/docker-ce/centos/ 
# https://docs.docker.com/install/linux/linux-postinstall/

# 卸载旧版本
yum remove -y docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine

# 设置 yum repository
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装并启动 docker
yum install -y docker-ce-18.09.7 docker-ce-cli-18.09.7 containerd.io
systemctl enable docker
systemctl start docker

# 安装 nfs-utils
# 必须先安装 nfs-utils 才能挂载 nfs 网络存储
yum install -y nfs-utils

# 关闭 防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SeLinux
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 关闭 swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab

# 修改 /etc/sysctl.conf
# 如果有配置，则修改
sed -i "s#^net.ipv4.ip_forward.*#net.ipv4.ip_forward=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-ip6tables.*#net.bridge.bridge-nf-call-ip6tables=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-iptables.*#net.bridge.bridge-nf-call-iptables=1#g"  /etc/sysctl.conf
# 可能没有，追加
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
# 执行命令以应用
sysctl -p

# 配置K8S的yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 卸载旧版本
yum remove -y kubelet kubeadm kubectl

# 安装kubelet、kubeadm、kubectl
yum install -y kubelet-1.15.3 kubeadm-1.15.3 kubectl-1.15.3

# 修改docker Cgroup Driver为systemd
# # 将/usr/lib/systemd/system/docker.service文件中的这一行 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# # 修改为 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
# 如果不修改，在添加 worker 节点时可能会碰到如下错误
# [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". 
# Please follow the guide at https://kubernetes.io/docs/setup/cri/
sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service

# 设置 docker 镜像，提高 docker 镜像下载速度和稳定性
# 如果您访问 https://hub.docker.io 速度非常稳定，亦可以跳过这个步骤
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io

# 重启 docker，并启动 kubelet
systemctl daemon-reload
systemctl restart docker
systemctl enable kubelet && systemctl start kubelet

docker version
```



4.初始化master节点

一键初始化 .执行以下命令:

```
# 只在 master 节点执行
# 替换 x.x.x.x 为 master 节点实际 IP（请使用内网 IP）
# export 命令只在当前 shell 会话中有效，开启新的 shell 窗口后，如果要继续安装过程，请重新执行此处的 export 命令

export MASTER_IP=x.x.x.x

# 替换 apiserver.demo 为 您想要的 dnsName (不建议使用 master 的 hostname 作为 APISERVER_NAME)

export APISERVER_NAME=apiserver.demo
export POD_SUBNET=10.100.0.1/20
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
curl -sSL https://kuboard.cn/install-script/v1.15.3/init-master.sh | sh
```

也可以手工方式初始化:

```
# 只在 master 节点执行
# 替换 x.x.x.x 为 master 节点实际 IP（请使用内网 IP）
# export 命令只在当前 shell 会话中有效，开启新的 shell 窗口后，如果要继续安装过程，请重新执行此处的 export 命令
export MASTER_IP=x.x.x.x
# 替换 apiserver.demo 为 您想要的 dnsName (不建议使用 master 的 hostname 作为 APISERVER_NAME)
export APISERVER_NAME=apiserver.demo
export POD_SUBNET=10.100.0.1/20
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts


# 只在 master 节点执行

# 查看完整配置选项 https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2

rm -f ./kubeadm-config.yaml
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.15.3
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "${APISERVER_NAME}:6443"
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "${POD_SUBNET}"
  dnsDomain: "cluster.local"
EOF

# kubeadm init
# 根据您服务器网速的情况，您需要等候 3 - 10 分钟
kubeadm init --config=kubeadm-config.yaml --upload-certs

# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config

# 安装 calico 网络插件
# 参考文档 https://docs.projectcalico.org/v3.8/getting-started/kubernetes/
rm -f calico.yaml
wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
sed -i "s#192\.168\.0\.0/16#${POD_SUBNET}#" calico.yaml
kubectl apply -f calico.yaml
```

> 如果不像安卓calico网络插件,而是安装flannal插件,可以将上个安装calico网络插件步骤替换成如下flannal部署方式:

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```



5.检查初始化结果



```
# 只在 master 节点执行

# 执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes
```



6.获得join命令参数:

```
# 只在 master 节点执行
kubeadm token create --print-join-command

执行这个命令会输出如下类似信息:

# kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
```

---



二.初始化node节点



在两台node节点分别执行:

```
# 只在 worker 节点执行
# 替换 ${MASTER_IP} 为 master 节点实际 IP
# 替换 ${APISERVER_NAME} 为初始化 master 节点时所使用的 APISERVER_NAME
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts

# 替换为 master 节点上 kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
```



四.检查初始化结果

在master节点上执行

```
# 只在 master 节点执行
kubectl get nodes

#输出结果如下:

[root@k8s-m01 ~]# kubectl get nodes
NAME      STATUS     ROLES    AGE     VERSION
k8s-m01   Ready      master   7m52s   v1.15.3
k8s-n01   Ready      <none>   50s     v1.15.3
k8s-n02   NotReady   <none>   5s      v1.15.3
```



至此,已经完成了kubeadm工具安装部署kubernetes集群的工作

----



## 移除node节点



**方式一.**

* 在node节点上执行

```
# 只在 worker 节点执行
kubeadm reset
```

**方式二.**

* 在master节点上执行

```
# 只在 master 节点执行
kubectl delete node k8s-n01
```



---



## 重置k8s集群配置

在master节点上执行

```
kubeadm reset
```



然后在node节点上执行reset,再重新添加回集群

```
kubeadm reset
```

