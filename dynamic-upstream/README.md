Nginx 是一款开源、高性能、高可靠的 Web 和反向代理服务器，性能是 Nginx 最重要的考量，其占用内存少、并发能力强。
Nginx 最常见的使用场景就是反向代理，Nginx 接收客户端的请求并通过相应的负载均衡算法将流量转发给后端的多台应用服务器。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710195924.png)

## 传统做法

通常我们先会配置一个 upstream 地址池，包含后端的多台应用服务器，然后通过 proxy_pass 将流量分发给 upstream 中的成员。

```nginx
http {
    
    upstream upstream_server{
        server 192.168.1.134:81;
        server 192.168.1.134:82;
    }

    server {
        listen       80;
        server_name localhost;

        location / {
            proxy_pass http://upstream_server;
        }
    }
}
```
假如现在由于应用服务器压力比较大，要新增一台服务器，那么需要修改 upstream 为：

```nginx
upstream upstream_server{
    server 192.168.1.134:81;
    server 192.168.1.134:82;
    #新增的服务器
    server 192.168.1.134:83;
}
```
修改完成之后，需要通过 `nginx -s reload` 命令重新加载配置，才能使配置生效。虽然 Nginx 可以做到平滑地重载配置，但是每次应用服务器增加或删除时都要改动 Nginx 显得并不是那么智能。如果有大量的 Nginx 需要管理，每次都需要手动操作将会极大地增加运维的负担。

## Dynamic Upstream

 基于传统做法的弊端，我们引入了注册中心保存应用服务信息，Nginx 通过动态获取注册中心中的服务信息，更新 upstream 配置，无需人为干预和重启。实际生产应用中我们可以将 CMDB  和 注册中心整合，管理人员只需要在 CMDB 上维护应用服务信息即可。

