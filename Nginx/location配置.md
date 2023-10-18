---
title: Nginx location配置
categories:
- [Nginx]
tags:
- Nginx
---



## location最后带不带斜杠的区别

```nginx
location /swagger {
  proxy_pass http://localhost:9262;
}
```

不带斜杠能匹配：

http://kangmei-hospital.haikevr.com/swagger-ui/index.html（/swagger/ 就不能匹配了）

```nginx
location /api/ {
  proxy_pass http://localhost:9262;
}
```

带斜杠必须是 api 路径：

http://kangmei-hospital.haikevr.com/api/login

http://kangmei-hospital.haikevr.com/api/test

## proxy_pass最后是否带斜杠的区别

```nginx
server {
  listen 80;
  server_name kangmei-hospital.haikevr.com;
  root  /disk/data/html/kangmei-hospital/build;
  index index.html;

  location /api/ {
    proxy_pass http://localhost:9262;  # 追加包括api的路径
  }
}
```

访问 URl ：http://kangmei-hospital.haikevr.com/api/login

转发的实际地址：http://localhost:9262/api/login

不带斜杠 nginx 会把匹配到的内容追加到 proxy_pass 地址后面

```nginx
server {
  listen 80;
  server_name kangmei-hospital.haikevr.com;
  root  /disk/data/html/kangmei-hospital/build;
  index index.html;

  location /api/ {
    proxy_pass http://localhost:9262/;  # 追加不包括api的路径
  }
}
```

访问 URl ：http://kangmei-hospital.haikevr.com/api/login

转发的实际地址：http://localhost:9262/login

带斜杠 nginx 不会把匹配到的内容追加到 proxy_pass 地址后面

## alias和root的区别

alias 是一个目录别名的定义，root 则是最上层目录的定义。

```nginx
location ^~ /123/abc/ {
  root /data/www;  # 最后的斜杠可有可无；追加匹配路径
}
```

访问 URl ：http://kangmei-hospital.haikevr.com/123/abc/logo.png

实际：/data/www/123/abc/logo.png

```nginx
location ^~ /123/abc/ {
  alias /data/www/;  # 最后必须是斜杠；不追加匹配路径
}
```

访问 URl ：http://kangmei-hospital.haikevr.com/123/abc/logo.png

实际：/data/www/logo.png