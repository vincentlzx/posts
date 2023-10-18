---
title: Frp实现内网穿透
categories:
- [内网穿透]
tags:
- Frp
- 内网穿透
---

## 场景

现有一台公网服务器和域名 frp.liuzx.com.cn ，其中 80 端口已被 Nginx 监听占用。

假设本机内网有个接口 `http://localhost:8080/test` ，能用以下公网地址访问：

- http://frp.liuzx.com.cn/test（直接域名不带端口访问）
- https://frp.liuzx.com.cn/test（https 访问）
- http://frp.liuzx.com.cn:8080/test（域名 + 端口访问）
- http://ip:8080/test（IP + 端口访问）

> 不能停止 Nginx 服务，意味着 Frp 与 Nginx 需要同时使用 80 端口。主要思路就是通过 Nginx 转发到 Frp 监听的 HTTP 请求端口。

## Frp实现内网穿透

Frp（Fast Reverse Proxy）是一种用于实现内网穿透的工具，它允许您通过公网访问位于内网的服务器或设备。以下是使用 Frp 实现内网穿透的基本步骤：

1. **安装 Frp**：

   首先，您需要在公网服务器和内网设备上分别安装 Frp。目前可以在 Github 的 [Release](https://github.com/fatedier/frp/releases) 页面中下载到最新版本的客户端和服务端二进制文件，所有文件被打包在一个压缩包中。下载并解压适用于服务端和客户端操作系统的 Frp 二进制文件。

   > 其中 fprc 和 frpc.ini 是客户端需要用到的文件；fprs 和 frps.ini 是服务端需要用到的文件

2. **配置 Frp 服务器端**：

   在公网服务器上配置 Frp 服务器端，以监听指定的端口，并将请求转发到内网设备。编辑 Frp 服务器的配置文件 `frps.ini`。

   ```ini
   [common]
   bind_port = 7000
   vhost_http_port = 780
   vhost_https_port = 7443
   ```

   - `bind_port`：7000，用于 Frp 客户端与 Frp 服务器端之间的通信。
   - `vhost_http_port`：780，用于处理 HTTP 流量。
   - `vhost_https_port`：7443，用于处理 HTTPS 流量。

   > vhost_http_port = 80 和 vhost_https_port = 443 使用默认端口 80 和 443 可以实现不加端口访问

3. **配置 Frp 客户端端**：

   在内网设备上配置 Frp 客户端，以将内网设备的服务暴露给公网。编辑 Frp 客户端的配置文件 `frpc.ini`。

   ```ini
   [common]
   server_addr = 143.167.51.17
   server_port = 7000
   
   [web]
   type = http
   local_port = 8080
   custom_domains = frp.liuzx.com.cn
   
   [web2]
   type = http
   local_port = 8080
   custom_domains = 143.167.51.17
   
   [web3]
   type = https
   custom_domains = frp.liuzx.com.cn
   
   plugin = https2http
   plugin_local_addr = 127.0.0.1:8080
   
   # HTTPS 证书相关的配置
   plugin_crt_path = /Users/vincent/liuzx.com.cn/frp.liuzx.com.cn_nginx/frp.liuzx.com.cn_bundle.crt
   plugin_key_path = /Users/vincent/liuzx.com.cn/frp.liuzx.com.cn_nginx/frp.liuzx.com.cn.key
   plugin_host_header_rewrite = 127.0.0.1
   plugin_header_X-From-Where = frp
   ```

   - web ：`local_port` 为本地机器上 Web 服务监听的端口，绑定自定义域名为 `frp.liuzx.com.cn`。
   - web2 ：`local_port` 为本地机器上 Web 服务监听的端口，绑定服务器 IP 。
   - web3 ：`plugin` 配置为 `https2http`，表示使用 HTTPS 到 HTTP 的插件进行转发。`plugin_local_addr` 是本地内网服务的地址。

4. **服务端 Nginx 配置：**

   主要思路就是通过 Nginx 转发 Frp 服务监听的端口。

   ```nginx
   server {
     listen 80;
     server_name frp.liuzx.com.cn;
     location / {
       proxy_pass http://127.0.0.1:780;
       proxy_redirect https://$host/ https://$http_host/;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_ssl_server_name on;
       proxy_set_header Host $host;
     }
   }
   server {
     listen 8080;
     server_name frp.liuzx.com.cn;
     location / {
       proxy_pass http://127.0.0.1:780;
       proxy_redirect https://$host/ https://$http_host/;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_ssl_server_name on;
       proxy_set_header Host $host;
     }
   }
   server {
     listen 8080;
     location / {
       proxy_pass http://127.0.0.1:780;
       proxy_redirect https://$host/ https://$http_host/;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_ssl_server_name on;
       proxy_set_header Host $host;
     }
   }
   server {
     #SSL 默认访问端口号为 443
     listen 443 ssl;
     #请填写绑定证书的域名（腾讯云免费域名只支持绑定单个域名）
     server_name frp.liuzx.com.cn;
     #请填写证书文件的相对路径或绝对路径
     ssl_certificate /etc/nginx/conf.d/frp.liuzx.com.cn/frp.liuzx.com.cn_bundle.crt;
     #请填写私钥文件的相对路径或绝对路径
     ssl_certificate_key /etc/nginx/conf.d/frp.liuzx.com.cn/frp.liuzx.com.cn.key;
     ssl_session_timeout 5m;
     #请按照以下协议配置
     ssl_protocols TLSv1.2 TLSv1.3;
     #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
     ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
     ssl_prefer_server_ciphers on;
     location / {
       proxy_pass http://127.0.0.1:780;
       proxy_redirect https://$host/ https://$http_host/;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_ssl_server_name on;
       proxy_set_header Host $host;
     }
   }
   ```

5. 通过 `./frps -c ./frps.ini` 启动服务端，再通过 `./frpc -c ./frpc.ini` 启动客户端。

测试可以穿透到内网的访问地址：

1. 直接使用 Frp 监听的端口，不经过 Nginx 转发：
   - http://frp.liuzx.com.cn:780/test
   - https://frp.liuzx.com.cn:7443/test
   - http://175.178.51.17:780/test
2. 通过 Nginx 转发：
   - http://frp.liuzx.com.cn/test
   - https://frp.liuzx.com.cn/test
   - http://frp.liuzx.com.cn:8080/test
   - http://175.178.51.17:8080/test

## 参考

Frp 官方文档：https://gofrp.org/docs/

Frp 和 Nginx 共用端口：https://www.cnblogs.com/hahaha111122222/p/14544367.html