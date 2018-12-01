---
layout:     post
title:      "Docker inside LXC"
subtitle:   "ProxmoxVE容器LXC中docker部署"
date:       2017-09-03 11:19:00
author:     "Leon"
header-img: "img/life-bg.jpg"
tags:
    - docker
---


## pve中创建lxc

OS|CentOS7



## 修改lxc配置

> 配置lxc

```
# cat <<EOF > /etc/pve/lxc/[ctid].conf 
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop: 
EOF
```


## 安装docker

> 添加清华大学docker源

```
# cat <<EOF > /etc/yum.repos.d/docker-ce.repo
[docker-ce-stable-tuna]
name=Docker CE Stable - $basearch
baseurl=https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/7/x86_64/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/gpg
EOF

# yum clean all
```

> 安装docker-ce

```
# yum install docker-ce

# systemctl enable docker
```


## 添加国内源

> 查看docker.service文件路径

```
# systemctl status docker
```

> 修改启动参数

```
# vi /usr/lib/systemd/system/docker.servic
...
ExecStart=/usr/bin/dockerd --registry-mirror=https://docker.mirrors.ustc.edu.cn
...
```

> 启动docker服务

```
# systemctl daemon-reload && systemctl restart docker
```


