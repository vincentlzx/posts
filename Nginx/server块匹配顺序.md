---
title: Nginx server块匹配顺序
categories:
- [Nginx]
tags:
- Nginx
---



在 Nginx 中，`server_name` 的匹配顺序遵循以下规则：

1. 精确匹配：如果请求的主机名与某个 `server` 块的 `server_name` 完全匹配，则使用该 `server` 块处理请求。
2. 通配符前缀匹配：如果请求的主机名与某个 `server` 块的 `server_name` 以通配符（`*.`）开头的部分匹配，则使用该 `server` 块处理请求。通配符匹配可以用于处理子域名的请求。
3. 长度最长匹配：如果有多个 `server` 块的 `server_name` 部分匹配请求的主机名，则使用最长匹配的 `server` 块处理请求。这意味着 Nginx 会选择最能精确匹配请求主机名的 `server` 块。
5. 默认服务器：如果以上规则都不匹配，则使用配置文件中定义的第一个 `server` 块作为默认服务器块来处理请求。

```nginx
server {
    listen 80;
    server_name example.com;
    ...
}

server {
    listen 80;
    server_name subdomain.example.com;
    ...
}

server {
    listen 80;
    server_name *.example.com;
    ...
}

server {
    listen 80 default_server;
    server_name _;
    ...
}

```

在上述示例中：

- 如果请求的主机名是 `example.com`，将使用第一个 `server` 块处理请求。
- 如果请求的主机名是 `subdomain.example.com`，将使用第二个 `server` 块处理请求。
- 如果请求的主机名是 `abc.example.com` 或其他以 `.example.com` 结尾的子域名，将使用第三个 `server` 块处理请求。
- 如果请求的主机名与上述情况都不匹配，将使用最后一个 `server` 块作为默认服务器块处理请求。

> 为什么不是第一个 server 块作为默认，因为下划线在 server_name 中表示默认服务。

