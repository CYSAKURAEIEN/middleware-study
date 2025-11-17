# Elasticsearch 搜索引擎架构

## Elasticsearch 整体架构

### 1. ES 集群架构
```mermaid
graph TB
    subgraph "客户端层"
        CLIENT1[Java Client]
        CLIENT2[REST API]
        CLIENT3[Kibana]
    end

    subgraph "Elasticsearch 集群"
        subgraph "节点类型"
            MASTER[Master Node<br/>主节点<br/>集群管理]
            DATA1[Data Node 1<br/>数据节点<br/>存储+查询]
            DATA2[Data Node 2<br/>数据节点<br/>存储+查询]
            DATA3[Data Node 3<br/>数据节点<br/>存储+查询]
            COORD[Coordinating Node<br/>协调节点<br/>请求路由]
            INGEST[Ingest Node<br/>预处理节点<br/>数据转换]
        end
    end

    subgraph "索引分片"
        subgraph "索引: products"
            P0[Primary Shard 0]
            R0[Replica Shard 0]
            P1[Primary Shard 1]
            R1[Replica Shard 1]
        end
    end

    CLIENT1 -->|请求| COORD
    CLIENT2 -->|HTTP| COORD
    CLIENT3 -->|查询| COORD

    COORD -->|路由| DATA1
    COORD -->|路由| DATA2
    COORD -->|路由| DATA3

    MASTER -.管理.-> DATA1
    MASTER -.管理.-> DATA2
    MASTER -.管理.-> DATA3

    DATA1 --> P0
    DATA1 --> R1
    DATA2 --> P1
    DATA2 --> R0
    DATA3 --> P1

    P0 -.复制.-> R0
    P1 -.复制.-> R1

    style MASTER fill:#FF6B6B
    style DATA1 fill:#4ECDC4
    style DATA2 fill:#4ECDC4
    style DATA3 fill:#4ECDC4
    style COORD fill:#FFB84D
    style INGEST fill:#A78BFA
    style P0 fill:#4CAF50
    style P1 fill:#4CAF50
    style R0 fill:#FFE66D
    style R1 fill:#FFE66D
```

### 2. ES 核心概念映射
```mermaid
graph TB
    subgraph "概念对比"
        subgraph "Elasticsearch"
            ES_CLUSTER[Cluster 集群]
            ES_NODE[Node 节点]
            ES_INDEX[Index 索引]
            ES_TYPE[Type 类型<br/>7.x已废弃]
            ES_DOC[Document 文档]
            ES_FIELD[Field 字段]
            ES_SHARD[Shard 分片]

            ES_CLUSTER --> ES_NODE
            ES_NODE --> ES_INDEX
            ES_INDEX --> ES_TYPE
            ES_TYPE --> ES_DOC
            ES_DOC --> ES_FIELD
            ES_INDEX --> ES_SHARD
        end

        subgraph "关系数据库"
            DB_CLUSTER[Database Cluster]
            DB_SERVER[Database Server]
            DB_DATABASE[Database]
            DB_TABLE[Table]
            DB_ROW[Row]
            DB_COLUMN[Column]
            DB_PARTITION[Partition]

            DB_CLUSTER --> DB_SERVER
            DB_SERVER --> DB_DATABASE
            DB_DATABASE --> DB_TABLE
            DB_TABLE --> DB_ROW
            DB_ROW --> DB_COLUMN
            DB_TABLE --> DB_PARTITION
        end
    end

    ES_CLUSTER -.对应.-> DB_CLUSTER
    ES_NODE -.对应.-> DB_SERVER
    ES_INDEX -.对应.-> DB_DATABASE
    ES_TYPE -.对应.-> DB_TABLE
    ES_DOC -.对应.-> DB_ROW
    ES_FIELD -.对应.-> DB_COLUMN
    ES_SHARD -.对应.-> DB_PARTITION

    style ES_INDEX fill:#4ECDC4
    style DB_DATABASE fill:#FFB84D
```

