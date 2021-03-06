---
layout:     post
title:      "自建yum源（rpm包仓库）"
date:       2018-05-26 18:22:00
author:     "Leon"
header-img: "img/life-bg.jpg"
tags:
    - yum
---


> 本文介绍如何自建对外提供服务的yum源以及客户端yum源的使用

---

## 操作步骤说明

步骤如下：
1. 创建rpm包归档目录并拷贝相关rpm至该目录

2. 使用createrepo命令建立rpm包归档目录的索引信息，描述各包所需依赖的信息，生成元数据信息

3. 通过nginx提供http服务

4. 客户端配置源地址

---


## 执行

***服务器端***

1. 创建rpm包归档目录

    ```
# mkdir /rpms
    ```
   

2. 建立索引

    ```
# yum -y install createrepo
# createrepo --version
# createrepo -v /rpms
# ls /rpms/repodata             # 可见生成repodate
# createrepo --update -v /rpms  # 若归档目录添加或删除了rpm包，可以update更新索引，不必重新生成，节省时间
    ```

    repodata目录包含多个文件，rpm包相关的索引信息：
    * *-filelists.xml.gz
    * *-primary.sqlite.bz2
    * *-primary.xml.gz
    * *-filelists.sqlite.bz2
    * *-other.sqlite.bz2
    * *-other.xml.gz
    * repomd.xml
    
    其中最主要的是repomd.xml文件。
    
    一般出现的找不到源文件的错误的原因通常有三个：
    
    1. 路径问题
    2. 没有生成repodate目录
    3. *.repo配置文件冲突（这个需要特意注意）

    
3. nginx的安装及配置

```
# cat <<EOF > /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/6/x86_64
gpgcheck=0
enabled=1
EOF

# yum install nginx -y

# cat <<EOF > /etc/nginx/conf.d/rpms.conf   # 添加nginx配置
server {
    listen          80;
    server_name     192.168.247.165;
    root            /rpms;
    location / {
        autoindex on;
    }
}
EOF

# /etc/init.d/nginx restart
# netstat -lpnt | grep ":80"
```

---


***客户端***

1. 配置使用yum源

    * 在本地直接使用（在yum源的服务器上直接使用）

    ```
    # cat <<EOF > /etc/yum.repos.d/local-rpms.repo
    [local-rpms-repo]
    name=Local Rpms repo for RHEL/CentOS $releasever
    baseurl=file:///rpms
    enabled=1
    gpgcheck=0
    priority=1
    EOF

    # yum clean all     # 清空本地缓存
    # yum repolist
    # yum makecache     # 重建缓存
    ```

    * 通过http使用yum源
    
    ```
    # cat <<EOF > /etc/yum.repos.d/rpms.repo
    [rpms-repo]
    name=Rpms repo for RHEL/CentOS $releasever
    baseurl=http://$service_ip/rpms
    enabled=1
    gpgcheck=0
    priority=1
    EOF

    # yum clean all     # 清空本地缓存
    # yum repolist
    # yum makecache     # 重建缓存
    ```

2. 安装应用

    ```
    yum install $software
    ```

---


## createrepo

> yum(Yellow dog Updater,Modified)主要的功能是方便添加、删除和更新rpm软件包。可以解决软件包依存问题，更便于管理大量的系统更新问题。它可以同时配置多个仓库或叫资源库(repository)，就是存放更新和依存的软件包的地方。

> createrepo命令用于创建yum源（软件仓库），即为存放于本地的rpm包目录建立索引，描述各包所需依赖的信息，生成元数据信息。This utility will generate a common metadata repository from a directory of rpm packages



**语法：**
```
createrepo [option] <directory>
```

**参数选项说明**

