# 服务治理架构

## Nacos 架构详解

### 1. Nacos 整体架构
```mermaid
graph TB
    subgraph "应用层 - 微服务"
        MS1[订单服务]
        MS2[用户服务]
        MS3[商品服务]
        MS4[支付服务]
    end

    subgraph "Nacos Server 集群"
        subgraph "核心功能模块"
            NAMING[服务注册与发现<br/>Naming Service]
            CONFIG[配置管理<br/>Config Service]
            METADATA[元数据管理]
            HEALTH[健康检查]
        end

        subgraph "数据存储"
            MEMORY[内存<br/>服务实例]
            MYSQL[(MySQL<br/>配置持久化)]
            RAFT[Raft协议<br/>集群一致性]
        end

        NAMING --> MEMORY
        CONFIG --> MYSQL
        NAMING --> RAFT
        CONFIG --> RAFT
    end

    subgraph "客户端 SDK"
        SDK1[Nacos Client<br/>服务注册]
        SDK2[Nacos Client<br/>配置订阅]
    end

    MS1 --> SDK1
    MS2 --> SDK1
    MS3 --> SDK1
    MS4 --> SDK1

    MS1 --> SDK2
    MS2 --> SDK2

    SDK1 -->|注册/心跳| NAMING
    SDK2 -->|拉取配置| CONFIG

    NAMING -->|服务发现| SDK1
    CONFIG -->|推送变更| SDK2

    style NAMING fill:#4CAF50
    style CONFIG fill:#4ECDC4
    style METADATA fill:#FFB84D
    style HEALTH fill:#A78BFA
```

### 2. Nacos 服务注册与发现
```mermaid
sequenceDiagram
    participant MS as 微服务
    participant NC as Nacos Client
    participant NS as Nacos Server
    participant DB as 注册表

    MS->>NC: 1. 启动应用
    NC->>NS: 2. 服务注册(serviceName, ip, port)
    NS->>DB: 3. 保存服务实例
    NS->>NC: 4. 注册成功

    loop 心跳保活
        NC->>NS: 5. 发送心跳(每5秒)
        NS->>NS: 6. 更新最后心跳时间
    end

    MS->>NC: 7. 调用其他服务
    NC->>NS: 8. 服务发现(订阅服务列表)
    NS->>NC: 9. 返回服务实例列表
    NC->>NC: 10. 本地缓存 + 负载均衡
    NC->>MS: 11. 返回目标服务地址

    Note over NS,DB: 健康检查：超15秒未心跳标记不健康<br/>超30秒剔除实例
```

### 3. Nacos 配置中心
```mermaid
graph TB
    subgraph "配置管理流程"
        subgraph "配置发布"
            PUB1[开发修改配置]
            PUB2[Nacos控制台]
            PUB3[配置存储MySQL]
            PUB4[配置版本管理]

            PUB1 --> PUB2 --> PUB3 --> PUB4
        end

        subgraph "配置订阅"
            SUB1[应用启动]
            SUB2[拉取配置]
            SUB3[本地缓存]
            SUB4[长轮询监听]

            SUB1 --> SUB2 --> SUB3 --> SUB4
        end

        subgraph "配置推送"
            PUSH1[配置变更]
            PUSH2[推送通知]
            PUSH3[应用接收]
            PUSH4[动态刷新]

            PUSH1 --> PUSH2 --> PUSH3 --> PUSH4
        end
    end

    PUB4 -.触发.-> PUSH1
    SUB4 -.监听.-> PUSH2

    style PUB2 fill:#4CAF50
    style SUB3 fill:#4ECDC4
    style PUSH4 fill:#FFB84D
```

### 4. Nacos 命名空间与分组
```mermaid
graph TB
    subgraph "多环境隔离"
        subgraph "命名空间: dev"
            DEV_GROUP1[分组: DEFAULT_GROUP]
            DEV_GROUP2[分组: PAYMENT_GROUP]

            DEV_SERVICE1[订单服务]
            DEV_SERVICE2[支付服务]

            DEV_GROUP1 --> DEV_SERVICE1
            DEV_GROUP2 --> DEV_SERVICE2
        end

        subgraph "命名空间: test"
            TEST_GROUP1[分组: DEFAULT_GROUP]
            TEST_SERVICE1[订单服务]

            TEST_GROUP1 --> TEST_SERVICE1
        end

        subgraph "命名空间: prod"
            PROD_GROUP1[分组: DEFAULT_GROUP]
            PROD_GROUP2[分组: PAYMENT_GROUP]

            PROD_SERVICE1[订单服务]
            PROD_SERVICE2[支付服务]

            PROD_GROUP1 --> PROD_SERVICE1
            PROD_GROUP2 --> PROD_SERVICE2
        end
    end

    style DEV_GROUP1 fill:#A78BFA
    style TEST_GROUP1 fill:#4ECDC4
    style PROD_GROUP1 fill:#FF6B6B
```

