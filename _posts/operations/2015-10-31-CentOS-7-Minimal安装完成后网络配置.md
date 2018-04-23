---
title: CentOS-7-Minimal安装完成后网络配置
date: 2015-10-31 10:36:30
category: 运维
excerpt: "记录一下测试用的CentOS-7-Minimal虚拟机的的网络配置"
toc: true
classes: narrow
---

## 环境
* MacOS 10.10.5
* VMware Fuson 7.0.0 虚拟机
* 将要配置的ip=192.168.17.100
* 虚拟机装好后的mac地址：00:0c:29:ba:76:ba

## 配置步骤

### 安装好虚拟机，并登录

### 配置ifcfg-eth0

```shell
$ vi /etc/sysconfig/network-scripts/ifcfg-eth0
```

ifcfg-eth0文件本来是不存在的需要创建

然后添加如下内容

```shell
DEVICE="eth0"
NM_Controlled="no"
HWADDR="00:0c:29:09:e4:36"
ONBOOT="yes"
BOOTPROTO="static"
TYPE="Ethernet"
IPADDR=192.168.17.100
NETMASK=255.255.255.0
GATEWAY=172.16.137.2
BROADCAST=172.16.137.255
IPV6INIT=no
IPV6_AUTOCONF=no
```

其中HWADDR的查看方式

```shell
$ ip addr
```

 或者

```shell
$ ip link
```

IPADDR是你需要配置的ip地址

### 配置resolv.conf  

```shell
$ vi /etc/resolv.conf
```

添加如下内容

```shell
nameserver 172.16.137.2
```

### 配置route-eth0

```shell
$ vi /etc/sysconfig/network-scripts/route-eth0
```

添加如下内容

```shell
0.0.0.0 via 172.16.137.2
```

### 重启网卡

```shell
$ service network restart
```

重启网卡如果还是不行，就重启虚拟机吧

### 你会发现没有ifconfi命令，没有就自己装吧

```shell
$ yum install net-tools
```
