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

## 安装中文分词插件

到开源仓库 **[elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)** 中查找与 Elasticsearch 版本号一致的插件（版本号必须一致，否则启动报错）。

安装流程自行参考项目说明。

## 参考

Elasticsearch Docker方式部署官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docker.html

Kibana Docker部署官方文档：https://www.elastic.co/guide/en/kibana/7.17/docker.html

IK 分词插件：https://github.com/medcl/elasticsearch-analysis-ik