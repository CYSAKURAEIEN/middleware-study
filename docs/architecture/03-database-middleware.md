# 数据库中间件架构

## ShardingSphere 架构详解

### 1. ShardingSphere 生态架构
```mermaid
graph TB
    subgraph "ShardingSphere 生态"
        subgraph "ShardingSphere-JDBC - 客户端"
            JDBC[ShardingSphere-JDBC<br/>轻量级Java框架]
            JDBC_FEAT[数据分片<br/>读写分离<br/>分布式事务<br/>数据加密]
            JDBC --> JDBC_FEAT
        end

        subgraph "ShardingSphere-Proxy - 代理端"
            PROXY[ShardingSphere-Proxy<br/>数据库代理]
            PROXY_FEAT[透明化接入<br/>异构语言支持<br/>DBA运维]
            PROXY --> PROXY_FEAT
        end

        subgraph "ShardingSphere-Sidecar - 云原生"
            SIDECAR[ShardingSphere-Sidecar<br/>Service Mesh]
            SIDECAR_FEAT[Kubernetes<br/>云原生数据库]
            SIDECAR --> SIDECAR_FEAT
        end
    end

    subgraph "应用层"
        APP1[Java应用]
        APP2[Python应用]
        APP3[K8s Pod]
    end

    APP1 --> JDBC
    APP2 --> PROXY
    APP3 --> SIDECAR

    style JDBC fill:#4ECDC4
    style PROXY fill:#FFB84D
    style SIDECAR fill:#A78BFA
```

### 2. ShardingSphere-JDBC 架构
```mermaid
graph TB
    subgraph "应用层"
        APP[业务应用]
    end

    subgraph "ShardingSphere-JDBC 层"
        API[SQL解析器]
        ROUTER[路由引擎]
        REWRITE[SQL改写]
        EXEC[执行引擎]
        MERGE[结果归并]

        API --> ROUTER --> REWRITE --> EXEC --> MERGE
    end

    subgraph "数据库层"
        subgraph "用户库分片"
            DS1[(db_user_0)]
            DS2[(db_user_1)]
        end

        subgraph "订单库分片"
            DS3[(db_order_0)]
            DS4[(db_order_1)]
        end

        subgraph "主从复制"
            MASTER[(Master)]
            SLAVE1[(Slave1)]
            SLAVE2[(Slave2)]
        end
    end

    APP -->|JDBC| API

    EXEC -->|分片| DS1
    EXEC -->|分片| DS2
    EXEC -->|分片| DS3
    EXEC -->|分片| DS4

    EXEC -->|写| MASTER
    EXEC -->|读| SLAVE1
    EXEC -->|读| SLAVE2

    MASTER -.复制.-> SLAVE1
    MASTER -.复制.-> SLAVE2

    style API fill:#FFE66D
    style ROUTER fill:#FFE66D
    style REWRITE fill:#FFE66D
    style EXEC fill:#FFE66D
    style MERGE fill:#FFE66D
```

### 3. 分库分表策略
```mermaid
graph TB
    subgraph "分片策略"
        subgraph "垂直分片"
            V1[按业务模块分库]
            V2[用户库: db_user]
            V3[订单库: db_order]
            V4[商品库: db_product]
            V1 --> V2
            V1 --> V3
            V1 --> V4
        end

        subgraph "水平分片"
            H1[按数据量分库分表]
            H2[用户ID % 4<br/>分4个库]
            H3[订单ID % 8<br/>分8张表]
            H1 --> H2
            H1 --> H3
        end

        subgraph "分片算法"
            A1[取模算法<br/>id % n]
            A2[范围算法<br/>id范围]
            A3[哈希算法<br/>hash id]
            A4[时间算法<br/>按日期]
            A5[自定义算法]
        end
    end

    style V1 fill:#4ECDC4
    style H1 fill:#FFB84D
    style A1 fill:#A78BFA
    style A2 fill:#A78BFA
    style A3 fill:#A78BFA
    style A4 fill:#A78BFA
    style A5 fill:#A78BFA
```

