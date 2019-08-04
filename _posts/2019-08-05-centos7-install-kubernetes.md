---
layout:     post
title:      "CentOS7使用kubeadm部署kubernetes非高可用环境"
subtitle:   "CentOS7 install kubernetes"
date:       2019-7-16 17:39:00
author:     "Leon"
header-img: "img/life-bg.jpg"
tags:
    - kubernetes
---

> 

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。
kubeadm可帮助你快速部署一套kubernetes集群。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# kubeadm init                        # 创建Master节点
# kubeadm join <Master节点的IP和端口 >  # 将Node节点加入到集群，指令为kubeadm init的结果输出
```


## 所有节点安装docker

```
# wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo
# yum repolist
# yum -y install docker-ce    # yum -y install docker-ce-18.06.1.ce-3.el7.x86_64
# docker --version
# systemctl enable docker && systemctl status docker
# systemctl start docker && systemctl status docker
```
**设置国内镜像源**
```
# sed -s 's/^ExecStart=.*$/& --registry-mirror=https:\/\/docker.mirrors.ustc.edu.cn/' -i /usr/lib/systemd/system/docker.service
# systemctl daemon-reload
# systemctl enable docker
# systemctl restart docker && systemctl status docker
```

## 所有节点关闭防火墙、关闭selinux、关闭swap、设置iptables对bridge的数据处理、修改主机名和修改hosts文件

**关闭防火墙：**
```
# systemctl stop firewalld
# systemctl disable firewalld
```
**关闭selinux：**
```
# sed -i 's/enforcing/disabled/' /etc/selinux/config 
# setenforce 0
```
**关闭swap：**
```
# swapoff -a      # 临时
# vim /etc/fstab  # 永久
```

**设置iptables对bridge的数据处理**
```
# cat <<EOF > /etc/sysctl.d/k8s.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# sysctl --system
```

**修改主机名**

| 节点       | 主机名 |
| :-------- | -----:|
| master节点 | master|
| node节点1  | node1 |
| node节点2  | node2 |

```
# hostnamectl set-hostname [主机名信息master|node1|node2]
```

**添加主机名与IP对应关系**
```
# cat /etc/hosts
192.168.247.211 master
192.168.247.212 node1
192.168.247.213 node2
```


## 所有节点安装kubeadm和kubelet
link: https://kubernetes.io/docs/setup/independent/install-kubeadm/

kubeadm的版本与kubernetes版本一致
获取版本信息（当前版本为1.15.1）

```
# yum info kubeadm
```

```
# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
# yum install -y kubelet-1.15.1 kubeadm-1.15.1
# systemctl enable kubelet && systemctl start kubelet
```

## master节点初始化集群

1. 安装kubectl
```
# yum install -y kubectl-1.15.1
```

2. 初始化集群
```
# kubeadm init \
  --apiserver-advertise-address=192.168.247.211 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.15.1 \
  --service-cidr=10.1.0.0/16\
  --pod-network-cidr=10.244.0.0/16
```
**以下操作为kubeadm init的结果，仅参考**
```
# mkdir -p $HOME/.kube
# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# chown $(id -u):$(id -g) $HOME/.kube/config
```

**注意**：--apiserver-advertise-address为master主机IP, 指定阿里云镜像仓库地址

3. 安装网络组件（CNI）
```
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

4. 查看
```
# kubectl get pods -n kube-system
# kubectl get nodes
```

## node节点上加入集群

**注意：**执行在kubeadm init输出的kubeadm join命令，类似<<kubeadm join 192.168.247.211:6443 --token .......>>

## 查看集群信息
```
# kubectl get pods -n kube-system
# kubectl get nodes
```

## 部署Dashboard

1. 下载kubernetes-dashboard.yaml文件
 -- 访问 https://github.com/kubernetes/dashboard
  查看Getting Started  2019-6-22 版本为v1.10.1
  即：https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

2. 修改yaml文件
  * 修改国内镜像源【Deployment】
    image: registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1
  * 默认Dashboard只能集群内部访问，修改Service为NodePort类型，暴露到外部【Service】
    type: NodePort
    nodePort: 30001

3. 部署dashboard
```
# kubectl apply -f kubernetes-dashboard.yaml
# # kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml  # 镜像不能正常pull
```

4. 访问dashboard

 * 生成token信息，创建service account并绑定默认cluster-admin管理员集群角色：

```
$ kubectl create serviceaccount dashboard-admin -n kube-system
$ kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
$ kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

* 登录，使用输出的token登录Dashboard。
**访问地址：**http://NodeIP:30001    若有不能访问情况，尝试使用火狐浏览器