## Consul 架构详解

### 1. Consul 集群架构
```mermaid
graph TB
    subgraph "应用层"
        APP1[应用1]
        APP2[应用2]
        APP3[应用3]
    end

    subgraph "Consul Server 集群"
        subgraph "数据中心 DC1"
            LEADER[Consul Server<br/>Leader]
            FOLLOWER1[Consul Server<br/>Follower]
            FOLLOWER2[Consul Server<br/>Follower]

            LEADER <-.Raft共识.-> FOLLOWER1
            LEADER <-.Raft共识.-> FOLLOWER2
        end
    end

    subgraph "Consul Agent 客户端"
        AGENT1[Consul Agent<br/>节点1]
        AGENT2[Consul Agent<br/>节点2]
        AGENT3[Consul Agent<br/>节点3]
    end

    subgraph "核心功能"
        SD[服务发现]
        HC[健康检查]
        KV[键值存储]
        SC[服务网格]
    end

    APP1 --> AGENT1
    APP2 --> AGENT2
    APP3 --> AGENT3

    AGENT1 <-->|Gossip协议| LEADER
    AGENT2 <-->|Gossip协议| FOLLOWER1
    AGENT3 <-->|Gossip协议| FOLLOWER2

    LEADER --> SD
    LEADER --> HC
    LEADER --> KV
    LEADER --> SC

    style LEADER fill:#FF6B6B
    style FOLLOWER1 fill:#4ECDC4
    style FOLLOWER2 fill:#4ECDC4
    style SD fill:#FFB84D
    style HC fill:#FFB84D
    style KV fill:#FFB84D
    style SC fill:#FFB84D
```

### 2. Consul 服务注册与健康检查
```mermaid
sequenceDiagram
    participant S as 服务
    participant A as Consul Agent
    participant C as Consul Server

    S->>A: 1. 注册服务(HTTP/DNS)
    A->>C: 2. 转发注册信息
    C->>C: 3. Raft同步到集群
    C->>A: 4. 注册成功

    loop 健康检查
        A->>S: 5. HTTP/TCP/Script检查
        S->>A: 6. 返回健康状态
        A->>C: 7. 上报健康状态
    end

    Note over A,C: 健康检查失败后<br/>服务自动从注册表移除

    S->>A: 8. 服务发现请求
    A->>C: 9. 查询服务列表
    C->>A: 10. 返回健康服务实例
    A->>S: 11. 返回可用服务
```

### 3. Consul 多数据中心
```mermaid
graph LR
    subgraph "数据中心 DC1 - 北京"
        DC1_S1[Consul Server]
        DC1_S2[Consul Server]
        DC1_S3[Consul Server]
        DC1_APP[应用集群1]

        DC1_APP --> DC1_S1
    end

    subgraph "数据中心 DC2 - 上海"
        DC2_S1[Consul Server]
        DC2_S2[Consul Server]
        DC2_APP[应用集群2]

        DC2_APP --> DC2_S1
    end

    subgraph "数据中心 DC3 - 深圳"
        DC3_S1[Consul Server]
        DC3_APP[应用集群3]

        DC3_APP --> DC3_S1
    end

    DC1_S1 <-.WAN Gossip.-> DC2_S1
    DC2_S1 <-.WAN Gossip.-> DC3_S1
    DC3_S1 <-.WAN Gossip.-> DC1_S1

    style DC1_S1 fill:#FF6B6B
    style DC2_S1 fill:#4ECDC4
    style DC3_S1 fill:#FFB84D
```

## Zookeeper 架构详解

### 1. Zookeeper 集群架构
```mermaid
graph TB
    subgraph "客户端"
        CLIENT1[应用客户端1]
        CLIENT2[应用客户端2]
        CLIENT3[应用客户端3]
    end

    subgraph "Zookeeper 集群 - 3节点"
        LEADER[ZK Leader<br/>处理写请求]
        FOLLOWER1[ZK Follower<br/>处理读请求]
        FOLLOWER2[ZK Follower<br/>处理读请求]
    end

    subgraph "数据模型"
        ZNODE[ZNode树形结构]

        subgraph "节点类型"
            PERSIST[持久节点<br/>Persistent]
            EPHEMERAL[临时节点<br/>Ephemeral]
            SEQUENCE[顺序节点<br/>Sequential]
        end

        ZNODE --> PERSIST
        ZNODE --> EPHEMERAL
        ZNODE --> SEQUENCE
    end

    CLIENT1 -->|读写| LEADER
    CLIENT2 -->|读| FOLLOWER1
    CLIENT3 -->|读| FOLLOWER2

    LEADER <-.ZAB协议同步.-> FOLLOWER1
    LEADER <-.ZAB协议同步.-> FOLLOWER2

    LEADER --> ZNODE
    FOLLOWER1 --> ZNODE
    FOLLOWER2 --> ZNODE

    style LEADER fill:#FF6B6B
    style FOLLOWER1 fill:#4ECDC4
    style FOLLOWER2 fill:#4ECDC4
    style PERSIST fill:#FFE66D
    style EPHEMERAL fill:#FFB84D
    style SEQUENCE fill:#A78BFA
```

