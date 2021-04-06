---
title: "Nginx 正向代理、反向代理、CORS跨域、负载均衡配置"
date: 2021-04-06T13:55:35+08:00
draft: true
target: ["Nginx", "Nginx 正向代理", "Nginx 反向代理", "Nginx 负载均衡", "Nginx 跨域", "Nginx 自定义日志"]
categories: ["Nginx 常用代理、负载均衡配置"]
---

Nginx 正向代理、反向代理、CORS跨域、负载均衡配置

<!--more-->

## 正向代理

```bash
# 添加如下配置到 nginx.conf
server {
    # 指定 DNS 服务器 IP 地址 
    resolver 8.8.8.8;
    listen 8089;
    location / {
        # 设定代理服务器的协议、地址及参数 
        proxy_pass $scheme://$host$request_uri;
        proxy_set_header HOST $host;
        proxy_buffers 256 4k;
        proxy_max_temp_file_size 0k;
        proxy_connect_timeout 30;
        proxy_send_timeout 60;
        proxy_read_timeout 60;
        proxy_next_upstream error timeout invalid_header http_502;
    }
}

# 测试
curl  -I --proxy 192.168.33.10:8089 http://example.com

# 注意：Nginx 不支持 CONNECT，所以无法代理 https 的网站。如果访问 https://www.baidu.com，Nginx access.log 日志如下：
"CONNECT www.baidu.com:443 HTTP/1.1" 400
```

## 反向代理

```bash
# 自定义日志格式
log_format main '$remote_addr - $http_x_forwarded_for - $http_x_real_ip - $remote_user [$time_local] "$request" '                                            
'$status $body_bytes_sent "$http_referer" '
'"$http_user_agent" "$request_time"';

server
{
    listen 80;
    server_name example.com;

    # API服务
    location /api/ {
        proxy_set_header   Host               $host;
        proxy_set_header   X-Real-IP          $remote_addr;
        proxy_set_header   X-Forwarded-Proto  $scheme;
        proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:8080;
        # 设置日志使用自定义格式
        access_log /home/wwwlogs/example.com.api.log main;
    }
    # 前台服务
    location / {
        proxy_pass http://192.168.33.10:8081;
        access_log /home/wwwlogs/example.com.web.log;
    }
}
```

## CORS 跨域配置

```bash
server
    {
        listen 8080;
        listen [::]:8080;

        server_name localhost;
        index index.php index.html index.htm default.html default.htm default.php;
        root  /data/wwwroot/espier-bloated/public;

        # 设置允许请求的源
        add_header Access-Control-Allow-Origin '*' always;
        # 自定义头信息
        add_header Access-Control-Allow-Headers "Content-Type, Access-Control-Allow-Headers, Authorization, X-Requested-With";
        add_header Access-Control-Expose-Headers "Authorization";
        # 设置允许请求的 HTTP 方法
        add_header Access-Control-Allow-Methods "DELETE, GET, HEAD, POST, PUT, OPTIONS, TRACE, PATCH";

        if ($request_method = OPTIONS ) {
            return 204;
        }

        include enable-php-pathinfo.conf;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
        {
            expires      30d;
        }

        location ~ .*\.(js|css)?$
        {
            expires      12h;
        }

        location ~ /.well-known {
            allow all;
        }

        location ~ /\.
        {
            deny all;
        }
    }
```

## 负载均衡

Nginx 负载均衡常用的几种方式：

1. 轮询（默认）

```bash
upstream backend {
    server 192.168.33.10;
    server 192.168.33.12;
}
```

2. weight（指定轮询比重）

```bash
upstream backend {
    server 192.168.33.10 weight=3;
    server 192.168.33.12 weight=7;
}
```

3. ip_hash

```bash
upstream backend {
    ip_hash;
    server 192.168.33.10;
    server 192.168.33.12;
}
```

示例：

```bash
upstream backend {
    server backend0.example.com       down; # down 表示当前 server 暂不参与负载
    server backend1.example.com       weight=5; # weight 默认为1。weight 越大，负载的权重就越大
    server backend2.example.com:8080; # 指定端口
    server unix:/tmp/backend3; # unix socket

    server backup3.example.com:8080   backup; # 其他所有非 backup 的机器忙碌或者为 down 时，请求 backup 机器。
    server backup4.example.com:8080   backup;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

