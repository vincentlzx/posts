---
title: Nginx配置https证书
categories:
- [Nginx]
tags:
- Nginx
---



## 腾讯云证书

- 腾讯云免费证书只支持单个域名；如配置证书域名为 liuzx.com.cn ，若访问 blog.liuzx.com.cn 会有证书无效的错误。
- 把 http 的域名请求转成 https 后，http server 块中的 location 配置失效（其他配置应该也会失效），需要在 https server 块中重新配置。

```nginx
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
  worker_connections 1024;
}

http {
  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
  '$status $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';

  access_log /var/log/nginx/access.log main;

  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 65;
  types_hash_max_size 4096;

  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  # 浏览器默认协议是http
  server {
    listen 80;
    listen [::]:80;
    server_name liuzx.com.cn;
    location / {
      proxy_pass http://localhost:3000;
    }
    # 把http的域名请求转成https，location和其他配置将不会生效
    return 301 https://$host$request_uri; 
  }

  # Load modular configuration files from the /etc/nginx/conf.d directory.
  # See http://nginx.org/en/docs/ngx_core_module.html#include
  # for more information.
  include /etc/nginx/conf.d/*.conf;

  # Settings for a TLS enabled server.
  server {
    #SSL 默认访问端口号为 443
    listen 443 ssl;
    #请填写绑定证书的域名（腾讯云免费域名只支持绑定单个域名）
    server_name liuzx.com.cn;
    #请填写证书文件的相对路径或绝对路径
    ssl_certificate liuzx.com.cn_bundle.crt;
    #请填写私钥文件的相对路径或绝对路径
    ssl_certificate_key liuzx.com.cn.key;
    ssl_session_timeout 5m;
    #请按照以下协议配置
    ssl_protocols TLSv1.2 TLSv1.3;
    #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    # 需要在https块中重新配置
    location / {
      proxy_pass http://localhost:3000;
    }
  }
}
```