### 2. Zookeeper 数据模型
```mermaid
graph TB
    ROOT[/ 根节点]

    ROOT --> SERVICES[/services]
    ROOT --> CONFIG[/config]
    ROOT --> LOCKS[/locks]

    SERVICES --> ORDER[/order-service]
    SERVICES --> USER[/user-service]

    ORDER --> INSTANCE1[/instance-001<br/>临时顺序节点]
    ORDER --> INSTANCE2[/instance-002<br/>临时顺序节点]

    USER --> INSTANCE3[/instance-001<br/>临时顺序节点]

    CONFIG --> DB[/database<br/>持久节点]
    CONFIG --> REDIS[/redis<br/>持久节点]

    LOCKS --> LOCK1[/lock-0000000001<br/>临时顺序节点]
    LOCKS --> LOCK2[/lock-0000000002<br/>临时顺序节点]

    style ROOT fill:#4CAF50
    style SERVICES fill:#4ECDC4
    style CONFIG fill:#FFB84D
    style LOCKS fill:#FF6B6B
```

### 3. Zookeeper 服务注册与发现
```mermaid
sequenceDiagram
    participant S as 服务实例
    participant ZK as Zookeeper
    participant C as 消费者

    S->>ZK: 1. 创建临时节点<br/>/services/order/instance-001
    ZK->>ZK: 2. 创建成功
    ZK->>S: 3. 返回节点路径

    C->>ZK: 4. 订阅服务<br/>watch /services/order
    ZK->>C: 5. 返回子节点列表

    Note over S,ZK: 服务保持会话心跳

    alt 服务宕机
        S--xZK: 6. 会话超时
        ZK->>ZK: 7. 删除临时节点
        ZK->>C: 8. 触发Watch通知
        C->>ZK: 9. 重新获取服务列表
    end
```

### 4. Zookeeper Watch 机制
```mermaid
graph TB
    subgraph "Watch 监听机制"
        subgraph "客户端操作"
            C1[设置Watch]
            C2[接收通知]
            C3[重新设置Watch]

            C1 --> C2 --> C3
        end

        subgraph "服务端触发"
            S1[节点数据变更]
            S2[子节点变更]
            S3[节点删除]

            S1 -.触发.-> C2
            S2 -.触发.-> C2
            S3 -.触发.-> C2
        end

        subgraph "Watch特性"
            F1[一次性触发]
            F2[轻量级通知]
            F3[顺序保证]
        end
    end

    style C1 fill:#4ECDC4
    style S1 fill:#FFB84D
    style S2 fill:#FFB84D
    style S3 fill:#FFB84D
    style F1 fill:#FF6B6B
```

## 三大服务治理组件对比

### 功能对比
```mermaid
graph TB
    subgraph "对比分析"
        subgraph "Nacos - 阿里巴巴"
            N1[服务注册发现]
            N2[配置管理]
            N3[服务健康检查]
            N4[多种协议支持]
            N5[动态DNS服务]
            N6[适合Spring Cloud]
        end

        subgraph "Consul - HashiCorp"
            C1[服务注册发现]
            C2[健康检查]
            C3[KV存储]
            C4[多数据中心]
            C5[服务网格]
            C6[适合多语言]
        end

        subgraph "Zookeeper - Apache"
            Z1[分布式协调]
            Z2[配置管理]
            Z3[命名服务]
            Z4[分布式锁]
            Z5[集群管理]
            Z6[老牌稳定]
        end
    end

    style N1 fill:#4CAF50
    style N2 fill:#4CAF50
    style C1 fill:#4ECDC4
    style C5 fill:#4ECDC4
    style Z1 fill:#FFB84D
    style Z4 fill:#FFB84D
```

