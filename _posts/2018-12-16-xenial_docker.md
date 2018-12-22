---
layout:     post
title:      "Ubuntu xenial安装docker"
subtitle:   "Ubuntu16.04 install docker"
date:       2018-12-16 17:39:00
author:     "Leon"
header-img: "img/life-bg.jpg"
tags:
    - docker
---


## 安装ubuntu16.04

> 配置静态地址

```
# vi /etc/network/interfaces
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18 
iface ens18 inet static 
address 192.168.247.188
netmask 255.255.255.0
gateway 192.168.247.1
dns-nameservers 192.168.247.1
```

> 配置ubuntu清华大学源

```
# vi /etc/apt/sources.list
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial universe
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates universe
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu xenial-security main restricted
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu xenial-security universe
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu xenial-security multiverse

# apt-get update
```


## 安装docker

> 添加清华大学docker源

```
# cat <<EOF > /etc/apt/sources.list.d/docker-ce.list
deb [arch=amd64] $mirrors/docker-ce/linux/ubuntu xenial stable
EOF
# apt-get update
# apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/gpg | apt-key add -
```

> 安装docker-ce

```
# apt-cache madison docker-ce
# apt-get -y install docker-ce=18.06.1~ce~3-0~ubuntu
```


## docker服务添加国内源

> 查看docker.service文件路径

```
# systemctl status docker
```

> 修改启动参数

```
# vi /lib/systemd/system/docker.service
...
ExecStart=/usr/bin/dockerd -H fd:// --registry-mirror=https://docker.mirrors.ustc.edu.cn
...
```

> 启动docker服务

```
# systemctl daemon-reload && systemctl restart docker
```


## github
https://github.com/leonlm/opm-tools/blob/master/install/docker.sh



## 参考
官网：https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1