### 3. 倒排索引原理
```mermaid
graph TB
    subgraph "原始文档"
        DOC1[文档1: The quick brown fox]
        DOC2[文档2: Quick brown dogs]
        DOC3[文档3: The lazy dog]
    end

    subgraph "分词处理"
        ANALYZE[Analyzer 分析器]
        TOKENIZE[Tokenizer 分词]
        FILTER[Token Filter 过滤]
        LOWER[转小写]

        ANALYZE --> TOKENIZE --> FILTER --> LOWER
    end

    subgraph "倒排索引 - Inverted Index"
        TERM_TABLE[Term Dictionary<br/>词项字典]

        subgraph "Posting List - 倒排列表"
            TERM1[the → Doc1, Doc3]
            TERM2[quick → Doc1, Doc2]
            TERM3[brown → Doc1, Doc2]
            TERM4[fox → Doc1]
            TERM5[dog → Doc2, Doc3]
            TERM6[lazy → Doc3]
        end

        TERM_TABLE --> TERM1
        TERM_TABLE --> TERM2
        TERM_TABLE --> TERM3
        TERM_TABLE --> TERM4
        TERM_TABLE --> TERM5
        TERM_TABLE --> TERM6
    end

    DOC1 --> ANALYZE
    DOC2 --> ANALYZE
    DOC3 --> ANALYZE

    LOWER --> TERM_TABLE

    style TERM_TABLE fill:#4CAF50
    style TERM1 fill:#FFE66D
    style TERM2 fill:#FFE66D
    style TERM3 fill:#FFE66D
    style TERM4 fill:#FFE66D
    style TERM5 fill:#FFE66D
    style TERM6 fill:#FFE66D
```

## 数据写入流程

### 1. 写入流程详解
```mermaid
sequenceDiagram
    participant C as 客户端
    participant COORD as 协调节点
    participant PRIMARY as 主分片
    participant REPLICA as 副本分片
    participant MEM as 内存Buffer
    participant SEG as Segment

    C->>COORD: 1. 索引文档请求
    COORD->>COORD: 2. 计算文档路由<br/>hash(doc_id) % num_shards
    COORD->>PRIMARY: 3. 转发到主分片

    PRIMARY->>MEM: 4. 写入内存Buffer
    PRIMARY->>PRIMARY: 5. 写入Translog(持久化)

    loop Refresh - 每1秒
        MEM->>SEG: 6. 生成Segment<br/>文档可搜索
    end

    PRIMARY->>REPLICA: 7. 同步到副本分片
    REPLICA->>REPLICA: 8. 副本写入数据

    PRIMARY->>COORD: 9. 主分片返回成功
    REPLICA->>COORD: 10. 副本返回成功
    COORD->>C: 11. 返回写入结果

    Note over SEG: Flush操作：合并Segment<br/>清空Translog
```

### 2. Refresh 和 Flush 机制
```mermaid
graph TB
    subgraph "写入流程"
        subgraph "实时写入"
            WRITE[写入请求]
            BUFFER[内存Buffer<br/>Index Buffer]
            TRANSLOG[Translog<br/>事务日志]

            WRITE --> BUFFER
            WRITE --> TRANSLOG
        end

        subgraph "Refresh - 每1秒"
            SEGMENT1[生成Segment]
            SEARCH[可被搜索<br/>Near Real-time]

            BUFFER -.refresh.-> SEGMENT1
            SEGMENT1 --> SEARCH
        end

        subgraph "Flush - 每30分钟"
            MERGE[合并Segment]
            COMMIT[写入磁盘]
            CLEAR[清空Translog]

            SEGMENT1 -.flush.-> MERGE
            MERGE --> COMMIT
            COMMIT --> CLEAR
            TRANSLOG -.flush.-> CLEAR
        end
    end

    style BUFFER fill:#FFE66D
    style TRANSLOG fill:#FF6B6B
    style SEGMENT1 fill:#4ECDC4
    style MERGE fill:#4CAF50
```

## 数据查询流程

### 1. 查询流程 - Query Then Fetch
```mermaid
sequenceDiagram
    participant C as 客户端
    participant COORD as 协调节点
    participant SHARD1 as 分片1
    participant SHARD2 as 分片2
    participant SHARD3 as 分片3

    C->>COORD: 1. 发送查询请求
    COORD->>COORD: 2. 解析查询DSL

    par Query阶段 - 查询
        COORD->>SHARD1: 3. 广播查询
        COORD->>SHARD2: 3. 广播查询
        COORD->>SHARD3: 3. 广播查询

        SHARD1->>SHARD1: 4. 执行查询
        SHARD2->>SHARD2: 4. 执行查询
        SHARD3->>SHARD3: 4. 执行查询

        SHARD1->>COORD: 5. 返回DocID+分数
        SHARD2->>COORD: 5. 返回DocID+分数
        SHARD3->>COORD: 5. 返回DocID+分数
    end

    COORD->>COORD: 6. 合并排序结果

    par Fetch阶段 - 获取
        COORD->>SHARD1: 7. 请求完整文档
        COORD->>SHARD2: 7. 请求完整文档

        SHARD1->>COORD: 8. 返回文档内容
        SHARD2->>COORD: 8. 返回文档内容
    end

    COORD->>C: 9. 返回最终结果
```

