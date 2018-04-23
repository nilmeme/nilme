---
title: 初始化一台服务器并做简单的自动化部署
date: 2017-07-03 16:57:41
category: 运维
toc: true
classes: narrow
---

每次重新安装一台服务器，重复流程，但是有时候又会忘记一些东西，这里做一下记录，方便后面的查找和回忆。

<!--more-->

## 开始

### 安装jdk

选择正确版本的jdk安装，主要是区分32位操作系统还是64位。

```shell
$ rpm -ivh jdk-xxxx-linux-xxxx.rpm
```

rpm的方式安装会自动的配置好环境变量， **jdk** 的安装目录在 **/usr/java/jdk1.8.0_131**

若没有自动配置

```shell
$ vi /etc/profile
```

在profile文件末位加上如下内容：

```shell
#set java JDK
JAVA_HOME=/usr/local/jdkxxx/
JRE_HOME=/usr/local/jdkxxx/jre/
PATH=$PATH:$JAVA_HOME/bin:$JRE_home/bin
CLASSPATH=$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar
export JAVA_HOME
export JRE_HOME
export PATH
export CLASSPATH
```

### 安装maven

```shell
wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
yum -y install apache-maven
yum -y install ant
```

这种方式安装快速方便，只要加一下apache源配置就可以了

### 安装git、tomcat

略

## 自动化部署

这里简单的自动部署使用的是[webhookit](https://github.com/hustcc/webhookit),还有更复杂的方式[Git WebHook](https://github.com/NetEaseGame/git-webhook)。

### 安装pyhon

有时候服务器自带的python是2.6的，**webhookit** 不支持2.6的pyhton，所以要升级一下。并同时更新一下 **pip**

```shell
yum groupinstall -y "Development tools"

yum -y install gcc gcc-c++ make zlib-devel pcre-devel openssl-devel install perl perl-devel perl-ExtUtils-Embed libcurl-devel lrzsz  lzo lzop  libcurl-devel pycurl  zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel python-devel

wget http://python.org/ftp/python/2.7.10/Python-2.7.10.tar.xz

tar xf Python-2.7.10.tar.xz
cd Python-2.7.10

./configure --prefix=/usr/local --enable-unicode=ucs4 --enable-shared LDFLAGS="-Wl,-rpath /usr/local/lib"

make && make altinstall

wget https://bootstrap.pypa.io/get-pip.py
python2.7 get-pip.py
```

### 安装 **webhookit**

```shell
$ pip install webhookit
```

### 配置 **webhookit**

config-auto-deploy.py

```shell
#webhook request from `repo_name` on branch `branch_name`,
# will exec SCRIPT on servers config in the array.
WEBHOOKIT_CONFIGURE = {
    # a web hook request can trigger multiple servers.
    '你的项目/master': [{
        # if exec shell on local server, keep empty.
        'HOST': '',  # will exec shell on which server.
        'PORT': '',  # ssh port, default is 22.
        'USER': '',  # linux user name
        'PWD': '',  # user password or private key.
        # The webhook shell script path.
        'SCRIPT': '/root/exec_hook_shell.sh'
    }]
}
```

/root/exec_hook_shell.sh就是webhook成功后要调用的脚本。

### 编写自动部署脚本

exec_hook_shell.sh

```shell
#!/bin/bash
#关闭tomcat
cd /root/apache-tomcat/bin
./shutdown.sh

sleep 5  #具体时间就看你得webapp在调用shutdown.sh后多久后处于僵死状态

ps -ef | grep 'tomcat' | grep -v grep| awk '{print $2}' | xargs kill -9

sleep 2  #给kill进程一点时间

#删除旧项目
cd /root/apache-tomcat/webapps/ROOT/
rm -rf *
#git更新项目
cd /root/xxxx/
git pull
#删除旧的编译文件
cd /root/xxxx/target/
rm -rf *
#重新打包
cd /root/xxxx/
mvn package
#复制到tomcat目录下
cp /root/xxxx/target/xxxx.war /root/apache-tomcat/webapps/ROOT/
#解压war
cd /root/apache-tomcat/webapps/ROOT/
unzip xxxx.war
#启动tomcat
cd /root/apache-tomcat/bin
./startup.sh
```

中间应该还有有备份等等操作，这里只是简略的步奏。

### 启动 **webhookit**

```shell
$ webhookit -c config-auto-deploy.py
```

### git配置webhook

这时候只要有git push的动作，服务器就可以自动部署了，建议这套方案只在测试服务器上使用。
