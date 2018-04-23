---
title: SaltStack-api-ssh证书配置
date: 2015-10-31 10:36:30
category: 运维
toc: true
---

SaltStack-api ssh证书配置，并实现java代码单项握手验证

### 环境
* 本机MacOS 10.10.5
* 开发环境Eclipse
* master和minion机CentOS-7-Minimal
* 虚拟环境VMware Fuson 7.0.0 虚拟机
* master机ip：192.168.17.101
* master机SaltStack-api开放端口：8888

<!--more-->

### 安装Salt
1、master和minion安装过程参考[这里](http://docs.saltstack.cn/zh_CN/latest/topics/tutorials/walkthrough.html)

2、SaltStack-api安装请参考[这里](http://www.saltstack.cn/projects/cssug-kb/wiki/Salt-api-deploy-and-use)

注意：如果你使用的不是ContOS，而是Ubuntu那你在/etc/pki/tls/certs这个目录下是没有Makefile这个文件，考虑到使用Ubuntu的同学还是挺多的，我贴出一下这份Makefile文件[点击下载](https://github.com/babydance/babydance.github.io/blob/master/resources/Makefile)

最后你在Mac上执行

```shell
$ curl -k https://192.168.17.101:8888/login -H "Accept: application/x-yaml" -d username='salt' -d password='salt' -d eauth='pam'
```

可以看到结果![](http://image.hibabydance.com/151031salt1.png)

### 为客户端（api接口调用端）生成一个证书和一个证书仓库

```shell
$ keytool -genkey -v -alias client -keyalg RSA -keystore client_ks -dname "CN=192.168.2.145,OU=cn,O=cn,L=cn,ST=cn,C=cn" -storepass 123456 -keypass 123456
```

keytool命令参数如下：
| 参数         | 说明          |
| ------------|:-------------:|
|-genkey      |产生一个别名为client的证书，和别名为client_ks证书仓库|
|-alias       |产生别名|
|-keystore    |指定密钥库的名称(产生的各类信息将不在.jks文件中|
|-keyalg      |指定密钥的算法    |
|-validity    |指定创建的证书有效期多少天|
|-keysize     |指定密钥长度|
|-storepass   |指定密钥库的密码|
|-keypass     |指定别名条目的密码|
|-dname       |指定证书拥有者信息|

注意：CN=192.168.2.145这里的ip应该是你本地的ip或者是域名，因为服务器在双手验证的时候需要确认这个。

### 准备服务端（master机）的cer文件，即master机的公钥
获取方式一：
Safari浏览器打开 https://192.168.17.101:8888 ,这个连接在你配置完SaltStack-api之后并启动后就可以使用。会看考Welcome的Json的数据。
Safari会要求信任该证书
![](http://image.hibabydance.com/151031salt2.png)

点击查看证书并信任该证书，可以看到master机的证书信息
这时候打开钥匙串，左侧找到证书一栏，选择刚才的证书，证书名就是192.168.101，右键导出cer格式的公钥，重命名为server.cer。

### 服务端证书导入客户端证书仓库

```
$ keytool -import -trustcacerts -alias server -file server.cer -keystore client_ks
```

### java测试

```java
public class Test
{
	public static void certTest()throws Exception{  
	        String httpsURL = "https://192.168.17.101:8888";  
	        String trustStor="/Users/root/Desktop/client_ks";
	        System.setProperty("javax.net.ssl.trustStore", trustStor);  
	        URL myurl = new URL(httpsURL);
	        HttpsURLConnection con = (HttpsURLConnection) myurl.openConnection();  
	        con.setHostnameVerifier(hv);  
	        InputStream ins = con.getInputStream();  
	        InputStreamReader isr = new InputStreamReader(ins);  
	        BufferedReader in = new BufferedReader(isr);  
	        String inputLine=null;  
	        while ((inputLine = in.readLine()) != null) {  
	            System.out.println(inputLine);  
	        }  
	        in.close();  
	}  
	private static HostnameVerifier hv = new HostnameVerifier() {  
	        public boolean verify(String urlHostName, SSLSession session) {  
	            return urlHostName.equals(session.getPeerHost());  
	        }  
	};
	public static void main() throws Exception {
		certTest();
	}
}
```

### 使用saltstack-netapi调用SaltStack-api

在test文件夹下的com.suse.saltstack.netapi.client包内打开SaltStackClientTest在init方法前添加两行代码

```java
@Before
public void init() throws Exception {
        String trustStor="/Users/root/Desktop/client_ks";
        System.setProperty("javax.net.ssl.trustStore", trustStor);
        URI uri = URI.create("https://192.168.17.101:" + Integer.toString(MOCK_HTTP_PORT));
        client = new SaltStackClient(uri);
}
```

最后运行testLoginOk，一切就ok了！！！

注意：这里只是做了ssh的单向握手验证，因为只是本机认可了master机，master机并没有认可本机，需要做的就是把本机上面生成的client公钥导入到master机的证书库里。
