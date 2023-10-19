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

- `dockerfile`：指定用于构建容器的Dockerfile的路径。

### image

用于指定现有Docker镜像的名称。

### container_name

假设有 docker-compose.yml 文件：

```yaml
services:
  elasticsearch:
    image: elasticsearch:7.17.10
    container_name: "es01"
```

服务名称为 elasticsearch，容器名称为 es01 。

> 同一台服务器，容器名称必须唯一，但是服务名称只需要保证同一个docker-compose中唯一即可。但是需要确保两个docker-compose文件在不同目录中。

- 服务名称在 Docker Compose 中用于标识和管理容器配置以及服务之间的连接等。
- 容器名称是容器实例的唯一标识符。

### command

覆盖默认的 command 命令。

```yaml
command: bundle exec thin -p 3000
```

command 可以是一个列表：

```yaml
command: ["bundle", "exec", "thin", "-p", "3000"]
```

### depends_on

容器中服务之间的依赖关系。

```yaml
version: "3.8"
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```

> `depends_on` does not wait for `db` and `redis` to be "ready" before starting `web` - only until they have been started. If you need to wait for a service to be ready, see [Controlling startup order](https://docs.docker.com/compose/startup-order/) for more on this problem and strategies for solving it.

### entrypoint

覆盖默认的 entrypoint 。

```yaml
entrypoint: /code/entrypoint.sh
```

entrypoint 可以是一个列表：

```yaml
entrypoint: ["php", "-d", "memory_limit=-1", "vendor/bin/phpunit"]
```

> Setting `entrypoint` both overrides any default entrypoint set on the service's image with the `ENTRYPOINT` Dockerfile instruction, *and* clears out any default command on the image - meaning that if there's a `CMD` instruction in the Dockerfile, it is ignored.

### env_file

从文件中添加环境变量，可以是单个或多个文件。

如果使用 docker-compose -f FILE 指定了 Compose 文件，则 env_file 中的路径相对于该文件所在的目录。

在 environment 配置中声明的环境变量会覆盖这些值——即使这些值是空的或未定义的，这也成立。

```yaml
env_file: .env
```

```yaml
env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/runtime_opts.env
```

env_file 文件中的每一行都采用 VAR=VAL 格式。以 # 开头的行被视为注释并被忽略。空行也会被忽略。

```yaml
# Set Rails/Rack environment
RACK_ENV=development
```

Compose 还可以识别内联注释，例如：

```yaml
MY_VAR = value # this is a comment
```

为了避免将“#”解释为内联注释，请使用引号：

```yaml
MY_VAR = "All the # inside are taken as part of the value"
```

> 如果您的服务指定了构建选项，则环境文件中定义的变量在构建过程中不会自动可见。使用 build 的 args 子选项来定义构建时环境变量。

列表中文件的顺序对于确定分配给多次出现的变量的值非常重要。列表中的文件从上到下处理。

```yaml
services:
  some-service:
    env_file:
      - a.env
      - b.env
```

```properties
# a.env
VAR=1
```

```properties
# b.env
VAR=hello
```

$VAR 的值是 hello 。

### environment

添加环境变量。可以使用数组或字典。任何布尔值（true、false、yes、no）都需要用引号引起来，以确保它们不会被 YML 解析器转换为 True 或 False。

```yaml
environment:
  RACK_ENV: development
  SHOW: 'true'
  SESSION_SECRET:
```

```yaml
environment:
  - RACK_ENV=development
  - SHOW=true
  - SESSION_SECRET
```

> 如果您的服务指定了构建选项，则环境中定义的变量在构建过程中不会自动可见。使用 build 的 args 子选项来定义构建时环境变量。

### expose

公开端口而不将它们发布到主机 - 它们只能由链接的服务访问。只能指定内部端口。

```yaml
expose:
  - "3000"
  - "8000"
```

### external_links

链接到在此 docker-compose.yml 外部甚至 Compose 外部启动的容器，特别是对于提供共享或公共服务的容器。在指定容器名称和链接别名 (CONTAINER:ALIAS) 时，external_links 遵循与旧版选项链接类似的语义。

```yaml
external_links:
  - redis_1
  - project_db_1:mysql
  - project_db_1:postgresql
```

```yaml
外部创建的容器必须至少连接到与链接到它们的服务相同的网络之一。建议使用 networks 配置。 	
```

### image

指定启动容器的镜像。

如果 image 不存在，Compose 会尝试从 Docker Hub 中拉取它。除非指定 build 配置，在这种情况下，会使用指定的选项构建它并使用指定的标签对其进行标记。

### network_mode

网络模式。使用与 docker run --network 参数相同的值，加上特殊形式 service:[service name]。

```yaml
network_mode: "bridge"

network_mode: "host"

network_mode: "none"

network_mode: "service:[service name]"

network_mode: "container:[container name/id]"
```

### networks

加入网络。

```yaml
services:
  some-service:
    networks:
     - some-network
     - other-network
```

#### aliases

该服务在网络上的别名（备用主机名）。同一网络上的其他容器可以使用服务名称或该别名来连接该服务。

由于别名是网络范围的，因此同一服务在不同网络上可以有不同的别名。

```yaml
services:
  some-service:
    networks:
      some-network:
        aliases:
          - alias1
          - alias3
      other-network:
        aliases:
          - alias2
```

示例：

```yaml
version: "3.8"

services:
  web:
    image: "nginx:alpine"
    networks:
      - new

  worker:
    image: "my-worker-image:latest"
    networks:
      - legacy

  db:
    image: mysql
    networks:
      new:
        aliases:
          - database
      legacy:
        aliases:
          - mysql

networks:
  new:
  legacy:
```

#### ipv4_address

加入网络时为该服务的容器指定静态 IP 地址。

示例：

```yaml
version: "3.8"

services:
  app:
    image: nginx:alpine
    networks:
      app_net:
        ipv4_address: 172.16.238.10

networks:
  app_net:
    ipam:
      driver: default
      config:
        - subnet: "172.16.238.0/24"
```

> The corresponding network configuration in the [top-level networks section](https://docs.docker.com/compose/compose-file/compose-file-v3/#network-configuration-reference) must have an `ipam` block with subnet configurations covering each static address.

### prots

暴露端口。

> 与 host 网络模式冲突。

可以使用 HOST:CONTAINER 的方式指定端口，也可以指定容器端口（选择临时主机端口），宿主机会随机映射端口。

```yaml
ports:
  - "3000"
  - "3000-3005"
  - "8000:8000"
  - "9090-9091:8080-8081"
  - "49100:22"
  - "127.0.0.1:8001:8001"
  - "127.0.0.1:5000-5010:5000-5010"
  - "127.0.0.1::5000"
  - "6060:6060/udp"
  - "12400-12500:1240"
```

> 当使用 HOST:CONTAINER 格式来映射端口时，如果使用的容器端口小于60可能会得到错误得结果，因为YAML 将会解析 xx:yy 这种数字格式为 60 进制，所以建议采用字符串格式。

### restart

默认值为 no ，即在任何情况下都不会重新启动容器；当值为 always 时，容器总是重新启动；当值为 on-failure 时，当出现 on-failure 报错容器退出时，容器重新启动。

```yaml
restart: "no"
restart: always
restart: on-failure
restart: unless-stopped
```

### volumes

挂载宿主机路径或 named volumes 。

```yaml
version: "3.8"
services:
  web:
    image: nginx:alpine
    volumes:
      - type: volume
        source: mydata  # 挂载named volumes
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

  db:
    image: postgres:latest
    volumes:
      - "/var/run/postgres/postgres.sock:/var/run/postgres/postgres.sock"
      - "dbdata:/var/lib/postgresql/data"  # 挂载named volumes

# 在Docker中创建named volumes
volumes:
  mydata:
  dbdata:
```

> named volumes 是指由 docker volume create 创建的 Docker named volumes ，并不是值宿主机数据卷路径。

#### 简短语法

简短语法使用通用 [SOURCE:]TARGET[:MODE] 格式，其中 SOURCE 可以是主机路径或卷名称。 TARGET 是容器路径。MODE 是 ro 表示只读，rw 表示读写（默认）。

可以使用宿主机的相对路径，该路径相对于正在使用的 Compose 配置文件的目录进行扩展。

```yaml
volumes:
  # Just specify a path and let the Engine create a volume
  - /var/lib/mysql

  # Specify an absolute path mapping
  - /opt/data:/var/lib/mysql

  # Path on the host, relative to the Compose file
  - ./cache:/tmp/cache

  # User-relative path
  - ~/configs:/etc/configs/:ro

  # Named volume
  - datavolume:/var/lib/mysql
```

## 什么是Docker named volumes

Docker命名的卷是一种在Docker容器中保存数据的方法。它们允许你创建一个命名的卷，并将其挂载到容器上，这样即使容器被停止或删除，存储在卷中的数据也会被保留下来。

要在Docker中创建一个命名的卷，你可以使用docker volume create命令。

## 参考

服务配置参考官方文档：https://docs.docker.com/compose/compose-file/compose-file-v3/

docker-compose编排参数详解：https://www.cnblogs.com/wutao666/p/11332186.html

Docker named volumes 解释：https://geek-docs.com/docker/docker-tutorials/t_docker-named-volumes-vs-doc-data-only-containers.html