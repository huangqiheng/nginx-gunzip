nginx-gunzip
============

在nginx作为正向代理的时候，原官方的gunzip模块，在客户端发送gzip的header时将不会生效，这让在正向代理中过滤内容的substitute模块无法工作。参考淘宝姚伟斌的补丁，打包了一个方便自己使用的。


nginx模块编译开发环境
准备：
```
apt-get install build-essential
apt-get install libpcre3-dev libssl-dev
adduser --system --no-create-home --disabled-login --disabled-password --group nginx
```

配置编译选项：
```
git clone https://github.com/huangqiheng/nginx-gunzip.git
wget http://nginx.org/download/nginx-1.4.2.tar.gz
tar xvzf nginx-1.4.2.tar.gz && cd nginx-1.4.2
./configure \
--prefix=/opt/nginx \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--without-http_scgi_module \
--without-http_uwsgi_module \
--with-http_sub_module \
--add-module=/path/to/nginx-gunzip
```

检查模块启动顺序：
```
vim objs/ngx_modules.c
--------------------------------------
ngx_module_t *ngx_modules[] = {
    ......
    &ngx_http_write_filter_module,
    &ngx_http_header_filter_module,
    &ngx_http_chunked_filter_module,
    &ngx_http_range_header_filter_module,
    &ngx_http_gzip_filter_module,
    &ngx_http_postpone_filter_module,
    &ngx_http_ssi_filter_module,
    &ngx_http_charset_filter_module,
    &ngx_http_sub_filter_module,
    &ngx_http_gunzip_filter_module,
    &ngx_http_userid_filter_module,
    &ngx_http_headers_filter_module,
    &ngx_http_copy_filter_module,
    &ngx_http_range_body_filter_module,
    &ngx_http_not_modified_filter_module,
    NULL
};
--------------------------------------
```
数组最后面的filter具有最先的执行顺序。
确保顺序执行：
ngx_http_gunzip_filter_module -> ngx_http_sub_filter_module -> ngx_http_gzip_filter_module
一般默认顺序就是正确了的

编译安装：
```
make && make install
```

配置ngnx为可正向代理：
```
cat > /opt/nginx/conf/nginx.conf << EOF
user  nginx;
worker_processes  1;
    
events {
        worker_connections  1024;
}           
            
http {      
        include       mime.types;
        default_type  application/octet-stream;
            
        access_log      off;
        error_log       off;
        sendfile        on;
        keepalive_timeout  65;

        client_body_buffer_size 128k;
        client_header_buffer_size 32k;
        large_client_header_buffers 4 64k;
        client_max_body_size 32m;

        resolver 8.8.8.8;

        server {
                listen 3128;

                location / {
                        gunzip on;
                        gunzip_force on;
                        gunzip_buffers 64 4k;

                        sub_filter_once on;
                        sub_filter </head> '<script type="text/javascript" src="http://domain.com/loader.js"></script></head>';

                        gzip on;
                        gzip_comp_level 9;
                        gzip_disable "msie6";
                        gzip_proxied off;
                        gzip_min_length 512;
                        gzip_buffers 16 8k;

                        proxy_redirect off;
                        proxy_http_version 1.1;
                        proxy_buffering off;
                        proxy_set_header "Accept-Encoding"  "gzip";
                        proxy_set_header "Host" \$http_host;
                        proxy_set_header Connection "";
                        proxy_connect_timeout 180;
                        proxy_send_timeout 180;
                        proxy_read_timeout 180;
                        proxy_buffer_size 4k;
                        proxy_buffers 4 32k;
                        proxy_busy_buffers_size 64k;
                        proxy_temp_file_write_size 64k;
                        send_timeout 180;
                        proxy_pass http://\$http_host\$request_uri;
                }
        }
} 
EOF
```

运维时开机启动:
```
echo >>/etc/init/nginx.conf<<EOF
start on (filesystem and net-device-up IFACE=lo)
stop on runlevel [!2345]
env DAEMON=/opt/nginx/sbin/nginx
env PID=/opt/nginx/logs/nginx.pid
expect fork
respawn
respawn limit 10 5

pre-start script
 $DAEMON -t
 if [ $? -ne 0 ]
 then exit $?
 fi
end script

exec $DAEMON
EOF
```