### 4. 读写分离架构
```mermaid
graph LR
    subgraph "应用层"
        APP[应用]
    end

    subgraph "ShardingSphere 路由"
        ROUTER{读写分离路由}
    end

    subgraph "数据库集群"
        MASTER[(主库 Master<br/>写操作)]
        SLAVE1[(从库 Slave1<br/>读操作)]
        SLAVE2[(从库 Slave2<br/>读操作)]
        SLAVE3[(从库 Slave3<br/>读操作)]
    end

    APP -->|SQL| ROUTER

    ROUTER -->|INSERT/UPDATE/DELETE| MASTER
    ROUTER -->|SELECT| SLAVE1
    ROUTER -->|SELECT| SLAVE2
    ROUTER -->|SELECT| SLAVE3

    MASTER -.主从复制.-> SLAVE1
    MASTER -.主从复制.-> SLAVE2
    MASTER -.主从复制.-> SLAVE3

    style MASTER fill:#FF6B6B
    style SLAVE1 fill:#4ECDC4
    style SLAVE2 fill:#4ECDC4
    style SLAVE3 fill:#4ECDC4
    style ROUTER fill:#FFB84D
```

## MyCAT 架构详解

### 1. MyCAT 整体架构
```mermaid
graph TB
    subgraph "客户端层"
        CLIENT1[MySQL Client]
        CLIENT2[JDBC]
        CLIENT3[应用程序]
    end

    subgraph "MyCAT 中间层"
        MYCAT[MyCAT Server<br/>端口:8066]

        subgraph "核心组件"
            PARSER[SQL解析器]
            ROUTER[路由器]
            CACHE[缓存模块]
            SEQUENCE[全局序列号]
            MONITOR[监控统计]
        end

        MYCAT --> PARSER
        PARSER --> ROUTER
        ROUTER --> CACHE
        ROUTER --> SEQUENCE
        MYCAT --> MONITOR
    end

    subgraph "数据存储层"
        subgraph "分片1"
            DN1[(DataNode1<br/>db1)]
        end

        subgraph "分片2"
            DN2[(DataNode2<br/>db2)]
        end

        subgraph "分片3"
            DN3[(DataNode3<br/>db3)]
        end
    end

    CLIENT1 -->|MySQL协议| MYCAT
    CLIENT2 -->|MySQL协议| MYCAT
    CLIENT3 -->|MySQL协议| MYCAT

    ROUTER -->|分片路由| DN1
    ROUTER -->|分片路由| DN2
    ROUTER -->|分片路由| DN3

    style MYCAT fill:#FFB84D
    style PARSER fill:#FFE66D
    style ROUTER fill:#FFE66D
    style CACHE fill:#FFE66D
```

### 2. MyCAT 逻辑库表映射
```mermaid
graph TB
    subgraph "逻辑视图 - MyCAT"
        LOGIC_DB[逻辑库: db_mall]

        subgraph "逻辑表"
            LOGIC_USER[user表<br/>逻辑表]
            LOGIC_ORDER[order表<br/>逻辑表]
        end

        LOGIC_DB --> LOGIC_USER
        LOGIC_DB --> LOGIC_ORDER
    end

    subgraph "物理视图 - 真实数据库"
        subgraph "数据节点1"
            PHY_DB1[db_mall_0]
            USER_0[user_0]
            ORDER_0[order_0]
            ORDER_1[order_1]
            PHY_DB1 --> USER_0
            PHY_DB1 --> ORDER_0
            PHY_DB1 --> ORDER_1
        end

        subgraph "数据节点2"
            PHY_DB2[db_mall_1]
            USER_1[user_1]
            ORDER_2[order_2]
            ORDER_3[order_3]
            PHY_DB2 --> USER_1
            PHY_DB2 --> ORDER_2
            PHY_DB2 --> ORDER_3
        end
    end

    LOGIC_USER -.映射.-> USER_0
    LOGIC_USER -.映射.-> USER_1

    LOGIC_ORDER -.映射.-> ORDER_0
    LOGIC_ORDER -.映射.-> ORDER_1
    LOGIC_ORDER -.映射.-> ORDER_2
    LOGIC_ORDER -.映射.-> ORDER_3

    style LOGIC_DB fill:#4CAF50
    style LOGIC_USER fill:#4ECDC4
    style LOGIC_ORDER fill:#4ECDC4
    style PHY_DB1 fill:#FFB84D
    style PHY_DB2 fill:#FFB84D
```

