---
title: Nginx location匹配规则
categories:
- [Nginx]
tags:
- Nginx
---



## Nginx配置

```nginx
server {
  listen 80;
  server_name test.liuzx.com.cn;
  return 301 https://$host$request_uri;
}

server {
  #SSL 默认访问端口号为 443
  listen 443 ssl;
  #请填写绑定证书的域名（腾讯云免费域名只支持绑定单个域名）
  server_name test.liuzx.com.cn;
  #请填写证书文件的相对路径或绝对路径
  ssl_certificate /etc/nginx/conf.d/test.liuzx.com.cn/test.liuzx.com.cn_bundle.crt;
  #请填写私钥文件的相对路径或绝对路径
  ssl_certificate_key /etc/nginx/conf.d/test.liuzx.com.cn/test.liuzx.com.cn.key;
  ssl_session_timeout 5m;
  #请按照以下协议配置
  ssl_protocols TLSv1.2 TLSv1.3;
  #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
  ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
  ssl_prefer_server_ciphers on;

  location = /aaa/ {
    add_header Content-Type text/plain;
    return 200 "location = /aaa/";
  }

  location ^~ /aaa/bbb/ {
    add_header Content-Type text/plain;
    return 200 "location ^~ /aaa/bbb/";
  }
  location /aaa/bbb/ccc/ {
    add_header Content-Type text/plain;
    return 200 "location /aaa/bbb/ccc/";
  }
  location ~ /aaa/bbb/ {
    add_header Content-Type text/plain;
    return 200 "location ~ /aaa/bbb/";
  }

  location /ddd/ {
    add_header Content-Type text/plain;
    return 200 "location /ddd/";
  }
  location /ddd/eee/ {
    add_header Content-Type text/plain;
    return 200 "location /ddd/eee/";
  }

  location ~ /ddd/eee/ {
    add_header Content-Type text/plain;
    return 200 "location ~ /ddd/eee/";
  }
  location ~ /ddd/eee/fff/ {
    add_header Content-Type text/plain;
    return 200 "location ~ /ddd/eee/fff/";
  }

  location ~* /aaa/bbb/ {
    add_header Content-Type text/plain;
    return 200 "location ~* /aaa/bbb/ccc/";
  }
}
```

## 匹配顺序

1. **先普通，再正则**。
2. 普通location之间的匹配顺序：**按最大前缀匹配**。
3. 正则location之间的匹配顺序：**按配置文件中的物理顺序匹配，只要匹配到一条正则，就不再考虑后面的**。
4. 普通location先匹配，匹配到了结果，只是一个临时结果；会继续正则location的匹配。
   1. 如果匹配到正则，则用匹配到的正则结果。
   2. 如果没有匹配到正则，则继续用普通匹配的那个结果。

> 匹配完普通location，还要继续匹配正则location。但是，也可以告诉nginx，匹配到了普通location，就不要再搜索匹配正则location了，通过在普通location前面加上`^~`符号，`^`表示非，`~`表示正则，`^~`就是表示不要继续匹配正则。除了`^~`，`=`也可阻止nginx继续匹配正则，区别在于`^~`依然遵循最大前缀匹配规则，而`=`是严格匹配

## 精准匹配

```nginx
location = /aaa/ {
  alias /etc/nginx/conf.d/test.liuzx.com.cn/;
  index location1.txt;
}
```

请求 https://test.liuzx.com.cn/aaa/ 返回内容 location = /aaa/ 。

## 精准前缀匹配

```nginx
location ^~ /aaa/bbb/ {
  alias /etc/nginx/conf.d/test.liuzx.com.cn/;
  index location1.txt;
}
```

- 请求 https://test.liuzx.com.cn/aaa/bbb/ 返回内容 location ^~ /aaa/bbb/ 。

- 请求 https://test.liuzx.com.cn/aaa/bbb/ccc 返回内容 location ^~ /aaa/bbb/ 。

- 请求 https://test.liuzx.com.cn/aaa/bbb/ccc/ 返回内容 location ~ /aaa/bbb/ 。

  !>最后一个请求为什么不是精准前缀匹配 location ^~ /aaa/bbb/ ，命中后直接返回，而是正则匹配 location ~ /aaa/bbb/ ？

  **因为无论是精准前缀匹配还是普通前缀匹配，两者都遵守遵循最大前缀匹配规则**。根据最大前缀匹配规则，https://test.liuzx.com.cn/aaa/bbb/ccc/ 匹配的是普通前缀匹配 location /aaa/bbb/ccc/ ，但这只是一个临时结果，往下进行正则 location 匹配，匹配到 location ~ /aaa/bbb/ ，正则匹配不遵循最大前缀匹配规则，而是按照配置文件中的顺序匹配，所以返回内容是 location ~ /aaa/bbb/ 。

## 普通前缀匹配

```nginx
location /aaa/bbb/ccc/ {
  add_header Content-Type text/plain;
  return 200 "location /aaa/bbb/ccc/";
}
location /ddd/ {
  add_header Content-Type text/plain;
  return 200 "location /ddd/";
}
location /ddd/eee/ {
  add_header Content-Type text/plain;
  return 200 "location /ddd/eee/";
}
```

- 请求 https://test.liuzx.com.cn/aaa/bbb/ccc/ 返回内容 location ~ /aaa/bbb/ 。上面已解释。
- 请求 https://test.liuzx.com.cn/ddd/ ，返回内容 location /ddd/ 。
- 请求 https://test.liuzx.com.cn/ddd/eee/ ，普通匹配 location /ddd/eee 作为临时结果，往后进行正则匹配 location ~ /ddd/eee/ ，所以返回内容 location ~ /ddd/eee/ 。

## 区分大小写正则匹配

 ```nginx
 location ~ /aaa/bbb/ {
   add_header Content-Type text/plain;
   return 200 "location ~ /aaa/bbb/";
 }
 location ~ /ddd/eee/ {
   add_header Content-Type text/plain;
   return 200 "location ~ /ddd/eee/";
 }
 location ~ /ddd/eee/fff/ {
   add_header Content-Type text/plain;
   return 200 "location ~ /ddd/eee/fff/";
 }
 ```

- 请求 https://test.liuzx.com.cn/ddd/eee/fff/ ，按照配置文件中的物理顺序正则匹配到 location ~ /ddd/eee/ ，直接返回内容 location ~ /ddd/eee/ 。

## 不区分大小写正则匹配

```nginx
location ~* /aaa/bbb/ {
  add_header Content-Type text/plain;
  return 200 "location ~* /aaa/bbb/ccc/";
}
```

请求 https://test.liuzx.com.cn/aAa/bbb/zzz ，返回内容 location ~* /aaa/bbb/ccc/ 。

## 参考

https://segmentfault.com/a/1190000019138014