### 2. 聚合查询流程
```mermaid
graph TB
    subgraph "聚合查询"
        QUERY[聚合查询请求]

        subgraph "Bucket聚合 - 分桶"
            TERMS[Terms聚合<br/>按字段分组]
            RANGE[Range聚合<br/>范围分桶]
            DATE[Date聚合<br/>时间分桶]
        end

        subgraph "Metric聚合 - 指标"
            AVG[平均值]
            SUM[求和]
            MAX[最大值]
            MIN[最小值]
            COUNT[计数]
        end

        subgraph "Pipeline聚合 - 管道"
            DERIVATIVE[导数]
            CUMULATIVE[累计]
        end
    end

    QUERY --> TERMS
    QUERY --> RANGE
    QUERY --> DATE

    TERMS --> AVG
    TERMS --> SUM
    RANGE --> MAX
    RANGE --> MIN
    DATE --> COUNT

    AVG --> DERIVATIVE
    SUM --> CUMULATIVE

    style TERMS fill:#4ECDC4
    style RANGE fill:#4ECDC4
    style DATE fill:#4ECDC4
    style AVG fill:#FFB84D
    style SUM fill:#FFB84D
    style DERIVATIVE fill:#A78BFA
```

## 索引设计与优化

### 1. Mapping 映射设计
```mermaid
graph TB
    subgraph "Mapping - 索引映射"
        subgraph "字段类型"
            TEXT[Text<br/>全文本<br/>分词索引]
            KEYWORD[Keyword<br/>精确匹配<br/>不分词]
            NUMERIC[数值类型<br/>long/double]
            DATE[日期类型<br/>date]
            BOOL[布尔类型<br/>boolean]
            NESTED[嵌套类型<br/>nested]
            GEO[地理类型<br/>geo_point]
        end

        subgraph "字段属性"
            INDEX[index<br/>是否索引]
            STORE[store<br/>是否存储]
            ANALYZER[analyzer<br/>分析器]
            FORMAT[format<br/>格式]
        end

        subgraph "常用分析器"
            STANDARD[standard<br/>标准分词]
            IK[IK分词器<br/>中文分词]
            PINYIN[拼音分词器]
        end
    end

    TEXT --> INDEX
    TEXT --> ANALYZER
    KEYWORD --> INDEX
    DATE --> FORMAT

    ANALYZER --> STANDARD
    ANALYZER --> IK
    ANALYZER --> PINYIN

    style TEXT fill:#4ECDC4
    style KEYWORD fill:#FFB84D
    style NUMERIC fill:#FFE66D
    style IK fill:#4CAF50
```

### 2. 索引别名与模板
```mermaid
graph TB
    subgraph "索引管理策略"
        subgraph "索引别名 - Alias"
            ALIAS[索引别名: logs]
            INDEX1[logs-2024-01]
            INDEX2[logs-2024-02]
            INDEX3[logs-2024-03]

            ALIAS --> INDEX1
            ALIAS --> INDEX2
            ALIAS --> INDEX3
        end

        subgraph "索引模板 - Template"
            TEMPLATE[Index Template]
            PATTERN[匹配模式: logs-*]
            SETTINGS[Settings配置]
            MAPPINGS[Mappings映射]

            TEMPLATE --> PATTERN
            TEMPLATE --> SETTINGS
            TEMPLATE --> MAPPINGS
        end

        subgraph "索引生命周期 - ILM"
            HOT[Hot阶段<br/>高频读写]
            WARM[Warm阶段<br/>低频查询]
            COLD[Cold阶段<br/>归档]
            DELETE[Delete阶段<br/>删除]

            HOT --> WARM --> COLD --> DELETE
        end
    end

    TEMPLATE -.自动创建.-> INDEX1
    TEMPLATE -.自动创建.-> INDEX2

    INDEX1 -.生命周期.-> HOT
    INDEX2 -.生命周期.-> WARM
    INDEX3 -.生命周期.-> COLD

    style ALIAS fill:#4CAF50
    style TEMPLATE fill:#FFB84D
    style HOT fill:#FF6B6B
    style WARM fill:#FFE66D
    style COLD fill:#4ECDC4
```

## 典型应用场景

### 1. ELK 日志分析架构
```mermaid
graph LR
    subgraph "数据源"
        APP1[应用服务器1]
        APP2[应用服务器2]
        APP3[应用服务器3]
        NGINX[Nginx日志]
    end

    subgraph "数据采集"
        FILEBEAT[Filebeat<br/>日志采集]
        LOGSTASH[Logstash<br/>数据处理]
    end

    subgraph "存储与搜索"
        ES[Elasticsearch<br/>存储+搜索]
    end

    subgraph "可视化"
        KIBANA[Kibana<br/>数据可视化]
    end

    APP1 --> FILEBEAT
    APP2 --> FILEBEAT
    APP3 --> FILEBEAT
    NGINX --> FILEBEAT

    FILEBEAT --> LOGSTASH
    LOGSTASH -->|清洗转换| ES
    ES --> KIBANA

    style FILEBEAT fill:#4ECDC4
    style LOGSTASH fill:#FFB84D
    style ES fill:#4CAF50
    style KIBANA fill:#FF6B6B
```

