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