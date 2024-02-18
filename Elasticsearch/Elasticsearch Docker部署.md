---
title: Elasticsearch7.17.7部署
categories:
- [Docker, Docker Compose, Elasticsearch]
tags:
- Docker
- Docker Compose
- Elasticsearch
---



## Elasticsearch7.17.7单机部署

编写 docker-compose.yaml 文件

```java
version: '3.8'

services:
  es01:
    image: elasticsearch:7.17.7
    container_name: es01
    ports:
      - "9200:9200"
    volumes:
      - ./elasticsearch/data:/usr/share/elasticsearch/data
      - ./elasticsearch/logs:/usr/share/elasticsearch/logs
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./elasticsearch/plugins:/usr/share/elasticsearch/plugins
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
  kibana:
    image: kibana:7.17.7
    ports:
      - "5601:5601"
    container_name: kibana
    depends_on:
      - es01
    volumes:
      - ./kibana/kibana.yml:/usr/share/kibana/config/kibana.yml
```

编写 elasticsearch.yml 文件

> 参考章节：https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docker.html#docker-config-bind-mount
>
> If you bind-mount a custom `elasticsearch.yml` file, ensure it includes the `network.host: 0.0.0.0` setting. This setting ensures the node is reachable for HTTP and transport traffic, provided its ports are exposed. The Docker image’s built-in `elasticsearch.yml` file includes this setting by default.

```yaml
http.host: 0.0.0.0
```

编写 kibana.yml 文件

```yaml
server.name: kibana
server.host: "0.0.0.0"
elasticsearch.hosts: http://es01:9200
xpack.monitoring.ui.container.elasticsearch.enabled: true
i18n.locale: zh-CN
```

## Elasticsearch7.17.16集群部署

编写 docker-compose.yaml 文件

```yaml
version: '3.8'
services:
  es01:
    image: elasticsearch:7.17.18
    container_name: es01
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      # - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es01/data:/usr/share/elasticsearch/data
      - ./es01/logs:/usr/share/elasticsearch/logs
      - ./es01/plugins:/usr/share/elasticsearch/plugins
      - ./es01/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
  es02:
    image: elasticsearch:7.17.18
    container_name: es02
    ports:
      - "9201:9200"
      - "9301:9300"
    environment:
      # - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es02/data:/usr/share/elasticsearch/data
      - ./es02/logs:/usr/share/elasticsearch/logs
      - ./es02/plugins:/usr/share/elasticsearch/plugins
      - ./es02/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
  es03:
    image: elasticsearch:7.17.18
    container_name: es03
    ports:
      - "9202:9200"
      - "9302:9300"
    environment:
      # - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es03/data:/usr/share/elasticsearch/data
      - ./es03/logs:/usr/share/elasticsearch/logs
      - ./es03/plugins:/usr/share/elasticsearch/plugins
      - ./es03/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
```

各个节点编写 elasticsearch.yml 配置文件

```yaml
# es01
# 数据存储地址
path.data: /usr/share/elasticsearch/data
# 日志存储地址
path.logs: /usr/share/elasticsearch/logs
# 集群名称
cluster.name: es-docker-cluster
# 节点名称
node.name: es01
# 0.0.0.0，表示对外开放，如对特定ip开放则改为指定ip
network.host: 0.0.0.0
# 节点列表
discovery.seed_hosts: ["10.147.17.206:9301","10.147.17.206:9302"]
# 初始化时master节点的选举列表
cluster.initial_master_nodes: ["es01","es02","es03"]
# 对外提供服务的端口
http.port: 9200
# 内部服务端口
transport.port: 9300    
# 跨域支持
http.cors.enabled: true
# 跨域访问允许的域名地址（正则）
http.cors.allow-origin: /.*/

# es02
# 数据存储地址
path.data: /usr/share/elasticsearch/data
# 日志存储地址
path.logs: /usr/share/elasticsearch/logs
# 集群名称
cluster.name: es-docker-cluster
# 节点名称
node.name: es02
# 0.0.0.0，表示对外开放，如对特定ip开放则改为指定ip
network.host: 0.0.0.0
# 节点列表
discovery.seed_hosts: ["10.147.17.206:9300","10.147.17.206:9302"]
# 初始化时master节点的选举列表
cluster.initial_master_nodes: ["es01","es02","es03"]
# 对外提供服务的端口
http.port: 9201
# 内部服务端口
transport.port: 9301  
# 跨域支持
http.cors.enabled: true
# 跨域访问允许的域名地址（正则）
http.cors.allow-origin: /.*/

# es03
# 数据存储地址
path.data: /usr/share/elasticsearch/data
# 日志存储地址
path.logs: /usr/share/elasticsearch/logs
# 集群名称
cluster.name: es-docker-cluster
# 节点名称
node.name: es03
# 0.0.0.0，表示对外开放，如对特定ip开放则改为指定ip
network.host: 0.0.0.0
# 节点列表
discovery.seed_hosts: ["10.147.17.206:9300","10.147.17.206:9301"]
# 初始化时master节点的选举列表
cluster.initial_master_nodes: ["es01","es02","es03"]
# 对外提供服务的端口
http.port: 9202
# 内部服务端口
transport.port: 9302
# 跨域支持
http.cors.enabled: true
# 跨域访问允许的域名地址（正则）
http.cors.allow-origin: /.*/
```

kibana 可视化部署和单机模式部署相同，修改 es 集群地址即可

```yaml
elasticsearch.hosts: ["http://10.147.17.206:9200","http://10.147.17.206:9201","http://10.147.17.206:9202"]
```

## 安装中文分词插件

到开源仓库 **[elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)** 中查找与 Elasticsearch 版本号一致的插件（版本号必须一致，否则启动报错）。

安装流程自行参考项目说明。

> 集群部署需要在每个节点都安装插件。

## 参考

Elasticsearch Docker方式部署官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docker.html

Elasticsearch 重要配置：https://www.elastic.co/guide/en/elasticsearch/reference/7.17/important-settings.html

Kibana Docker部署官方文档：https://www.elastic.co/guide/en/kibana/7.17/docker.html

IK 分词插件：https://github.com/medcl/elasticsearch-analysis-ik