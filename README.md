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
sql
CREATE TABLE queue1 (
message String
) ENGINE = Kafka('kafka:9092', 'flume_topic', 'group1', 'JSONAsString');
CREATE TABLE msg (
id UUID,
timestamp String,
remote_addr String,
name String,
agent String
) ENGINE = ReplacingMergeTree() ORDER BY id;
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
docker-compose exec clickhouse-server clickhouse-client --query "SELECT FROM msg"

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