```
-u  --baseurl <url>
    指定Base URL的地址

-o --outputdir <url>
    指定元数据的输出位置

-x --excludes <packages>
    指定在形成元数据时需要排除的包

-i --pkglist <filename>
    指定一个文件，该文件内的包信息将被包含在即将生成的元数据中，格式为每个包信息独占一行，不含通配符、正则，以及范围表达式。

-n --includepkg
    通过命令行指定要纳入本地库中的包信息，需要提供URL或本地路径。

-q --quiet
    安静模式执行操作，不输出任何信息。

-g --groupfile <groupfile>
    指定本地软件仓库的组划分，范例如下：
createrepo -g comps.xml /path/to/rpms
    注意：组文件需要和rpm包放置于同一路径下。

-v --verbose
    输出详细信息。

-c --cachedir <path>
    指定一个目录，用作存放软件仓库中软件包的校验和信息。
    当createrepo在未发生明显改变的相同仓库文件上持续多次运行时，指定cachedir会明显提高其性能。

--update
    如果元数据已经存在，且软件仓库中只有部分软件发生了改变或增减，
    则可用update参数直接对原有元数据进行升级，效率比重新分析rpm包依赖并生成新的元数据要高很多。

-p --pretty
    以整洁的格式输出xml文件。

-d --database
    该选项指定使用SQLite来存储生成的元数据，默认项。

```



# 打包rpm
使用rpmbuild打包软件，如下以golang-1.8.1为例

//安装rpm相关包开发开具
#yum install rpm* rpm-devel rpmdevtools
//下载golang-1.8.1
#cd ~
#wget https://storage.googleapis.com/golang/go1.8.1.linux-amd64.tar.gz
//编写.spec文件
#rpmdev-newspec -o golang.spec
#vim golang.spec
#cp golang.spec rpmbuild/SPECS/
//创建rpm包项目结构
#rpmdev-setuptree
#cd ~/rpmbuild
#copy go1.8.1.linux-amd64.tar.gz rpmbuild/SOURCE/
//生成rpm包
#cd SPECS
#rpmbuild -bb golang.spec
//复制生成.rpm包,到自已yum服务器目录
#cp RPMS/x86_64/golang-1.8.1-1.el6.x86_64.rpm  /yum_repo/centos/6/x86_64

部署成生成的yum包
//生成rpm包
#cd SPECS
#rpmbuild -bb golang.spec
//复制生成.rpm包,到自已yum服务器目录
#cp RPMS/x86_64/golang-1.8.1-1.el6.x86_64.rpm  /yum_repo/centos/6/x86_64
//更新yum服务器索引 
#createrepo --update -v /yum_repo/centos/6/x86_64
//yum客户端 重新yum makecache 即可

下载其它源的rpm包,加到自已的yum源服务器,以nginx为例
//安装yum downloadonly插件
#yum -y install yum-downloadonly
#yum -y install --downloadonly --downloaddir=/yum_repo/centos/6/x86_64 nginx
//更新服务器索引 
#createrepo --update -v /yum_repo/centos/6/x86_64
//下载时注意，如果已经安装过要下载的rpm包，请先行卸载：
#yum remove nginx

vim golang.spec 如下：
Name:golang
Version:1.8.1
Release:1%{?dist}
Summary:golangBinnary

#Group: system
License:GPL
Distribution:Red Hat Linux
#URL:http://golang.org
#Source0:go1.8.1.linux-amd64.tar.gz
Requires:glibc
Autoreq:0

%define userpath /usr/local

%description
golang 1.8.1

#%prep
#tar -xzvf ${RPM_SOURCE_DIR}/go1.8.1.linux-amd64.tar.gz

%install
install -d $RPM_BUILD_ROOT%{userpath}
tar -C $RPM_BUILD_ROOT%{userpath} -xzf ${RPM_SOURCE_DIR}/go1.8.1.linux-amd64.tar.gz
#sudo tar -C /usr/local -xzf ${RPM_SOURCE_DIR}/go1.8.1.linux-amd64.tar.gz

#sudo cp -r ${RPM_SOURCE_DIR}/go /usr/local/
#export PATH=$PATH:/usr/local/go/bin


%clean
rm -fr $RPM_BUILD_ROOT/*
rm -fr $RPM_BUILD_DIR/*


%files
%defattr(-,root,root,-)
%doc
%{userpath}/go



%changelog

--------------------- 

