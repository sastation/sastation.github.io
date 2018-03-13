---
title: Docker 进阶
date: 2016-12-28 22:36
tags:
    - 技术
    - Docker

---

# Docker 进阶

## 制作镜像 - Dockerfile - Example
```bash
vi Dockerfile
    # This is private image
    FROM debian:7
    MAINTAINER Docker David Wang<sa.station@gmail.com>
    COPY sources.deb7 /etc/apt/sources.list
    RUN apt-get -qq update \
    && apt-get -qqy \
        --no-install-recommends \
        --no-install-suggests \
        install vim-tiny net-tools procps \
    && apt-get -qqy autoremove \
    && apt-get -qqy clean
```
要点：
1. 能用 COPY 不用 ADD
2. 相关命令尽可能在一个 RUN 命令中完成 （可以减少镜像层数，控制尺寸）
3. 若有很多命令需要执行，可以考虑准备一个初始化脚本，先 COPY 再 RUN

## 缩减镜像文件的层数与尺寸
1. 按照 [Dockerfile 最佳实践](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/) 制作镜像
2. 使用 save/load 命令操作镜像可以减少层数并保留制作过程信息
3. 使用 export/import 命令操作容器可以删除所有中间层数（缺点是丢失所有复用链接，尺寸为实际大小）

```bash
    repo='offical'
    tmp='temp'
    fs='offical'

    docker build --force-rm -t=$tmp . 
    cid=`docker run -d $tmp` #生成容器
    docker export -o $fs.tar $cid
    docker import -m $repo $fs.tar $repo
    docker rm $cid #删除容器
    docker rmi $tmp #删除临时镜像
    rm $fs.tar
```

## 进入正在运行的容器内部
- **exec**: docker exec -it <container> /bin/bash
    + 新开一个终端进入
- attach: docker attach <container>
    + 容器中需要有个shell正在运行
    + 所有 attach 都在一个进程中
- nsenter: 
    + 第三方程序，需要安装

## 卷(Volume)的使用
- 映射宿主目录进容器(可读写)：docker run -it -v /home/zwang/test:/root/bin --name test debian:7 /bin/bash
- 映射宿主目录进容器(只读)：  docker run -it -v /home/zwang/test:/root/bin:ro --name test debian:7 /bin/bash


## 启动容器时自动启动服务
> 在容器中启动服务，若要保持容器运行状态不退出，需要有前台进程。有两种方法实现，一是服务本身运行在前台；二是服务运行在后台，同时前台有其他进程一直在运行。

1. **CMD** && ENTRYPOINT in Dockerfile
    - **CMD ['command','param1','param2',...]**, exec模式，直接运行command
    - CMD command param1 param2 ...，shell模式，使用 /bin/sh -c 'command param1 param2 ...'运行 
    - CMD 与 ENTRYPOINT 区别: 
        + CMD会被命令行 docker run <image> <command>中的command覆盖
        + ENTRYPOINT会将命令行 docker run <image> <command>中的command作为新的参数接收处理 (shell模式无效)
2. ~/.bash_rc
    - 配合 "docker run -itd <image> /bin/bash" 进行调用
4. docker run 
    - docker run -d <image> command param1 param2 ...  
3. supervisor
    - 起 watchdog 的作用

## 为容器配置固定IP地址 - Docker Bridge (不同宿主机网段)
```bash
docker network create -d bridge --subnet=192.168.200.0/24 --gateway=192.168.200.1 -o com.docker.network.bridge.name=br-home homenet

docker run -it --rm --net=homenet --ip=192.168.200.200 zwang:base bash
```

## 为容器配置固定IP地址- MACvlan (同宿主机网段)
1. 确定宿主机内核>=3.9
2. 配置 MACvlan

```bash
# open promisc model
ifconfig eth0 promisc

# Create a network for docker
docker network create -d macvlan --subnet=192.168.100.0/24 --gateway=192.168.100.1 --ip-range=192.168.100.230/28 -o parent=eth0 macvlan

# start a container
docker run -it --net=macvlan --ip=192.168.100.231 --rm zwang:base bash

# test
ping -c3 192.168.100.1
```

## 为容器配置固定IP地址 - IPvlan
1. kernel >= 3.19

```bash
docker network  create -d ipvlan --subnet=192.168.100.0/24 --ip-range=192.168.100.230/28 -o ipvlan_mode=l2 -o parent=eth0 ipvlan

docker run -it --net=ipvlan --ip=192.168.100.230 --rm zwang:base bash
```

## ~~为容器配置固定IP地址 - pipework~~ **不建议**
1. 在宿主机上安装 pipework

```bash
git clone https://github.com/jpetazzo/pipework
cd pipework
cp pipework  /usr/local/bin/
chmod +x /usr/local/bin/pipework
```
2. 配置宿主机网桥接口`br0`

