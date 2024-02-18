## Kafka KRaft集群部署

### docker-compose.yml编写

```yaml
version: '3.8'
services:
  # kafka:
  #   image: 'bitnami/kafka:3.4.1'
  #   ports:
  #     - '9092:9092'
  #   environment:
  #     - KAFKA_CFG_NODE_ID=0
  #     - KAFKA_CFG_PROCESS_ROLES=controller,broker
  #     - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
  #     - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
  #     - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
  #     - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
  kafka1:
    image: bitnami/kafka:3.4.1
    container_name: kafka1
    volumes:
      - ./kafka1:/bitnami/kafka
    ports:
      - "9092:9092"
      - "9093:9093"
    environment:
      ### 通用配置
      # KAFKA_CFG_NODE_ID 的值应该与 KAFKA_BROKER_ID 相同，以确保 KRaft 模式下节点 ID 与 Broker ID 一致。
      - KAFKA_CFG_NODE_ID=1
      # 替代Zookeeper
      - KAFKA_ENABLE_KRAFT=yes
      # KRaft 模式运行时的节点角色
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      # 侦听器名称
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      # 监听端口配置
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      # 定义安全协议
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      # 使用 Kafka Raft (KRaft) 时的 Kafka 集群 ID，在同一 KRaft 集群中的所有节点都应该有相同的 KAFKA_KRAFT_CLUSTER_ID
      - KAFKA_KRAFT_CLUSTER_ID=LelM2dIFQkiUFvXCEcqRWA
      # 集群地址
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      # Apache Kafka 的 Java 堆大小
      - KAFKA_HEAP_OPTS=-Xmx512M -Xms256M 
      # 允许使用PLAINTEXT监听器，默认false
      - ALLOW_PLAINTEXT_LISTENER=yes
      # 不允许自动创建主题
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=false

      ### broker配置
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://175.178.51.17:9092
      # broker.id，必须唯一
      - KAFKA_BROKER_ID=1
  kafka2:
    image: bitnami/kafka:3.4.1
    container_name: kafka2
    volumes:
      - ./kafka2:/bitnami/kafka
    ports:
      - "9192:9092"
      - "9193:9093"
    environment:
      ### 通用配置
      # KAFKA_CFG_NODE_ID 的值应该与 KAFKA_BROKER_ID 相同，以确保 KRaft 模式下节点 ID 与 Broker ID 一致。
      - KAFKA_CFG_NODE_ID=2
      # 替代Zookeeper
      - KAFKA_ENABLE_KRAFT=yes
      # KRaft 模式运行时的节点角色
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      # 侦听器名称
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      # 监听端口配置
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      # 定义安全协议
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      # 使用 Kafka Raft (KRaft) 时的 Kafka 集群 ID，在同一 KRaft 集群中的所有节点都应该有相同的 KAFKA_KRAFT_CLUSTER_ID
      - KAFKA_KRAFT_CLUSTER_ID=LelM2dIFQkiUFvXCEcqRWA
      # 集群地址
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      # Apache Kafka 的 Java 堆大小
      - KAFKA_HEAP_OPTS=-Xmx512M -Xms256M 
      # 允许使用PLAINTEXT监听器，默认false
      - ALLOW_PLAINTEXT_LISTENER=yes
      # 不允许自动创建主题
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=false

      ### broker配置
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://175.178.51.17:9192
      # broker.id，必须唯一
      - KAFKA_BROKER_ID=2
  kafka3:
    image: bitnami/kafka:3.4.1
    container_name: kafka3
    volumes:
      - ./kafka3:/bitnami/kafka
    ports:
      - "9292:9092"
      - "9293:9093"
    environment:
      ### 通用配置
      # KAFKA_CFG_NODE_ID 的值应该与 KAFKA_BROKER_ID 相同，以确保 KRaft 模式下节点 ID 与 Broker ID 一致。
      - KAFKA_CFG_NODE_ID=3
      # 替代Zookeeper
      - KAFKA_ENABLE_KRAFT=yes
      # KRaft 模式运行时的节点角色
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      # 侦听器名称
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      # 监听端口配置
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      # 定义安全协议
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      # 使用 Kafka Raft (KRaft) 时的 Kafka 集群 ID，在同一 KRaft 集群中的所有节点都应该有相同的 KAFKA_KRAFT_CLUSTER_ID
      - KAFKA_KRAFT_CLUSTER_ID=LelM2dIFQkiUFvXCEcqRWA
      # 集群地址
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      # Apache Kafka 的 Java 堆大小
      - KAFKA_HEAP_OPTS=-Xmx512M -Xms256M 
      # 允许使用PLAINTEXT监听器，默认false
      - ALLOW_PLAINTEXT_LISTENER=yes
      # 不允许自动创建主题
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=false

      ### broker配置
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://175.178.51.17:9292
      # broker.id，必须唯一
      - KAFKA_BROKER_ID=3

```

KAFKA_CFG_ADVERTISED_LISTENERS，和 RocketMQ 的 brokerIP1 配置相似，表示外部网络可以通过该 IP 和端口访问 Kafka 。

### Kafka可视化界面

```yaml
version: '3.8'
services:
  kafka-ui:
    container_name: kafka-ui
    image: provectuslabs/kafka-ui:latest
    network_mode: "host"
    environment:
      DYNAMIC_CONFIG_ENABLED: 'true'
      KAFKA_CLUSTERS_0_NAME: wizard_test
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: 175.178.51.17:9092,175.178.51.17:9192,175.178.51.17:9292
      SERVER_PORT: 19092
```



## 参考

Docker-Compose部署Kafka KRaft集群环境：https://juejin.cn/post/7187301063832109112#heading-11