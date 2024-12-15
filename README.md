# Nginx-Flume-Kafka-ClickHouse 日志采集系统

这是一个基于Docker Compose的日志采集系统，用于收集Nginx访问日志，通过Flume传输到Kafka，最终存储到ClickHouse中进行分析。

## 系统架构

- Nginx: 生成访问日志
- Flume: 采集Nginx日志
- Kafka: 消息队列
- ClickHouse: 数据存储和分析

## 目录结构


```
├── docker-compose.yml
├── nginx
│ ├── conf.d
│ │ └── default.conf
│ └── logs
├── flume
│ ├── conf
│ │ └── flume-kafka.conf
│ └── logs
├── kafka
│ └── data
├── zookeeper
│ ├── data
│ └── datalog
└── clickhouse
├── conf
├── data
└── logs
```

## 快速开始

### 1. 创建必要的目录

```bash
mkdir -p nginx/{conf.d,logs}
mkdir -p flume/{conf,logs}
mkdir -p kafka/data
mkdir -p zookeeper/{data,datalog}
mkdir -p clickhouse/{conf,data,logs}
```

### 2. 配置文件准备

1. 将 `default.conf` 复制到 `nginx/conf.d/` 目录
2. 将 `flume-kafka.conf` 复制到 `flume/conf/` 目录

### 3. 启动服务

```bash
docker-compose up -d
```

### 4. 创建ClickHouse表结构
1. 进入ClickHouse容器：
   
```bash
docker-compose exec clickhouse-server clickhouse-client
```

2. 执行以下SQL语句：

```sql

// 启用实验特性
SET allow_experimental_object_type = 1;

use bi;

// 创建一个Kafka队列
CREATE TABLE queue1 (
message String
) ENGINE = Kafka('kafka:9092', 'flume_topic', 'group1', 'JSONAsString');


// 创建一个表，用于存储Kafka消息
CREATE TABLE msg (
id UUID,
timestamp String,
remote_addr String,
name String,
agent String
) ENGINE = ReplacingMergeTree() ORDER BY id;


// 创建一个视图，将Kafka消息转换为ClickHouse格式
CREATE MATERIALIZED VIEW queue_msg TO msg AS
SELECT
generateUUIDv4() AS id,
JSONExtractString(message, 'timestamp') AS timestamp,
JSONExtractString(message, 'remote_addr') AS remote_addr,
JSONExtract(JSONExtract(message, 'request_body', 'String'), 'name', 'String') AS name,
JSONExtractString(message, 'agent') AS agent
FROM queue1;
```

## 测试系统
### 1. 测试Nginx日志生成

访问POST接口：

```bash
curl -X POST http://localhost/post -d '{"name":"test"}'
```

### 2. 检查Kafka消息

```bash
docker-compose exec kafka kafka-console-consumer.sh \
--bootstrap-server kafka:9092 \
--topic flume_topic \
--from-beginning
```

### 3. 查询ClickHouse数据

```bash
docker-compose exec clickhouse-server clickhouse-client --query "SELECT FROM bi.msg"

# 查询所有字段
docker-compose exec clickhouse-server clickhouse-client --query "SELECT * FROM bi.msg"


# 或者指定具体字段
docker-compose exec clickhouse-server clickhouse-client --query "SELECT id, timestamp, remote_addr, name, agent FROM bi.msg"

```

## 数据持久化
所有服务的数据和日志都通过Docker volumes映射到本地目录：

- Nginx日志: `./nginx/logs`
- Kafka数据: `./kafka/data`
- Zookeeper数据: `./zookeeper/data`
- ClickHouse数据: `./clickhouse/data`
- ClickHouse日志: `./clickhouse/logs`


## 服务端口
- Nginx: 80
- Zookeeper: 2181
- Kafka: 9092
- ClickHouse HTTP: 8123
- ClickHouse Native: 9000


## 故障排查

### 查看服务日志
```bash
docker-compose logs
```

### 查看特定服务日志

```bash
docker-compose logs nginx
docker-compose logs flume
docker-compose logs kafka
docker-compose logs clickhouse-server
docker-compose logs zookeeper
```

### 查看容器状态
docker-compose ps
docker-compose exec nginx nginx -t



### 常见问题
1. 如果Kafka连接失败，检查KAFKA_ADVERTISED_HOST_NAME配置是否正确
2. 如果Flume无法采集日志，检查日志文件权限是否正确
3. 如果ClickHouse无法接收数据，检查Kafka主题是否创建成功

## 停止服务

```bash
docker-compose down

 #停止并删除数据卷
docker-compose down -v
```


## 参考日志