```bash
apt-get install bridge-utils
# 单次生效
brctl addbr br0
ip link set br0 up
ip addr add 192.168.100.20/24 dev br0
ip addr del 192.168.100.20 dev ens33
brctl addif br0 ens33
route del default
route add default gw 192.168.100.1

# 永久生效
vi /etc/network/interfaces
    auto eth0
    iface eth0 inet manual

    auto br0
    iface br0 inet static
    address 192.168.100.13
    netmask 255.255.255.0
    gateway 192.168.100.1
    bridge_ports eth0
    bridge_stp off
    bridge_fd 0
    bridge_maxwait 0
    dns-nameservers 192.168.100.16 192.168.100.17
    dns-search lan
/etc/init.d/network restart
```
3. 使用 `--network="none"`启动容器，再用`pipework配置网络`

```bash
# 另外的网段s
ip route add 192.168.101.0/24 via 192.168.100.13
iptables -t nat -A POSTROUTING -s 192.168.101.0/24 -j MASQUERADE

docker run -idt --name test --net=none zwang:bind9 bash
sudo pipework br0 test 192.168.101.19/24@192.168.100.13
#sudo pipework docker0 test 172.17.0.19/16@172.17.0.1
```
4. 更改IP后需要使用`exec`命令启动或重启服务

```bash
docker exec -d test service bind9 <start|restart>
```

**************
##Hyper-V 中的注意事项
1. 默认情况下无法为 Docker 分配同宿主机网段，原因是没有打开混杂模式
2. 若要打开混杂模式，需要设置 “启用MAC地址欺骗”功能（Enable MAC Address Spoofing）
    - 虚拟机>设置>网络适配器(打开加号)>高级功能>启用MAC地址欺骗
**************

## Docker with UFW
###问题：
>在ubuntu中若容器使用`-p port:port`启动，无论ufw如何配置，都会暴露在公网中。

###原因：
>Docker默认会操作iptables转发与配置，由于docker在iptables中配置的链表(主要是nat链表)的位置高于ufw配置的链表(主要是filter链表)，所以造成ufw中的配置对Docker无效。即Docker使用-p参数传出的端口将失去ufw的保护而暴露在公网中。

###解决方法：推荐方法二+进阶
**方法一**：绑定本地IP，例如：`docker run -it -p 127.0.0.1:80:80 ubuntu:vps bash`，这样只有本地才能访问，而公网无法访问，也无需修改ufw与iptables。

**方法二**: 修改/etc/default/ufw与/etc/default/docker改变默认设置
> docker的网段为 192.168.0.0/20

- U16.04
```bash
vi /etc/systemd/system/multi-user.target.wants/docker.service
    ExecStart=<...> --iptables=false
systemctl daemon-reload; systemclt restart docker
```
- U14.04
```bash
vi /etc/default/docker
    DOCKER_OPTS="... --iptables=false"
service docker restart
```

```bash
vi /etc/default/ufw
    DEFAULT_FORWARD_POLICY="ACCEPT"
ufw disable; ufw enable (ufw reload)

# 确认iptables中存在如下规则，若没有需添加
iptables -t nat -A POSTROUTING ! -o docker0 -s 172.17.0.1/16 -j MASQUERADE
```
>按上述处理后，再启动容器其网络即可以通过ufw进行管理。不过**问题**是容器内所接受请求的**客户端IP**都会被宿主机的IP替代。

**方法二的进阶**：若需要真实的客户端IP，需手工处理iptables中的nat链， `iptables -t nat -A DOCKER ! -i docker0 -p <prot> -m <prot> --dport <port> -j DNAT --to-destination <ip:port>` , **注意** ufw配置将会绕过
> 以bind的53端口为例，bind容器ip为192.168.0.2

```bash
iptables -t nat -A DOCKER ! -i docker0 -p tcp -m tcp --dport 53 -j DNAT --to-destination 172.17.0.1:53
iptables -t nat -A DOCKER ! -i docker0 -p udp -m udp --dport 53 -j DNAT --to-destination 172.17.0.1:53
```

**TIPs: iptables**
- 查看NAT链内容并显示等号：`iptables -t nat -v -n -L <链表名称，空为全部> --line-number` 
- 删除指定规则：`iptables -t nat -D DOCKER <行号>` 
- 定制转发规则
```bash
iptables -t nat -A POSTROUTING ! -o docker0 -s 192.168.0.0/20 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.0.2 -p udp -m udp -d 192.168.0.2 --dport 53 -j MASQUERADE
iptables -t nat -A DOCKER ! -i docker0 -p udp -m udp --dport 53 -j DNAT --to-destination 192.168.0.2:53
```

## 修改 docker0 网络的 IP 范围
- 修改方法：使用 systemd 的系统 (ubuntu 16.04, Debian 8, CentOS 7 etc.)

```bash
vi /etc/systemd/system/multi-user.target.wants/docker.service
    ExecStart= <...> --bip=192.168.99.1/24
systemctl daemon-reload; systemctl restart docker
```
- 修改方法：使用 upstart 的系统 (Ubuntu 14.04、Debian 7 etc.)

```bash
vi /etc/default/docker
    DOCKER_OPTS="<...> --bip=192.168.99.1/24"
service docker restart
```