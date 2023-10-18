---
title: docker-compose搭建redis-cluster集群
categories:
- [Redis]
tags:
- Redis
---



## 脚本一键生成配置文件

在Redis Cluster集群中，每个节点的redis.conf文件通常需要保持一致。这是因为Redis Cluster的工作原理依赖于节点之间的一致性和相同的配置。以下是一些需要保持一致的重要配置项：

1. cluster-enabled：需要设置为yes，以启用Redis Cluster模式。
2. cluster-config-file：指定集群节点的配置文件路径，需要保持一致。
3. cluster-node-timeout：指定节点之间的超时时间，需要保持一致。
4. cluster-announce-ip和cluster-announce-port：如果在集群中的节点使用了静态IP和端口，则需要保持一致。

redis 配置文件模板 redis-cluster.tmpl 。

```shell
# redis端口
port ${port}
#redis 访问密码
requirepass p@ssw0rd
#redis 访问Master节点密码
masterauth p@ssw0rd
# 关闭保护模式
protected-mode no
# 启用Redis Cluster模式
cluster-enabled yes
# 指定集群节点的配置文件路径，各个节点需要保持一致
cluster-config-file nodes.conf
# 指定节点之间的超时时间
cluster-node-timeout 5000
# 集群节点IP
cluster-announce-ip 175.178.51.17
# 集群节点端口
cluster-announce-port ${port}
cluster-announce-bus-port 1${port}
# 开启 appendonly 备份模式
appendonly yes
# 每秒钟备份
appendfsync everysec
# 对aof文件进行压缩时，是否执行同步操作
no-appendfsync-on-rewrite no
# 当目前aof文件大小超过上一次重写时的aof文件大小的100%时会再次进行重写
auto-aof-rewrite-percentage 100
# 重写前AOF文件的大小最小值 默认 64mb
auto-aof-rewrite-min-size 64mb
# 开启混合持久化方式
aof-use-rdb-preamble yes

# 日志配置
# debug:会打印生成大量信息，适用于开发/测试阶段
# verbose:包含很多不太有用的信息，但是不像debug级别那么混乱
# notice:适度冗长，适用于生产环境
# warning:仅记录非常重要、关键的警告消息
loglevel notice
# 日志文件路径
logfile "/data/redis.log"
```

生成配置文件脚本 generate.sh 。

```shell
for no in `seq 6379 6381`; do \
  mkdir -p redis-${no}/conf \
  && port=${no} envsubst < redis-cluster.tmpl > redis-${no}/conf/redis.conf \
  && mkdir -p redis-${no}/data;\
done
```

>  cluster-announce-ip 和 端口情况根据自己机器的实际情况配置。
>
> 一般情况下都会创建 3 主 3 从的 redis 集群。因为服务器资源有限，只配置演示 3 主 0 从的 redis 集群。

生成完 reids cluster 配置文件后，可以通过 docker 的两种网络模式启动 redis cluster ，推荐使用 host 模式。

## 自建bridge网络模式

```yaml
version: "3.8"

# 定义服务，可以多个
services:
  redis-cluster:
    image: redis:7.0.6
    networks:
      redis:
        ipv4_address: 172.20.0.2
    command: redis-cli --cluster create 172.20.0.11:6379 172.20.0.12:6379 172.20.0.13:6379 --cluster-yes -a p@ssw0rd
    depends_on:
      - redis-1
      - redis-2
      - redis-3

  redis-1: # 服务名称
    image: redis:7.0.6 # 创建容器时所需的镜像
    container_name: redis-1 # 容器名称
    restart: "no" # 容器总是重新启动
    networks:
      redis:
        ipv4_address: 172.20.0.11
    ports:
      - "6379:6379"
      - "16379:16379"
    volumes: # 数据卷，目录挂载
      - ./etc_rc.local:/etc/rc.local
      - ./redis-1/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-1/data:/data
    command: redis-server /etc/redis/redis.conf # 覆盖容器启动后默认执行的命令
  redis-2:
    image: redis:7.0.6
    container_name: redis-2
    networks:
      redis:
        ipv4_address: 172.20.0.12
    ports:
      - "6380:6379"
      - "16380:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-2/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-2/data:/data
    command: redis-server /etc/redis/redis.conf
  redis-3:
    image: redis:7.0.6
    container_name: redis-3
    networks:
      redis:
        ipv4_address: 172.20.0.13
    ports:
      - "6381:6379"
      - "16381:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-3/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-3/data:/data
    command: redis-server /etc/redis/redis.conf

# 自动创建网络，并手动指定IP网段
networks:
  redis:
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

> 自动创建网络，并手动指定IP网段的方式启动的 redis cluster 集群，是无法通过宿主机 IP 进行连接的，因为各个节点下的 node.conf 文件记录的是各自节点的容器 IP 。

## host网络模式（推荐）

```yaml
version: "3.8"

# 定义服务，可以多个
services:
  redis-cluster:
    image: redis:7.0.6
    network_mode: "host"
    volumes:
      - ./redis-cluster/conf/redis.conf:/etc/redis/redis.conf
    command: redis-cli --cluster create 175.178.51.17:6379 175.178.51.17:6380 175.178.51.17:6381 --cluster-yes -a p@ssw0rd
    depends_on:
      - redis-6379
      - redis-6380
      - redis-6381

  redis-6379: # 服务名称
    image: redis:7.0.6 # 创建容器时所需的镜像
    container_name: redis-6379 # 容器名称
    restart: "no" # 容器总是重新启动
    network_mode: "host"
    volumes: # 数据卷，目录挂载
      - ./etc_rc.local:/etc/rc.local
      - ./redis-6379/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-6379/data:/data
    command: redis-server /etc/redis/redis.conf # 覆盖容器启动后默认执行的命令
  redis-6380:
    image: redis:7.0.6
    container_name: redis-6380
    network_mode: "host"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-6380/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-6380/data:/data
    command: redis-server /etc/redis/redis.conf
  redis-6381:
    image: redis:7.0.6
    container_name: redis-6381
    network_mode: "host"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-6381/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-6381/data:/data
    command: redis-server /etc/redis/redis.conf
```

通过 host 模式启动的 redis cluster 集群模式，各个节点使用宿主机 IP ，node.conf 文件中记录的就是宿主机的 IP ，所以可以通过宿主机 IP 连接。

node.conf 文件内容

```shell
Last login: Mon Jul 10 17:11:08 2023 from 220.198.245.66
5a17fa1735f401dc0781670c052aadd6c9735f4d 175.178.51.17:6379@16379 myself,master - 0 1685019165000 1 connected 0-5460
943f252f84359acd2057f968d7afa61d2c3fb06e 175.178.51.17:6381@16381 master - 0 1685019168337 3 connected 10923-16383
7d136cf9a321d17b2ee732c12ffbe2dcf97a246a 175.178.51.17:6380@16380 master - 0 1685019169339 2 connected 5461-10922
vars currentEpoch 3 lastVoteEpoch 0
```

## 参考

[docker-compose安装redis-cluster集群](https://www.cnblogs.com/devhg/p/15841237.html)