Nginx 第三方模块 [`nginx-upsync-module`](https://github.com/weibocom/nginx-upsync-module) 支持通过注册中心动态发现 upstream 信息。目前 `nginx-upsync-module` 模块支持 Consul 和 Etcd 作为 注册中心。

另外开源版本的 Nginx 默认只支持被动的健康检查，只有当客户端访问时，才会发起对后端节点的探测。假设本次请求中， Nginx 转发的后端节点正好出现了异常，Nginx 会将请求再转交给另一个 upstream 中的节点处理，所以不会影响到这次请求的正常进行，但是会影响效率，因为多了一次转发。并且自带模块无法做到预警。因此我们还使用了第三方模块 [`nginx_upstream_check_module`](https://github.com/yaoweibin/nginx_upstream_check_module) 用于健康检查，该模块不仅支持主动的健康检查还提供了 WebUI 用于查看健康检查状态。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710183225.png)

本示例 github 地址：https://github.com/cr7258/nginx-lab/tree/master/dynamic-upstream

目前还有其他产品支持动态配置，不仅仅是 upstream，还包括了其他方面的配置。例如 Nginx 的商业版本 Nginx Plus，和云原生结合得比较好的 Envoy、Kong、Traefik 等等。大家有兴趣可以自行了解。

## 注册中心 Consul 搭建

我们在一台虚拟机上启动 3 个 Consul 服务，组成一个伪集群。

下载安装包。

```sh
wget https://releases.hashicorp.com/consul/1.9.3/consul_1.9.3_linux_amd64.zip
unzip consul_1.9.3_linux_amd64.zip
mv consul /usr/local/bin/
```

创建相关目录：

```sh
mkdir /data/consul && cd $_
mkdir -pv /data/consul/node{1..3}
```

创建 3 个 Consul 节点使用的配置文件：

**node1**

vim /data/consul/node1/consul_config1.json
```sh
{
  "datacenter": "dev",
  "data_dir": "/data/consul/node1",
  "log_file": "/data/consul/node1/consul.log",
  "log_level": "INFO",
  "server": true,
  "node_name": "node1",
  "ui": true,
  "bind_addr": "192.168.1.134",
  "client_addr": "192.168.1.134",
  "advertise_addr": "192.168.1.134",
  "bootstrap_expect": 3,
  "ports":{
    "http": 8510,
    "dns": 8610,
    "server": 8310,
    "serf_lan": 8311,
    "serf_wan": 8312
    }
}
```

**node2**

vim /data/consul/node2/consul_config2.json

```sh
{
  "datacenter": "dev",
  "data_dir": "/data/consul/node2",
  "log_file": "/data/consul/node2/consul.log",
  "log_level": "INFO",
  "server": true,
  "node_name": "node2",
  "ui": true,
  "bind_addr": "192.168.1.134",
  "client_addr": "192.168.1.134",
  "advertise_addr": "192.168.1.134",
  "bootstrap_expect": 3,
  "ports":{
    "http": 8520,
    "dns": 8620,
    "server": 8320,
    "serf_lan": 8321,
    "serf_wan": 8322
    }
}
```

**node3**

vim /data/consul/node3/consul_config3.json


```sh
{
  "datacenter": "dev",
  "data_dir": "/data/consul/node3",
  "log_file": "/data/consul/node3/consul.log",
  "log_level": "INFO",
  "server": true,
  "node_name": "node3",
  "ui": true,
  "bind_addr": "192.168.1.134",
  "client_addr": "192.168.1.134",
  "advertise_addr": "192.168.1.134",
  "bootstrap_expect": 3,
  "ports":{
    "http": 8530,
    "dns": 8630,
    "server": 8330,
    "serf_lan": 8331,
    "serf_wan": 8332
    }
}
```

启动 Consul 集群：

```sh
nohup consul agent -config-file=/data/consul/node1/consul_config1.json > /dev/null 2>&1 &
nohup consul agent -config-file=/data/consul/node2/consul_config2.json -retry-join=192.168.1.134:8311 > /dev/null 2>&1 &
nohup consul agent -config-file=/data/consul/node3/consul_config3.json -retry-join=192.168.1.134:8311 > /dev/null 2>&1 &
```

启动之后，便可以通过 http://192.168.1.134:8510 访问，此处 192.168.1.134:8510 是 Leader 角色。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710185029.png)


## 启动后端应用服务

后端服务是用 Nginx 启动的 Web 服务。准备 3 个后端服务，IP 和 Port 分别是：

```sh
192.168.1.134:81
192.168.1.134:82
192.168.1.134:83
```

配置文件内容如下，3 个服务的配置基本一样，只是改了相关的端口和路径：
```nginx
user root;
events{}
http{
server {
    listen       81 default_server;
    listen       [::]:81 default_server;
    server_name  127.0.0.1;
    root         /root/myapp/nginx/html/81;
    server_tokens off;

    gzip on;
    gzip_buffers 16 8k;
    gzip_comp_level 6;
    gzip_http_version 1.1;
    gzip_min_length 256;
    gzip_proxied any;
    gzip_vary on;
    gzip_types text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml text/javascript application/javascript application/x-javascript text/x-json application/json application/x-web-app-manifest+json text/css text/plain text/x-component font/opentype application/x-font-ttf application/vnd.ms-fontobject image/x-icon;
    gzip_disable "msie6";


    location / {
        expires max;  
        open_file_cache max=1000 inactive=20s; 
        open_file_cache_valid 30s; 
        open_file_cache_min_uses 2; 
        open_file_cache_errors on;
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
  }
}
```
html 文件如下：

```html
[root@nginx-plus1 nginx]# cat html/81/index.html 
<html>
<head>
    <meta charset="utf-8">
    <title>server1</title>
</head>
<body style="background-color:blue;">
    <h1>Server 1 url 1<h1>
</body>
</html>
```

启动 3 个后端服务，在启动 Nginx 的时候指定配置文件即可：

```sh
sbin/nginx -c /root/myapp/nginx/81.conf
sbin/nginx -c /root/myapp/nginx/82.conf
sbin/nginx -c /root/myapp/nginx/83.conf
```


访问 3 个应用服务：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710185244.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710185254.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710185302.png)

## 编译 Nginx 

实现 Dynamic Upstream 需要添加 `nginx-upsync-module` 和`nginx_upstream_check_module` 两个第三方模块，在编译 Nginx 的时候要将这两个模块添加进去。这里准备了一个 Dockerfile，使用 `docker build -t 镜像名:标签名 .` 就可以构建出一个编译好的 Nginx Docker 镜像。

```sh
FROM debian:stretch-slim

RUN useradd  www && \
mkdir -p /logs/nginx/  /webserver/nginx /webserver/nginx/conf/upsync && \
chown -R www:www /logs/nginx/  /webserver/nginx && \
echo 'deb http://mirrors.163.com/debian/ stretch main non-free contrib' > /etc/apt/sources.list && \
echo 'deb http://mirrors.163.com/debian/ stretch-updates main non-free contrib' >> /etc/apt/sources.list && \
echo 'deb-src http://mirrors.163.com/debian/ stretch main non-free contrib' >> /etc/apt/sources.list && \
echo 'deb-src http://mirrors.163.com/debian/ stretch-updates main non-free contrib' >> /etc/apt/sources.list && \
echo 'deb-src http://mirrors.163.com/debian/ stretch-backports main non-free contrib' >> /etc/apt/sources.list && \
echo 'deb-src http://mirrors.163.com/debian-security/ stretch/updates main non-free contrib' >> /etc/apt/sources.list && \
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
apt-get update && \
apt-get install -y wget vim net-tools unzip libjemalloc-dev && \
apt-get build-dep -y nginx

RUN \
cd /usr/local/src/ && \
wget -c http://nginx.org/download/nginx-1.14.2.tar.gz && \
wget -c https://www.openssl.org/source/old/1.0.2/openssl-1.0.2m.tar.gz && \
wget -c https://github.com/simplresty/ngx_devel_kit/archive/v0.3.1rc1.tar.gz && \
wget -c https://github.com/openresty/lua-nginx-module/archive/v0.10.11.tar.gz && \
wget -c https://github.com/xiaokai-wang/nginx_upstream_check_module/archive/master.zip -O nginx_upstream_check_module.zip && \
wget -c https://github.com/weibocom/nginx-upsync-module/archive/master.zip -O nginx-upsync-module.zip && \
tar zxf ./nginx-1.14.2.tar.gz && rm nginx-1.14.2.tar.gz && \
tar zxf ./openssl-1.0.2m.tar.gz && rm openssl-1.0.2m.tar.gz && \
tar zxf ./v0.3.1rc1.tar.gz && rm v0.3.1rc1.tar.gz && \
tar zxf ./v0.10.11.tar.gz && rm v0.10.11.tar.gz &&  \
unzip ./nginx_upstream_check_module.zip && rm nginx_upstream_check_module.zip && \
unzip ./nginx-upsync-module.zip && rm nginx-upsync-module.zip

RUN \
cd /usr/local/src/nginx-1.14.2 &&\
patch -p1 < /usr/local/src/nginx_upstream_check_module-master/check_1.12.1+.patch &&\
./configure \
--prefix=/webserver/nginx \
--user=www --group=www --with-pcre \
--with-stream \
--with-http_v2_module \
--with-http_ssl_module \
--with-ld-opt=-ljemalloc \
--with-http_realip_module \
--with-http_gzip_static_module \
--with-http_stub_status_module \
--http-log-path=/logs/nginx/access.log \
--error-log-path=/logs/nginx/error.log \
--with-openssl=/usr/local/src/openssl-1.0.2m \
--add-module=/usr/local/src/ngx_devel_kit-0.3.1rc1 \
--add-module=/usr/local/src/lua-nginx-module-0.10.11 \
--add-module=/usr/local/src/nginx_upstream_check_module-master \ 
--add-module=/usr/local/src/nginx-upsync-module-master && \
make && \
make install
```

另外我也准备了一个已经构建好的镜像：registry.cn-shanghai.aliyuncs.com/public-namespace/nginx-dynamic-upstream:v1.0.0 ，可以直接拿来使用。

## 准备 Nginx 动态更新的配置文件

配置 nginx.conf 文件：

```nginx
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    upstream app {
        upsync 192.168.1.134:8510/v1/kv/upstreams/app/ upsync_timeout=6m upsync_interval=500ms upsync_type=consul strong_dependency=off;
        upsync_dump_path /webserver/nginx/conf/app.conf; # 当consul故障时候，就可以把此作为备份配置文件
        include /webserver/nginx/conf/app.conf; # 准备一个兼容的nginx测试文件，如果没有第一次启动会起不来
        check interval=1000 rise=2 fall=2 timeout=3000 type=http default_down=false;
        check_http_send "HEAD / HTTP/1.0\r\n\r\n";
        check_http_expect_alive http_2xx http_3xx;
        }
    server {
        listen       80;
        server_name  localhost;
        location / {
            proxy_pass http://app;
        }
        location /upstream_list {
            upstream_show;
        }
        location /upstream_status {
            check_status;
            access_log off;
        }
    }
}
```

app.conf 文件里随便写上一个 IP 和 Port 信息，可以是无法访问的服务，因为 Nginx 的 upstream 中必须要有地址才能启动 Nginx。我们后面会通过在 Consul 上注册服务让 Nginx 动态更新 Upstream。

```sh
server 0.0.0.0:12345 weight=1 max_fails=2 fail_timeout=10s;
```

在本地的 Mac 电脑上通过 Docker 启动 Nginx 容器：

```sh
docker run -d --name nginx-dynamic-upstream \
-v /Users/chengzhiwei/lab/docker-lab/nginx/dynamic-upstream/nginx.conf:/webserver/nginx/conf/nginx.conf \
-v /Users/chengzhiwei/lab/docker-lab/nginx/dynamic-upstream/app.conf:/webserver/nginx/conf/app.conf \
-p 80:80 -p 443:443 \
registry.cn-shanghai.aliyuncs.com/public-namespace/nginx-dynamic-upstream:v1.0.0 \
/webserver/nginx/sbin/nginx  -g "daemon off;"
```

通过 curl 命令发送  HTTP 请求往 Consul 中注册两个新的服务。

```sh
curl -X PUT -d '{"weight":1, "max_fails":2, "fail_timeout":10, "down":0}' http://192.168.1.134:8510/v1/kv/upstreams/app/192.168.1.134:81
curl -X PUT -d '{"weight":1, "max_fails":2, "fail_timeout":10, "down":0}' http://192.168.1.134:8510/v1/kv/upstreams/app/192.168.1.134:82
```

通过 http://localhost/upstream_list 查看 upstream 主机：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710191904.png)

通过 http://localhost/upstream_status 可以看到应用服务的健康检查状态：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710191917.png)

访问 http://localhost 可以代理到后端的应用服务：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710194548.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710194632.png)

此时我们停掉 192.168.1.134:81 的服务：

```sh
#查看监听 81 端口的进程号
[root@nginx-plus1 nginx]# lsof -i:81
COMMAND   PID USER   FD   TYPE    DEVICE SIZE/OFF NODE NAME
nginx   26047 root    6u  IPv4 202389912      0t0  TCP *:81 (LISTEN)
nginx   26047 root    7u  IPv6 202389913      0t0  TCP *:81 (LISTEN)
nginx   26048 root    6u  IPv4 202389912      0t0  TCP *:81 (LISTEN)
nginx   26048 root    7u  IPv6 202389913      0t0  TCP *:81 (LISTEN)
#停止服务
[root@nginx-plus1 nginx]# kill 26047
```

此时查看健康检查状态，发现 81 端口的服务已经被置为 down 了。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710192146.png)

此时再访问 http://localhost 就只能访问到端口为 82 的服务了。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210710194632.png)

## 参考链接
* https://cloud.tencent.com/developer/article/1802712?from=article.detail.1628590
* https://github.com/weibocom/nginx-upsync-module
* https://cloud.tencent.com/developer/article/1648733?from=article.detail.1802712

## 欢迎关注
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210306213609.png)
