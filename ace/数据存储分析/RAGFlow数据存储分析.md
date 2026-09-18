# RAGFlow 数据存储技术分析

## 概述

RAGFlow 采用可插拔的多存储架构,通过环境变量和 Docker Compose profiles 实现灵活的存储后端选择。本文档详细分析项目中使用的所有数据存储技术及其性能指标。

**核心配置机制:**
- `DOC_ENGINE`: 文档引擎选择(elasticsearch/infinity/opensearch/serenedb/seekdb/gaussdb)
- `DB_TYPE`: 元数据库选择(mysql/postgres/gaussdb/oceanbase)
- Docker Compose profiles: 条件化服务启动

---

## 1. 关系型数据库(元数据存储)

### 1.1 MySQL 8.0.40(默认选项)

**作用:**
- 存储用户账户、权限、文档索引元数据
- 存储知识库配置、对话历史、系统配置
- 通过 Peewee ORM 提供数据持久化层

**配置信息:**
```yaml
host: ${MYSQL_HOST:-mysql}
port: 3306
database: ${MYSQL_DBNAME:-rag_flow}
user: ${MYSQL_USER:-root}
password: ${MYSQL_PASSWORD:-infini_rag_flow}
max_connections: 1000
max_allowed_packet: 1073741824  # 1GB
stale_timeout: 300
```

**数据承载量:**
- 连接池: 900 个应用连接(max_connections 1000)
- 单包大小: 最大 1GB(支持大型 BLOB 字段)
- 存储容量: 受 Docker volume 限制,默认无上限
- 字符集: utf8mb4(完整 Unicode 支持)

**检索速度与性能:**
- 连接复用: 300秒连接池超时
- 索引策略: Peewee ORM 自动生成索引
- 事务支持: ACID 完整性保证
- 典型查询延迟: <10ms(单表查询)

**Docker 镜像:** `mysql:8.0.40`
**Profile 激活:** `mysql`, `metadata-mysql`, `metadata-MySQL`, `metadata-MYSQL`

---

### 1.2 PostgreSQL(可选)

**作用:**
- 可替代 MySQL 作为元数据存储
- 支持更复杂的 JSON 查询和全文检索

**配置信息:**
```yaml
host: ${POSTGRES_HOST:-postgres}
port: 5432
database: ${POSTGRES_DBNAME:-rag_flow}
user: ${POSTGRES_USER:-postgres}
password: ${POSTGRES_PASSWORD:-infini_rag_flow}
```

**数据承载量:**
- 连接数: 未显式限制(PostgreSQL 默认 100)
- 存储容量: 无硬性上限
- JSON 支持: 原生 JSONB 类型

**检索速度与性能:**
- 全文检索: 内置 tsvector/tsquery
- 并发性能: MVCC 多版本并发控制
- 复杂查询: 优于 MySQL 的查询优化器

**Profile 激活:** `metadata-postgres`

---

### 1.3 GaussDB(华为高斯数据库)

**作用:**
- 国产化替代方案
- 兼容 PostgreSQL 协议
- 适用于私有化部署和信创要求

**配置信息:**
```yaml
host: ${GAUSSDB_HOST:-gaussdb}
port: ${GAUSSDB_PORT:-5432}
database: ${GAUSSDB_DBNAME:-rag_flow}
```

**数据承载量:**
- 基于华为云架构,支持 PB 级存储
- 分布式存储和计算分离

**检索速度与性能:**
- 分布式查询优化
- 行列混合存储
- 适合 OLTP 和轻量 OLAP 混合场景

**Profile 激活:** `metadata-gaussdb`

---

### 1.4 OceanBase 4.4.1.0

**作用:**
- 国产分布式关系数据库
- 兼作元数据存储(`DB_TYPE=oceanbase`)和文档引擎(`DOC_ENGINE=oceanbase`)
- 支持 HTAP(混合事务/分析处理)

**配置信息:**
```yaml
host: ${OCEANBASE_HOST:-oceanbase}
port: 2881
doc_database: ${OCEANBASE_DOC_DBNAME:-ragflow_doc}
metadata_database: rag_flow
user: root@ragflow
password: ${OCEANBASE_PASSWORD:-infini_rag_flow}
```