### 3. MyCAT 分片规则
```mermaid
graph TB
    subgraph "MyCAT 分片规则配置"
        subgraph "rule.xml - 分片规则"
            RULE1[mod-long<br/>取模分片]
            RULE2[rang-long<br/>范围分片]
            RULE3[hash-int<br/>哈希分片]
            RULE4[sharding-by-date<br/>按日期分片]
            RULE5[sharding-by-pattern<br/>按字符串分片]
        end

        subgraph "schema.xml - 逻辑库表"
            SCHEMA[逻辑库配置]
            TABLE[分片表配置]
            DATANODE[数据节点配置]
            DATAHOST[数据主机配置]
        end

        subgraph "server.xml - 服务配置"
            USER[用户配置]
            CHARSET[字符集配置]
            PROCESSOR[处理器配置]
        end
    end

    TABLE --> RULE1
    TABLE --> DATANODE
    DATANODE --> DATAHOST

    style RULE1 fill:#FFE66D
    style RULE2 fill:#FFE66D
    style RULE3 fill:#FFE66D
    style RULE4 fill:#FFE66D
    style RULE5 fill:#FFE66D
    style SCHEMA fill:#4ECDC4
    style TABLE fill:#4ECDC4
```

## ShardingSphere vs MyCAT 对比

### 架构对比
```mermaid
graph TB
    subgraph "对比分析"
        subgraph "ShardingSphere"
            SS1[JDBC模式<br/>应用内集成]
            SS2[Proxy模式<br/>独立进程]
            SS3[功能丰富<br/>生态完善]
            SS4[支持多种数据库]
            SS5[社区活跃]
        end

        subgraph "MyCAT"
            MC1[独立中间件<br/>Proxy模式]
            MC2[MySQL协议]
            MC3[配置相对简单]
            MC4[老牌稳定]
            MC5[主要支持MySQL]
        end

        subgraph "选型建议"
            C1[新项目: ShardingSphere]
            C2[Java技术栈: JDBC模式]
            C3[多语言: Proxy模式]
            C4[MySQL老项目: MyCAT]
            C5[云原生: ShardingSphere]
        end
    end

    style SS1 fill:#4ECDC4
    style SS2 fill:#4ECDC4
    style SS3 fill:#4ECDC4
    style MC1 fill:#FFB84D
    style MC2 fill:#FFB84D
    style MC3 fill:#FFB84D
    style C1 fill:#4CAF50
    style C2 fill:#4CAF50
    style C3 fill:#4CAF50
    style C4 fill:#4CAF50
    style C5 fill:#4CAF50
```

## 分布式事务解决方案

### 1. ShardingSphere 分布式事务
```mermaid
graph TB
    subgraph "分布式事务类型"
        subgraph "XA事务 - 强一致性"
            XA1[两阶段提交]
            XA2[性能较低]
            XA3[适合短事务]
            XA1 --> XA2 --> XA3
        end

        subgraph "BASE事务 - 最终一致性"
            BASE1[Saga模式]
            BASE2[性能较好]
            BASE3[适合长事务]
            BASE1 --> BASE2 --> BASE3
        end

        subgraph "AT事务 - Seata"
            AT1[自动补偿]
            AT2[无侵入]
            AT3[Seata实现]
            AT1 --> AT2 --> AT3
        end
    end

    style XA1 fill:#FF6B6B
    style BASE1 fill:#4ECDC4
    style AT1 fill:#FFB84D
```

