---
title: Docker 基础
date: 2016-12-04 03:58
tags:
    - 技术
    - Docker

---

# Docker 基础

## 安装 Docker
- 使用官方脚本自动安装标准版：`curl -sSL https://get.docker.com/ | sh`
- 使用官方脚本自动安装带实验模块：`curl -sSL https://experimental.docker.com/ | sh`
- 手工安装：[Install Docker on Ubuntu](https://docs.docker.com/engine/installation/linux/ubuntulinux/)
- **注意**：Ubuntu:14.04之前版本需要先升级内核

## 镜像
>Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

### 镜像加速
- 国内加速源
    + REPO = http://hub-mirror.c.163.com (公用地址)
    + REPO = https://ftwpufzc.mirror.aliyuncs.com (阿里云个人账户中的地址)
- 修改方法：使用 upstart 的系统 (Ubuntu 14.04、Debian 7 etc.)
```bash
$ vi /etc/default/docker
    DOCKER_OPTS="--registry-mirror=<REPO>"
$ service docker restart
```
- 修改方法：使用 systemd 的系统 (ubuntu 16.04, Debian 8, CentOS 7 etc.)
```bash
$ vi /etc/systemd/system/multi-user.target.wants/docker.service
    ExecStart=/usr/bin/dockerd --registry-mirror=<REPO>
$ systemctl daemon-reload
$ systemctl restart docker
```

### 镜像使用方法
- 获取镜像
```bash
    $ sudo docker pull debian:8
    $ sudo docker pull ubuntu:16.04
    $ sudo docker pull centos:centos7
    $ sudo docker pull ubuntu (ubuntu:latest)
```
- 更改 tag: 
```bash
    $ sudo docker tag centos:centos7 centos:7
```
- 运行：以一个镜像为基础来启动一个容器
```bash
    $ sudo docker run -t -i --rm ubuntu:12.04 /bin/bash
```
- 列出本地镜像
```bash
    $ sudo docker images
```
- 修改已有镜像并保存
```bash
    $ sudo docker run -t -i centos:7 /bin/bash
        # yum install nmap
        # exit
    $ sudo docker ps -aq | awk '{print $1}' # get container ID
    $ sudo docker commit -m "centos7-nmap" -a "David Wang" da65a584e1a9 zwang/centos7:v1
    $ sudo docker run --name="zwang-c7" -i -t zwang/centos7:v1 /bin/bash
```
- 利用 Dockerfile 来创建镜像
```bash
    $ vi Dockerfile    
    $ dk build -t 'zwang/u1404' . 
    $ dk run --name='zwang-u1404' -p 10022:22 -d  zwang/u1404:ssh
```
- 导出镜像
```bash
    $ sudo docker images
    $ sudo docker save -o ubuntu_14.04.tar ubuntu:14.04
```
- 导入镜像
```bash
    $ sudo docker load -i ubuntu_14.04.tar
    $ sudo docker images
    $ sudo docker tag <id|name> zwang/u1404:v1 #若无tag使用ID
```
- 移除镜像
*注意：在删除镜像之前要先用 docker rm 删掉依赖于这个镜像的所有容器。
```bash
    $ sudo docker rmi zwang/u1404:v1
```

## 容器
>简单的说，容器是独立运行的一个或一组应用，以及它们的运行态环境。对应的，虚拟机可以理解为模拟运行的一整套操作系统（提供了运行态环境和其他系统环境）和跑在上面的应用。

### 启动
    $ sudo docker run ubuntu:14.04 /bin/echo 'Hello world'
    $ sudo docker run --name="u14-test" -t -i ubuntu:14.04 /bin/bash

    --name 选项用于为启动的容器命名，默认是一个随机名称
    -t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上 
    -i 则让容器的标准输入保持打开。

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：
- 检查本地是否存在指定的镜像，不存在就从公有仓库下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
- 从地址池配置一个 ip 地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止
### 启动已终止容器并进入容器
    $ sudo docker start u14-test
    $ sudo docker attach u14-test
### 守护态/后台进程运行
    $ sudo docker run -d ubuntu:14.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
    $ sudo docker ps -a
    $ sudo docker logs # 通过 docker logs 获取容器输出内容
### 终止容器
除交互时使用 exit 退出与终止外，也可在宿主机中使用 docker stop 命令来终止一个指定容器

    $ sudo docker stop u14-test
### 进入容器-attach 命令
当容器在运行使用，可以使用 docker attach 命令进入容器

    $ sudo docker attach u14-test
使用 attach 命令有时候并不方便。当多个窗口同时 attach 到同一个容器的时候，所有窗口都会同步显示。当某个窗口因命令阻塞时,其他窗口也无法执行操作了。
### 进入容器-nsenter 命令 
    *** ***
### 导出容器
    $ sudo docker export 7691a814370e > u14_test.tar
### 导入容器
    $ cat u14_test.tar | sudo docker import - test/buntu:v1.0
    $ sudo docker images
*注：用户既可以使用 docker load 来导入镜像存储文件到本地镜像库，也可以使用 docker import 来导入一个容器快照到本地镜像库。这两者的区别在于容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也要大。此外，从容器快照文件导入时可以重新指定标签等元数据信息。
### 查看容器
```bash
docker ps # 查看正在运行的容器
docker ps -a # 查看所有容器：
```
### 删除容器
    $ sudo docker rm  u14-test

## 仓库
仓库（Repository）是集中存放镜像的地方。
一个容易混淆的概念是注册服务器（Registry）。实际上注册服务器是管理仓库的具体服务器，每个服务器上可以有多个仓库，而每个仓库下面有多个镜像。从这方面来说，仓库可以被认为是一个具体的项目或目录。例如对于仓库地址 dl.dockerpool.com/ubuntu 来说，dl.dockerpool.com 是注册服务器地址，ubuntu 是仓库名。
大部分时候，并不需要严格区分这两者的概念。
### Docker Hub
    登录
    $ sudo docker login # 不上传不需要登录
    查找镜像
    $ sudo docker search centos
    下载
    $ sudo docker pull centos
### 自动创建
自动创建（Automated Builds）功能对于需要经常升级镜像内程序来说，十分方便。 有时候，用户创建了镜像，安装了某个软件，如果软件发布新版本则需要手动更新镜像。。

而自动创建允许用户通过 Docker Hub 指定跟踪一个目标网站（目前支持 GitHub 或 BitBucket）上的项目，一旦项目发生新的提交，则自动执行创建。

要配置自动创建，包括如下的步骤：

- 创建并登陆 Docker Hub，以及目标网站；
- 在目标网站中连接帐户到 Docker Hub；
- 在 Docker Hub 中 配置一个自动创建；
- 选取一个目标网站中的项目（需要含 Dockerfile）和分支；
- 指定 Dockerfile 的位置，并提交创建。

之后，可以 在Docker Hub 的 自动创建页面 中跟踪每次创建的状态。
### 私有仓库-安装容器并运行
    $ sudo docker run -d -p 5000:5000 registry
    $ sudo docker run --name="docker-reg" -d -p 5000:5000 -v /home/zwang/docker-images:/tmp/registry registry
### 私有仓库-本机安装并运行
    $ sudo apt-get install -y build-essential python-dev libevent-dev python-pip liblzma-dev
    $ sudo pip install docker-registry
    $ cp config/config_sample.yml config/config.yml
    $ sudo gunicorn -c contrib/gunicorn.py docker_registry.wsgi:application
### 私有仓库-上传、下载、搜索镜像
    上传
    $ sudo docker images
    $ sudo docker tag ubuntu:14.04 127.0.0.1:5000/ubuntu:zwang
    $ sudo docker push 127.0.0.1:5000/ubuntu:zwang
    查看
    $ curl http://127.0.0.1:5000/v1/search
    下载
    $ sudo docker pull 127.0.0.1:5000/ubuntu:zwang
### 仓库配置文件
在 config_sample.yml 文件中，可以看到一些现成的模板段
*** ***

## 数据管理
### 数据卷