**容器配置:**
```yaml
OB_CLUSTER_NAME: ragflow
OB_TENANT_NAME: ragflow
OB_MEMORY_LIMIT: 10G
OB_SYSTEM_MEMORY: 2G
OB_DATAFILE_SIZE: 20G
OB_LOG_DISK_SIZE: 20G
```

**数据承载量:**
- 租户内存上限: 10GB
- 系统内存预留: 2GB
- 数据文件空间: 20GB
- 日志磁盘空间: 20GB
- 集群级扩展: 支持多租户、多副本

**检索速度与性能:**
- 事务延迟: <5ms(单租户内)
- 分布式一致性: Paxos 协议
- 混合负载: OLTP 和 OLAP 同时支持
- 自适应查询优化器

**Docker 镜像:** `oceanbase/oceanbase-ce:4.4.1.0-101000022026082014`
**Profile 激活:** `oceanbase`, `doc-oceanbase`, `metadata-oceanbase`

---

### 1.5 SeekDB(OceanBase Lite)

**作用:**
- OceanBase 的轻量单机版本
- 作为文档引擎使用(`DOC_ENGINE=seekdb`)
- 低资源消耗,适合开发测试环境

**配置信息:**
```yaml
host: ${SEEKDB_HOST:-seekdb}
port: 2881
database: ${SEEKDB_DOC_DBNAME:-ragflow_doc}
user: root
password: ${SEEKDB_PASSWORD:-infini_rag_flow}
memory_limit: 2G
```

**数据承载量:**
- 内存限制: 2GB(最小资源模式)
- 存储方式: 单机文件系统
- 适用规模: 中小型文档集合(<100万文档)

**检索速度与性能:**
- 简化版 OceanBase 内核
- 无分布式开销
- 查询延迟: <50ms(典型向量检索)

**Docker 镜像:** `seekdb/seekdb:latest`
**Profile 激活:** `seekdb`, `doc-seekdb`

---

## 2. 向量/全文检索引擎

### 2.1 Elasticsearch 8.11.3(默认文档引擎)

**作用:**
- RAGFlow 默认文档引擎
- 存储文档向量、分词索引、元数据
- 提供混合检索(向量+BM25)能力

**配置信息:**
```yaml
hosts: ["http://es01:9200"]
user: elastic
password: ${ELASTIC_PASSWORD:-infini_rag_flow}
cluster.name: docker-cluster
discovery.type: single-node
```

**数据承载量:**
- JVM 堆内存: MEM_LIMIT/2(默认 4GB)
- 索引分片: 自动管理
- 存储容量: Docker volume,无硬性上限
- 向量维度: 支持最高 2048 维

**检索速度与性能:**
- 向量召回延迟: <100ms(百万级)
- BM25 全文检索: <50ms
- 混合检索: 向量+关键词联合排序
- 倒排索引: 实时刷新
- 集群扩展: 支持水平扩展(生产环境)

**Docker 镜像:** `docker.elastic.co/elasticsearch/elasticsearch:8.11.3`
**Profile 激活:** `elasticsearch`(默认)

---

### 2.2 Infinity

**作用:**
- 高性能向量数据库
- 支持向量检索、全文检索、混合检索
- 提供 Thrift/HTTP/PostgreSQL 三种协议接入
- 可作为文档引擎(`DOC_ENGINE=infinity`)

**配置信息:**
```yaml
host: ${INFINITY_HOST:-infinity}
thrift_port: 23817
http_port: 23820
psql_port: 5432
database: default_db
user: ${INFINITY_USER:-infinity}
password: ${INFINITY_PASSWORD:-infini_rag_flow}
```

**数据承载量:**
- 单机模式,容量受限于磁盘空间
- 支持海量向量数据存储
- 列式存储格式,空间效率高

**检索速度与性能:**
- 向量检索延迟: <10ms(百万级数据)
- 支持 HNSW/IVF 等高效索引算法
- 混合检索(向量+全文): 支持
- 并发查询: 高吞吐优化

**Docker 镜像:** `infiniflow/infinity:nightly`
**Profile 激活:** `infinity`