### flume 启动日志
```
arch-flume-1  |         compression.type = none
arch-flume-1  |         metric.reporters = []
arch-flume-1  |         metadata.max.age.ms = 300000
arch-flume-1  |         metadata.fetch.timeout.ms = 60000
arch-flume-1  |         reconnect.backoff.ms = 50
arch-flume-1  |         sasl.kerberos.ticket.renew.window.factor = 0.8
arch-flume-1  |         bootstrap.servers = [kafka:9092]
arch-flume-1  |         retry.backoff.ms = 1000
arch-flume-1  |         sasl.kerberos.kinit.cmd = /usr/bin/kinit
arch-flume-1  |         buffer.memory = 33554432
arch-flume-1  |         timeout.ms = 30000
arch-flume-1  |         key.serializer = class org.apache.kafka.common.serialization.StringSerializer
arch-flume-1  |         sasl.kerberos.service.name = null
arch-flume-1  |         sasl.kerberos.ticket.renew.jitter = 0.05
arch-flume-1  |         ssl.keystore.type = JKS
arch-flume-1  |         ssl.trustmanager.algorithm = PKIX
arch-flume-1  |         block.on.buffer.full = false
arch-flume-1  |         ssl.key.password = null
arch-flume-1  |         max.block.ms = 60000
arch-flume-1  |         sasl.kerberos.min.time.before.relogin = 60000
arch-flume-1  |         connections.max.idle.ms = 540000
arch-flume-1  |         ssl.truststore.password = null
arch-flume-1  |         max.in.flight.requests.per.connection = 5
arch-flume-1  |         metrics.num.samples = 2
arch-flume-1  |         client.id = 
arch-flume-1  |         ssl.endpoint.identification.algorithm = null
arch-flume-1  |         ssl.protocol = TLS
arch-flume-1  |         request.timeout.ms = 30000
arch-flume-1  |         ssl.provider = null
arch-flume-1  |         ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
arch-flume-1  |         acks = 1
arch-flume-1  |         batch.size = 16384
arch-flume-1  |         ssl.keystore.location = null
arch-flume-1  |         receive.buffer.bytes = 32768
arch-flume-1  |         ssl.cipher.suites = null
arch-flume-1  |         ssl.truststore.type = JKS
arch-flume-1  |         security.protocol = PLAINTEXT
arch-flume-1  |         retries = 3
arch-flume-1  |         max.request.size = 1048576
arch-flume-1  |         value.serializer = class org.apache.kafka.common.serialization.ByteArraySerializer
arch-flume-1  |         ssl.truststore.location = null
arch-flume-1  |         ssl.keystore.password = null
arch-flume-1  |         ssl.keymanager.algorithm = SunX509
arch-flume-1  |         metrics.sample.window.ms = 30000
arch-flume-1  |         partitioner.class = class org.apache.kafka.clients.producer.internals.DefaultPartitioner
arch-flume-1  |         send.buffer.bytes = 131072
arch-flume-1  |         linger.ms = 0
arch-flume-1  | 
arch-flume-1  | 2024-12-15 09:37:41,425 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.instrumentation.MonitoredCounterGroup.register(MonitoredCounterGroup.java:119)] Monitored counter group for type: SOURCE, name: source1: Successfully registered new MBean.
arch-flume-1  | 2024-12-15 09:37:41,425 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.instrumentation.MonitoredCounterGroup.start(MonitoredCounterGroup.java:95)] Component type: SOURCE, name: source1 started
arch-flume-1  | 2024-12-15 09:37:41,512 (lifecycleSupervisor-1-5) [INFO - org.apache.kafka.common.utils.AppInfoParser$AppInfo.<init>(AppInfoParser.java:82)] Kafka version : 0.9.0.1
arch-flume-1  | 2024-12-15 09:37:41,516 (lifecycleSupervisor-1-5) [INFO - org.apache.kafka.common.utils.AppInfoParser$AppInfo.<init>(AppInfoParser.java:83)] Kafka commitId : 23c69d62a0cabf06
arch-flume-1  | 2024-12-15 09:37:41,522 (lifecycleSupervisor-1-5) [INFO - org.apache.flume.instrumentation.MonitoredCounterGroup.register(MonitoredCounterGroup.java:119)] Monitored counter group for type: SINK, name: sink1: Successfully registered new MBean.
arch-flume-1  | 2024-12-15 09:37:41,523 (lifecycleSupervisor-1-5) [INFO - org.apache.flume.instrumentation.MonitoredCounterGroup.start(MonitoredCounterGroup.java:95)] Component type: SINK, name: sink1 started
```


### kafka 消费者日志

```bash

$ docker-compose exec kafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic flume_topic --from-beginning

{"timestamp": "2024-12-15T09:30:38+00:00", "remote_addr": "192.168.112.1", "request_body": "", "agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" }
{"timestamp": "2024-12-15T09:30:38+00:00", "remote_addr": "192.168.112.1", "request_body": "", "agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" }
{"timestamp": "2024-12-15T09:30:38+00:00", "remote_addr": "192.168.112.1", "request_body": "", "agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" }
{"timestamp": "2024-12-15T09:30:38+00:00", "remote_addr": "192.168.112.1", "request_body": "", "agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" }
{"timestamp": "2024-12-15T09:30:39+00:00", "remote_addr": "192.168.112.1", "request_body": "", "agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" }

```