### 2. 分布式事务执行流程
```mermaid
sequenceDiagram
    participant App as 应用
    participant SS as ShardingSphere
    participant DB1 as 数据库1
    participant DB2 as 数据库2

    App->>SS: 1. 开启分布式事务
    SS->>DB1: 2. 执行SQL(分片1)
    SS->>DB2: 3. 执行SQL(分片2)

    alt XA两阶段提交
        SS->>DB1: 4a. Prepare
        SS->>DB2: 4b. Prepare
        DB1->>SS: 5a. Ready
        DB2->>SS: 5b. Ready
        SS->>DB1: 6a. Commit
        SS->>DB2: 6b. Commit
    else Saga补偿
        SS->>DB1: 4a. 正向操作
        SS->>DB2: 4b. 正向操作
        alt 出现异常
            SS->>DB2: 5. 执行补偿
            SS->>DB1: 6. 执行补偿
        end
    end

    SS->>App: 7. 返回结果
```

## 典型应用场景

### 1. 电商订单分库分表
```mermaid
graph TB
    subgraph "业务场景"
        USER[用户下单]
    end

    subgraph "ShardingSphere 路由"
        ROUTER{分片路由<br/>订单ID % 4}
    end

    subgraph "数据库分片"
        DB0[(db_order_0<br/>订单0,4,8...)]
        DB1[(db_order_1<br/>订单1,5,9...)]
        DB2[(db_order_2<br/>订单2,6,10...)]
        DB3[(db_order_3<br/>订单3,7,11...)]
    end

    USER -->|创建订单| ROUTER
    ROUTER --> DB0
    ROUTER --> DB1
    ROUTER --> DB2
    ROUTER --> DB3

    style ROUTER fill:#FFB84D
    style DB0 fill:#4ECDC4
    style DB1 fill:#4ECDC4
    style DB2 fill:#4ECDC4
    style DB3 fill:#4ECDC4
```

### 2. 用户数据水平分片
```mermaid
graph LR
    subgraph "应用"
        APP[用户服务]
    end

    subgraph "数据库中间件"
        MYCAT[MyCAT]
    end

    subgraph "分片存储"
        U0[(user_0<br/>0-999万)]
        U1[(user_1<br/>1000-1999万)]
        U2[(user_2<br/>2000-2999万)]
        U3[(user_3<br/>3000万+)]
    end

    APP --> MYCAT
    MYCAT -->|用户ID范围路由| U0
    MYCAT -->|用户ID范围路由| U1
    MYCAT -->|用户ID范围路由| U2
    MYCAT -->|用户ID范围路由| U3

    style MYCAT fill:#FFB84D
```

## 最佳实践建议

### 1. 分片键选择
- ✅ 选择高频查询字段
- ✅ 数据分布均匀的字段
- ✅ 避免频繁跨分片查询
- ❌ 避免使用时间戳（数据倾斜）

### 2. 全局主键生成
```mermaid
graph LR
    APP[应用] --> GEN{主键生成策略}
    GEN --> SNOW[雪花算法<br/>Snowflake]
    GEN --> UUID[UUID]
    GEN --> SEQ[数据库序列]
    GEN --> REDIS[Redis自增]

    style SNOW fill:#4CAF50
    style UUID fill:#FFB84D
    style SEQ fill:#4ECDC4
    style REDIS fill:#FF6B6B
```

### 3. 跨分片查询优化
- 避免全表扫描
- 使用绑定表（关联表同分片键）
- 广播表（小表冗余到所有分片）
- 数据汇总到ES或数仓

### 4. 扩容策略
- 双倍扩容法（2→4→8）
- 平滑数据迁移
- 使用一致性哈希算法
- 预留扩容空间

### 5. 监控告警
- SQL执行时间监控
- 分片数据倾斜检测
- 连接池使用率
- 慢SQL分析