---

### 2.3 OpenSearch

**作用:**
- Elasticsearch 的开源分支
- 提供向量检索和全文检索能力
- 可作为文档引擎(`DOC_ENGINE=opensearch`)

**配置信息:**
```yaml
host: ${OS_HOST:-opensearch01}
port: 1201
user: admin
password: ${OPENSEARCH_PASSWORD:-infini_rag_flow_OS_01}
```

**数据承载量:**
- 单节点内存限制: 8GB(受 MEM_LIMIT 约束)
- 支持分片和副本机制
- 集群模式下可水平扩展

**检索速度与性能:**
- 向量检索: k-NN 插件支持
- 全文检索: BM25 算法
- 混合检索: 支持,通过脚本评分组合
- 聚合分析: 丰富的 Aggregation API

**Docker 镜像:** `opensearchproject/opensearch:2.18.0`
**Profile 激活:** `opensearch`

---

### 2.4 SereneDB

**作用:**
- PostgreSQL 生态兼容的向量数据库
- 支持 pgvector 扩展
- 可作为文档引擎(`DOC_ENGINE=serenedb`)
- 支持 PostgreSQL 有线协议,与现有工具无缝集成

**配置信息:**
```yaml
host: ${SERENEDB_HOST:-serenedb}
port: 7890
database: ${SERENEDB_DBNAME:-ragflow_doc}
user: ${SERENEDB_USER:-serenedb}
password: ${SERENEDB_PASSWORD:-infini_rag_flow}
```

**数据承载量:**
- 基于 PostgreSQL 架构
- 支持大规模向量存储
- 利用 PostgreSQL 成熟的事务和持久化机制

**检索速度与性能:**
- 向量检索: pgvector 扩展支持
- 索引算法: IVFFlat, HNSW
- SQL 查询: 完整的 PostgreSQL 兼容性
- 事务保证: ACID 完整支持

**Docker 镜像:** `docker.io/kwdb/serenedb:latest`
**Profile 激活:** `serenedb`

---

## 3. 对象存储

### 3.1 MinIO(默认)

**作用:**
- S3 兼容的对象存储服务
- 存储文档原始文件、解析结果、模型文件等
- 支持多租户隔离
- RAGFlow 的主要文件存储后端

**配置信息:**
```yaml
host: ${MINIO_HOST:-minio}
console_port: 9001
api_port: 9000
user: ${MINIO_USER:-rag_flow}
password: ${MINIO_PASSWORD:-infini_rag_flow}
```

**数据承载量:**
- 容量: 受限于宿主机磁盘空间
- 单对象大小: 最大 5TB
- Bucket 数量: 无限制
- 多版本支持: 可选

**检索速度与性能:**
- 读写延迟: ~10-50ms(本地部署)
- 吞吐量: 受网络和磁盘 I/O 限制
- 并发连接: 支持高并发
- 多部分上传: 支持大文件分片上传

**Docker 镜像:** `minio/minio:latest`
**Profile 激活:** 默认启用

---

### 3.2 云存储替代方案

**作用:**
- 通过 `STORAGE_FACTORY` 环境变量切换云存储后端
- 支持 AWS S3、阿里云OSS、Azure Blob、Google Cloud Storage

**配置信息:**
```yaml
# S3
STORAGE_FACTORY: s3
AWS_ACCESS_KEY_ID: <access_key>
AWS_SECRET_ACCESS_KEY: <secret_key>
AWS_REGION: <region>

# OSS
STORAGE_FACTORY: oss
OSS_ACCESS_KEY_ID: <access_key>
OSS_SECRET_ACCESS_KEY: <secret_key>
OSS_ENDPOINT: <endpoint>
```

**数据承载量:**
- 依托云服务商基础设施
- 理论容量: 无上限
- 成本: 按用量计费

**检索速度与性能:**
- 延迟: 20-100ms(依赖网络)
- 带宽: 依赖云服务商 SLA
- 可用性: 99.9%~99.99%(云服务商保证)

---

## 4. 缓存与键值存储

### 4.1 Redis/Valkey 8(Python 服务)

