---
title: Docker Compose编排常用参数
categories:
- [Docker, Docker Compose]
tags:
- Docker
- Docker Compose
---



## 常用参数

### build

服务除了可以基于指定的镜像，还可以基于一份 Dockerfile，在使用 up 启动之时执行构建任务，这个构建标签就是 build，它可以指定 Dockerfile 所在文件夹的路径。Compose 将会利用它自动构建这个镜像，然后使用这个镜像启动服务容器。

```yaml
services:
  cloudplatform-mysql:
    build:
      context: ./
      dockerfile: ./db/Dockerfile
```

- `context`：表示构建上下文的路径。

  >COPY 和 ADD 命令不能拷贝上下文之外的本地文件。
  >
  >假设 dockerfile 文件有如下命令行：
  >
  >COPY ./db/1schema.sql /docker-entrypoint-initdb.d
  >
  >表示将上下文路径下的 db/1schema.sql 文件（必须在 build.context 指定的路径下）拷贝到容器 /docker-entrypoint-initdb.d 路径下。

- `dockerfile`：指定用于构建容器的Dockerfile的路径

### image

用于指定现有Docker镜像的名称。

### container_name

假设有 docker-compose.yml 文件：

```yaml
services:
  es01:
    image: elasticsearch:7.17.10
    container_name: "es02"
```

服务名称为 es01 ，容器名称为 es02 。

> 同一台服务器，容器名称必须唯一，但是服务名称只需要保证同一个docker-compose中唯一即可。但是需要确保两个docker-compose文件在不同目录中。

- 服务名称在 Docker Compose 中用于标识和管理容器配置以及服务之间的连接等。
- 容器名称是容器实例的唯一标识符。





## 参考

[docker-compose编排参数详解 ](https://www.cnblogs.com/wutao666/p/11332186.html)