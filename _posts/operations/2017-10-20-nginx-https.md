---
title: nginx配置https
date: 2017-10-20 11:55:30
toc: true
category: 运维
---

在开发小程序的时候必须使用https的接口，记录一下简单的CA证书申请和配置过程备忘

<!--more-->

### 准备阿里的CA证书

1. 在阿里云的 **云盾控制台概览** 中选择 **CA证书服务**
2. 选择购买证书->Symantec->免费型DV SSL->立即购买
3. 返回CA证书订单列表，完善信息提交认证
4. 审核通过后点击下载

### 安装nginx

```shell
yum install nginx
```

yum 安装的 nginx 配置文件在/etc/nginx目录下，contos6.x版本和7.x版本的配置有些许不同，主配置文件在nginx.conf

###  配置nginx的ssl

1. 在Nginx的配置目录下创建cert目录，并且将下载的全部文件拷贝到cert目录中。
2. 打开 Nginx 配置目录中的 nginx.conf 文件，找到：

```shell
# HTTPS server
# #server {
# listen 443;
# server_name localhost;
# ssl on;
# ssl_certificate cert.pem;
# ssl_certificate_key cert.key;
# ssl_session_timeout 5m;
# ssl_protocols SSLv2 SSLv3 TLSv1;
# ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
# ssl_prefer_server_ciphers on;
# location / {
#
#
#}
#}
```

3. 将其修改为 (以下属性中ssl开头的属性与证书配置有直接关系，其它属性请结合自己的实际情况复制或调整) :

```shell
server {
    listen 443;
    server_name localhost;
    ssl on;
    root html;
    index index.html index.htm;
    ssl_certificate   /etc/nginx/xxxx.pem;
    ssl_certificate_key  /etc/nginx/xxxx.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        root html;
        index index.html index.htm;
    }
}
```

另外要开启压缩的配置顺便记录一下，最后整个配置可能是这样的,作为一个参考

```shell
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/
#如果nginx启动后有权限相关的问题，可能是启动的用户有问题
user root;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    gzip  on;
    # 开启gzip
    # 启用gzip压缩的最小文件，小于设置值的文件将不会压缩
    gzip_min_length 1k;
    # gzip 压缩级别，1-10，数字越大压缩的越好，也越占用CPU时间，后面会有详细说明
    gzip_comp_level 2;
    # 进行压缩的文件类型。javascript有多种形式。其中的值可以在 mime.types 文件中找到。
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    # 是否在http header中添加Vary: Accept-Encoding，建议开启
    gzip_vary on;
    # 禁用IE 6 gzip
    gzip_disable "MSIE [1-6]\.";

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;


    server {
        listen       443 ssl;
        #listen       [::]:80 default_server;

        ssl on;
        ssl_certificate   /etc/nginx/cert/xxxxx.pem;  #注意这的值和你在cert里的命名一样，这里和阿里云给的文件名一样就行
        ssl_certificate_key  /etc/nginx/cert/xxxxx.key;
        ssl_session_timeout 5m;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;

        server_name  www.test.com;
        root         /home/admin/app;

        location /api/ {
             proxy_pass http://localhost:8080;
        }
        location /dashboard/  {
             proxy_pass http://localhost:8080;
        }

        location / {
            index /index.html;
        }
    }

    server {
        listen       80;
        #listen       [::]:80 default_server;
        server_name  admin.test.com;
        root         /home/admin/dashboard;

        location / {
             index index.html;
        }
    }
}
```