**作用:**
- Python 服务的缓存层
- 会话管理、任务队列(Redis Streams)
- 分布式锁
- 热数据缓存

**配置信息:**
```yaml
host: ${REDIS_HOST:-redis}
port: 6379
password: ${REDIS_PASSWORD:-infini_rag_flow}
db: 1
```

**数据承载量:**
- 内存容量: 依赖宿主机配置
- 键空间: 理论无上限
- 单个值: 最大 512MB
- 持久化: RDB + AOF 可选

**检索速度与性能:**
- 读写延迟: <1ms(内存操作)
- QPS: 10万+(单实例)
- 支持 Pipeline 批量操作
- 过期策略: 自动清理

**Docker 镜像:** `valkey/valkey:8-alpine`
**Profile 激活:** 默认启用

---

### 4.2 Kvrocks 2.16.0(Go 服务)

**作用:**
- Go 服务的 Redis 兼容存储
- 基于 RocksDB 的持久化 KV 存储
- 降低内存成本,适合大容量场景

**配置信息:**
```yaml
host: ${KVROCKS_HOST:-kvrocks}
port: 6666
password: ${KVROCKS_PASSWORD:-infini_rag_flow}
```

**数据承载量:**
- 存储介质: 磁盘(RocksDB)
- 容量: TB 级支持
- 内存占用: 远低于 Redis
- 冷热数据分层: 自动管理

**检索速度与性能:**
- 读延迟: 1-5ms(热数据)
- 写延迟: <10ms(顺序写优化)
- QPS: 数万级(单实例)
- LSM-Tree 结构,写入友好

**Docker 镜像:** `apache/kvrocks:2.16.0`
**Profile 激活:** 默认启用

---

## 5. 消息队列

### 5.1 NATS 2.14.2 + JetStream

**作用:**
- 分布式消息队列和事件流平台
- 支持发布/订阅、请求/响应、流式传输
- JetStream 提供持久化和回放能力
- 用于微服务间异步通信

**配置信息:**
```yaml
host: ${NATS_HOST:-nats}
port: 4222
expose_port: 4222
enable_jetstream: true
```

**数据承载量:**
- 内存模式: 受限于可用内存
- 文件持久化: 受限于磁盘空间
- 单消息大小: 默认 1MB,可配置
- 流保留策略: 时间/大小/消息数可配

**检索速度与性能:**
- 消息延迟: <1ms(内存模式)
- 吞吐量: 百万级 msg/s
- JetStream 持久化延迟: 2-5ms
- 支持水平扩展和集群模式

**Docker 镜像:** `nats:2.14.2-alpine`
**Profile 激活:** 默认启用

---

### 5.2 Redis Streams(Python 任务队列)

**作用:**
- Python 服务的任务队列
- Celery/Huey 等任务框架的后端
- 支持任务持久化和重试

**配置信息:**
```yaml
使用 Redis/Valkey 的 Streams 数据结构
consumer_group: ragflow_workers
```

**数据承载量:**
- 继承 Redis 内存限制
- 单 Stream 无理论上限
- 消费者组: 支持多消费者并发

**检索速度与性能:**
- 消息延迟: <5ms
- 任务调度: 异步非阻塞
- ACK 机制: 保证至少一次交付

---

## 6. 分析数据库

### 6.1 ClickHouse

**作用:**
- OLAP 列式数据库
- 存储用户行为日志、查询统计、性能指标
- 支持复杂分析查询和实时报表

**配置信息:**
```yaml
host: ${CLICKHOUSE_HOST:-clickhouse}
tcp_port: 9000
expose_tcp_port: 9900
http_port: 8123
user: ${CLICKHOUSE_USER:-ragflow}
password: ${CLICKHOUSE_PASSWORD:-infini_rag_flow}
database: ${CLICKHOUSE_DATABASE:-ragflow}
```

**数据承载量:**
- 存储: 基于磁盘,支持 PB 级
- 压缩比: 10:1 ~ 100:1(列式压缩)
- 表数量: 无限制
- 分区策略: 按时间/哈希分区

