---
title: 阿里云服务器ssh经常一段时间就断掉解决办法
date: 2017-11-19 22:40:30
category: 运维
---

### 常用阿里云的服务器，但是由于测试原因，经常

```shell
Connection to 111.231.246.192 port 22: Broken pipe
```

所以还是要要处理一下，解决方案记录一下备查

```shell
$ vim /etc/ssh/sshd_config
```

找到下面两行,如果没有就自己加一行

```shell
#ClientAliveInterval 0
#ClientAliveCountMax 3
```

去掉注释，改成

```shell
ClientAliveInterval 30
ClientAliveCountMax 10000
```

这两行的意思分别是

1、客户端每隔多少秒向服务发送一个心跳数据

2、客户端多少次超时没有响应，可以设置的稍微多一点

重启sshd服务

```shell
$ service sshd restart
```