### CAP 理论定位
```mermaid
graph TB
    subgraph "CAP 理论"
        CAP[CAP定理]

        C[Consistency<br/>一致性]
        A[Availability<br/>可用性]
        P[Partition tolerance<br/>分区容错性]

        CAP --> C
        CAP --> A
        CAP --> P
    end

    subgraph "组件定位"
        NACOS[Nacos<br/>AP架构<br/>也支持CP]
        CONSUL[Consul<br/>CP架构<br/>强一致性]
        ZK[Zookeeper<br/>CP架构<br/>强一致性]
    end

    C -.-> CONSUL
    C -.-> ZK
    A -.-> NACOS
    P -.-> NACOS
    P -.-> CONSUL
    P -.-> ZK

    style C fill:#FF6B6B
    style A fill:#4CAF50
    style P fill:#4ECDC4
    style NACOS fill:#FFB84D
    style CONSUL fill:#A78BFA
    style ZK fill:#FFE66D
```

## 典型应用场景

### 1. 微服务架构
```mermaid
graph TB
    subgraph "微服务生态"
        API[API Gateway]

        subgraph "业务服务"
            ORDER[订单服务]
            USER[用户服务]
            PRODUCT[商品服务]
            PAYMENT[支付服务]
        end
    end

    subgraph "Nacos 服务治理"
        NAMING[服务注册发现]
        CONFIG[配置中心]
        HEALTH[健康监控]
    end

    ORDER --> NAMING
    USER --> NAMING
    PRODUCT --> NAMING
    PAYMENT --> NAMING

    ORDER --> CONFIG
    USER --> CONFIG

    NAMING --> HEALTH

    API -->|服务发现| NAMING
    API -->|调用| ORDER
    API -->|调用| USER

    style NAMING fill:#4CAF50
    style CONFIG fill:#4ECDC4
```

### 2. 配置动态刷新
```mermaid
sequenceDiagram
    participant DEV as 开发人员
    participant NACOS as Nacos配置中心
    participant APP as 应用实例

    DEV->>NACOS: 1. 修改配置(数据库连接池)
    NACOS->>NACOS: 2. 保存配置并触发变更
    NACOS->>APP: 3. 推送配置变更通知
    APP->>APP: 4. 接收新配置
    APP->>APP: 5. 动态刷新Bean
    APP->>NACOS: 6. 确认配置更新

    Note over APP: 无需重启应用<br/>配置实时生效
```

### 3. 灰度发布场景
```mermaid
graph LR
    subgraph "用户请求"
        USER[用户]
    end

    subgraph "网关层"
        GATEWAY{API Gateway<br/>灰度路由}
    end

    subgraph "订单服务集群"
        subgraph "稳定版本 v1.0"
            V1_1[实例1<br/>90%流量]
            V1_2[实例2<br/>90%流量]
        end

        subgraph "灰度版本 v2.0"
            V2_1[实例3<br/>10%流量]
        end
    end

    subgraph "Nacos"
        META[元数据标签<br/>version=v1.0/v2.0]
    end

    USER --> GATEWAY

    GATEWAY -->|90%| V1_1
    GATEWAY -->|90%| V1_2
    GATEWAY -->|10%| V2_1

    V1_1 -.注册.-> META
    V1_2 -.注册.-> META
    V2_1 -.注册.-> META

    style V1_1 fill:#4CAF50
    style V1_2 fill:#4CAF50
    style V2_1 fill:#FFB84D
```

## 最佳实践建议

### 1. 选型建议

| 场景 | 推荐组件 | 理由 |
|------|---------|------|
| Spring Cloud 微服务 | Nacos | 与生态深度集成，功能全面 |
| 多语言微服务 | Consul | 支持多语言，HTTP API |
| 分布式协调、锁 | Zookeeper | 成熟稳定，CP架构 |
| 需要配置中心 | Nacos | 配置管理功能强大 |
| 服务网格 | Consul | 原生支持Service Mesh |
| 老系统改造 | Zookeeper | 兼容性好，稳定 |

### 2. Nacos 最佳实践
- 合理划分命名空间（dev/test/prod）
- 使用分组隔离不同业务模块
- 配置文件使用 YAML 格式，便于管理
- 开启鉴权，保证安全性
- 集群部署至少3个节点

### 3. Consul 最佳实践
- 服务端奇数个节点（3、5、7）
- 每个节点部署 Agent
- 配置多种健康检查（HTTP + TCP）
- 多数据中心部署，实现异地容灾
- 使用 ACL 进行权限控制

### 4. Zookeeper 最佳实践
- 集群节点数建议为奇数（3、5、7）
- 避免频繁创建删除节点
- 控制节点数据大小（<1MB）
- 及时清理无用的临时节点
- 使用独立磁盘存放事务日志

### 5. 监控告警
- 服务注册数量监控
- 心跳超时告警
- 配置变更审计
- 集群健康状态
- 网络延迟监控