**检索速度与性能:**
- 扫描速度: GB/s 级(单节点)
- 聚合查询: 亚秒级响应(十亿行数据)
- 并发查询: 支持高并发分析
- 写入吞吐: 百万行/秒

**Docker 镜像:** `clickhouse/clickhouse-server:latest`
**Profile 激活:** 可选(分析场景启用)

---

## 7. 知识图谱存储

### 7.1 图数据库方案

**作用:**
- RAGFlow 不使用独立的图数据库
- 知识图谱数据存储在文档引擎(Elasticsearch/Infinity/OpenSearch等)
- 实体和关系以文档形式索引
- 图遍历和关系查询通过应用层实现

**实现方式:**
```yaml
存储层: 复用文档引擎(DOC_ENGINE)
图结构:
  - 实体(Entity): 作为独立文档存储
  - 关系(Relation): 作为边文档存储,包含 source/target 引用
  - 属性(Property): 嵌入在实体/关系文档中
索引:
  - 实体索引: 支持向量检索(实体嵌入)
  - 关系索引: 支持快速查找邻居节点
```

**数据承载量:**
- 继承文档引擎的容量特性
- 实体数: 百万~千万级
- 关系数: 千万~亿级
- 图深度: 无硬性限制

**检索速度与性能:**
- 实体检索: 10-50ms(向量相似度)
- 1-hop 关系查询: 20-100ms
- 多跳图遍历: 100ms-数秒(依赖跳数)
- 混合查询(图+向量): 支持

**设计权衡:**
- 优势: 统一存储,降低运维复杂度,向量+图混合检索
- 劣势: 图遍历性能不如专用图数据库(Neo4j/JanusGraph)
- 适用场景: RAG 场景下的浅层图查询(1-3跳)

---

## 8. 总结

### 8.1 存储架构特点

1. **插拔式设计:** 关系数据库(5选1) + 文档引擎(6选1) + 对象存储(本地/云)
2. **双缓存体系:** Redis(Python) + Kvrocks(Go),协议兼容、职责分离
3. **混合检索:** 向量检索 + 全文检索 + 关系查询,三者融合
4. **云原生友好:** 容器化部署,支持 Kubernetes,兼容云存储

### 8.2 性能对比矩阵

| 存储类型 | 典型延迟 | 吞吐量 | 容量上限 | 适用场景 |
|---------|---------|--------|---------|---------|
| MySQL/PostgreSQL | 5-20ms | 数千 TPS | TB 级 | 元数据、关系数据 |
| Elasticsearch | 10-50ms | 数千 QPS | TB-PB 级 | 向量检索、全文检索 |
| Infinity | <10ms | 万级 QPS | TB 级 | 高性能向量检索 |
| MinIO | 10-50ms | GB/s | PB 级 | 文件存储 |
| Redis | <1ms | 10万+ QPS | GB 级 | 热数据缓存 |
| Kvrocks | 1-5ms | 数万 QPS | TB 级 | 大容量 KV |
| NATS | <1ms | 百万 msg/s | 内存/磁盘 | 消息队列 |
| ClickHouse | 秒级 | 百万行/s | PB 级 | 日志分析 |

### 8.3 容量规划建议

1. **小型部署(< 10万文档):**
   - 元数据: MySQL(默认)
   - 文档引擎: Elasticsearch/Infinity
   - 对象存储: MinIO(100GB)
   - 缓存: Redis(2GB)

2. **中型部署(10万-100万文档):**
   - 元数据: PostgreSQL(HA 双节点)
   - 文档引擎: Elasticsearch 集群(3节点)
   - 对象存储: MinIO 集群/S3
   - 缓存: Redis(8GB) + Kvrocks(50GB)

3. **大型部署(> 100万文档):**
   - 元数据: OceanBase(分布式)
   - 文档引擎: Infinity/OpenSearch 集群
   - 对象存储: 云存储(OSS/S3)
   - 缓存: Redis 集群 + Kvrocks 集群
   - 分析: ClickHouse 集群

---

**文档生成时间:** 2026-09-18  
**RAGFlow 版本:** v0.27.2  
**配置文件来源:** `docker/.env`, `docker/service_conf.yaml.template`, `docker/docker-compose-base.yml`