### 2. 电商搜索架构
```mermaid
graph TB
    subgraph "用户层"
        USER[用户搜索请求]
    end

    subgraph "应用层"
        SEARCH_API[搜索API服务]
    end

    subgraph "缓存层"
        REDIS[Redis缓存<br/>热门搜索词]
    end

    subgraph "搜索引擎"
        ES_CLUSTER[Elasticsearch集群]

        subgraph "索引"
            PRODUCT[商品索引]
            CATEGORY[类目索引]
            BRAND[品牌索引]
        end
    end

    subgraph "数据同步"
        CANAL[Canal<br/>MySQL日志解析]
        MYSQL[(MySQL<br/>商品数据库)]
    end

    USER --> SEARCH_API
    SEARCH_API --> REDIS
    REDIS -.未命中.-> ES_CLUSTER
    SEARCH_API --> ES_CLUSTER

    ES_CLUSTER --> PRODUCT
    ES_CLUSTER --> CATEGORY
    ES_CLUSTER --> BRAND

    MYSQL -.binlog.-> CANAL
    CANAL -->|实时同步| ES_CLUSTER

    style ES_CLUSTER fill:#4CAF50
    style REDIS fill:#FF6B6B
    style CANAL fill:#FFB84D
```

### 3. 全文搜索优化
```mermaid
graph TB
    subgraph "搜索优化策略"
        subgraph "相关性优化"
            BM25[BM25算法<br/>相关性打分]
            BOOST[字段权重<br/>Boost]
            FUNCTION[函数打分<br/>Function Score]
        end

        subgraph "性能优化"
            ROUTING[路由优化<br/>Routing]
            FILTER[过滤缓存<br/>Filter Cache]
            HIGHLIGHT[高亮优化<br/>Highlight]
        end

        subgraph "智能搜索"
            SUGGEST[搜索建议<br/>Suggest]
            FUZZY[模糊查询<br/>Fuzzy]
            SYNONYM[同义词<br/>Synonym]
            PINYIN[拼音搜索<br/>Pinyin]
        end
    end

    style BM25 fill:#4ECDC4
    style BOOST fill:#4ECDC4
    style ROUTING fill:#FFB84D
    style FILTER fill:#FFB84D
    style SUGGEST fill:#4CAF50
    style SYNONYM fill:#4CAF50
```

## 集群管理与监控

### 1. 集群健康状态
```mermaid
graph TB
    subgraph "集群状态"
        STATUS{Cluster Health}

        GREEN[Green 绿色<br/>所有分片正常]
        YELLOW[Yellow 黄色<br/>主分片正常<br/>部分副本异常]
        RED[Red 红色<br/>部分主分片异常<br/>数据不完整]

        STATUS --> GREEN
        STATUS --> YELLOW
        STATUS --> RED
    end

    subgraph "监控指标"
        CPU[CPU使用率]
        MEM[内存使用率]
        DISK[磁盘使用率]
        JVM[JVM堆内存]
        QUERY[查询性能]
        INDEX_RATE[索引速率]
    end

    GREEN -.监控.-> CPU
    YELLOW -.监控.-> MEM
    RED -.告警.-> DISK

    style GREEN fill:#4CAF50
    style YELLOW fill:#FFB84D
    style RED fill:#FF6B6B
```

## 最佳实践建议

### 1. 分片设计
- ✅ 单分片大小控制在 20-50GB
- ✅ 节点数 ≈ 主分片数
- ✅ 副本数至少为1（高可用）
- ❌ 避免过多小分片（overhead大）
- ❌ 避免单分片过大（影响恢复速度）

### 2. 性能优化
```mermaid
graph LR
    OPT[性能优化] --> WRITE[写入优化]
    OPT --> READ[查询优化]

    WRITE --> W1[增大refresh_interval]
    WRITE --> W2[批量写入bulk]
    WRITE --> W3[减少副本数]

    READ --> R1[使用filter代替query]
    READ --> R2[避免深分页]
    READ --> R3[使用routing]
    READ --> R4[开启查询缓存]

    style WRITE fill:#4ECDC4
    style READ fill:#FFB84D
```

### 3. 数据建模
- 使用 keyword 类型用于精确匹配和聚合
- 使用 text 类型用于全文搜索
- 合理使用 nested 和 parent-child
- 避免过深的嵌套结构
- 冗余设计，避免 join

### 4. 容量规划
- 预估数据量和增长速度
- 计算所需存储空间（考虑副本）
- 预留30%的磁盘空间
- 规划索引生命周期管理
- 定期清理过期数据

### 5. 安全加固
- 启用 X-Pack 安全认证
- 配置 SSL/TLS 加密
- 使用角色权限控制
- 审计日志记录
- 定期备份快照
