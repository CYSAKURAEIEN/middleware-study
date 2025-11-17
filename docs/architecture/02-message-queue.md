# 消息队列架构

## Kafka 架构详解

### 1. Kafka 集群架构
```mermaid
graph TB
    subgraph "生产者集群"
        P1[Producer 1]
        P2[Producer 2]
        P3[Producer 3]
    end

    subgraph "Kafka 集群"
        subgraph "Broker 1"
            T1P0[Topic1-Partition0<br/>Leader]
            T1P1[Topic1-Partition1<br/>Follower]
            T2P0[Topic2-Partition0<br/>Follower]
        end

        subgraph "Broker 2"
            T1P0_F[Topic1-Partition0<br/>Follower]
            T1P1_L[Topic1-Partition1<br/>Leader]
            T2P1[Topic2-Partition1<br/>Leader]
        end

        subgraph "Broker 3"
            T2P0_L[Topic2-Partition0<br/>Leader]
            T2P1_F[Topic2-Partition1<br/>Follower]
            T1P2[Topic1-Partition2<br/>Leader]
        end
    end

    subgraph "Zookeeper 集群"
        ZK[Zookeeper<br/>元数据管理<br/>Leader选举]
    end

    subgraph "消费者组"
        C1[Consumer 1]
        C2[Consumer 2]
        C3[Consumer 3]
    end

    P1 -->|发送消息| T1P0
    P2 -->|发送消息| T1P1_L
    P3 -->|发送消息| T2P0_L

    T1P0 -.复制.-> T1P0_F
    T1P1_L -.复制.-> T1P1
    T2P0_L -.复制.-> T2P0
    T2P1 -.复制.-> T2P1_F

    ZK -.管理.-> T1P0
    ZK -.管理.-> T1P1_L
    ZK -.管理.-> T2P0_L

    T1P0 -->|消费| C1
    T1P1_L -->|消费| C2
    T1P2 -->|消费| C3

    style T1P0 fill:#FFB84D
    style T1P1_L fill:#FFB84D
    style T2P0_L fill:#FFB84D
    style T2P1 fill:#FFB84D
    style T1P2 fill:#FFB84D
    style ZK fill:#4CAF50
```

### 2. Kafka Topic-Partition 模型
```mermaid
graph TB
    subgraph "Kafka Topic: orders"
        subgraph "Partition 0"
            M1[Message 0<br/>offset:0]
            M2[Message 1<br/>offset:1]
            M3[Message 2<br/>offset:2]
            M1 --> M2 --> M3
        end

        subgraph "Partition 1"
            M4[Message 0<br/>offset:0]
            M5[Message 1<br/>offset:1]
            M6[Message 2<br/>offset:2]
            M4 --> M5 --> M6
        end

        subgraph "Partition 2"
            M7[Message 0<br/>offset:0]
            M8[Message 1<br/>offset:1]
            M9[Message 2<br/>offset:2]
            M7 --> M8 --> M9
        end
    end

    subgraph "Consumer Group"
        CG1[Consumer 1<br/>读取 P0]
        CG2[Consumer 2<br/>读取 P1]
        CG3[Consumer 3<br/>读取 P2]
    end

    M3 -.->|消费| CG1
    M6 -.->|消费| CG2
    M9 -.->|消费| CG3

    style M1 fill:#FFE66D
    style M2 fill:#FFE66D
    style M3 fill:#FFE66D
    style M4 fill:#FFE66D
    style M5 fill:#FFE66D
    style M6 fill:#FFE66D
    style M7 fill:#FFE66D
    style M8 fill:#FFE66D
    style M9 fill:#FFE66D
```

### 3. Kafka 数据流转过程
```mermaid
sequenceDiagram
    participant P as Producer
    participant B as Broker-Leader
    participant F as Broker-Follower
    participant C as Consumer

    P->>B: 1. 发送消息
    B->>B: 2. 写入本地日志
    B->>F: 3. 同步数据到Follower
    F->>F: 4. 写入本地日志
    F->>B: 5. 发送ACK
    B->>P: 6. 返回成功响应
    C->>B: 7. 拉取消息(offset)
    B->>C: 8. 返回消息批次
    C->>C: 9. 处理消息
    C->>B: 10. 提交offset
```

## RabbitMQ 架构详解

### 1. RabbitMQ 集群架构
```mermaid
graph TB
    subgraph "生产者"
        P1[Producer 1]
        P2[Producer 2]
    end

    subgraph "RabbitMQ 集群"
        subgraph "节点1"
            E1[Exchange<br/>交换机]
            Q1[Queue 1<br/>队列]
            Q2[Queue 2<br/>队列]
        end

        subgraph "节点2 - 镜像"
            E2[Exchange<br/>镜像]
            Q1M[Queue 1<br/>镜像]
            Q2M[Queue 2<br/>镜像]
        end

        subgraph "节点3 - 镜像"
            E3[Exchange<br/>镜像]
            Q3[Queue 3<br/>队列]
        end
    end

    subgraph "消费者"
        C1[Consumer 1]
        C2[Consumer 2]
        C3[Consumer 3]
    end

    P1 -->|发布| E1
    P2 -->|发布| E1

    E1 -->|路由| Q1
    E1 -->|路由| Q2
    E1 -->|路由| Q3

    Q1 -.同步.-> Q1M
    Q2 -.同步.-> Q2M

    Q1 -->|消费| C1
    Q2 -->|消费| C2
    Q3 -->|消费| C3

    style E1 fill:#FFB84D
    style Q1 fill:#4ECDC4
    style Q2 fill:#4ECDC4
    style Q3 fill:#4ECDC4
```

### 2. RabbitMQ 交换机类型
```mermaid
graph TB
    subgraph "RabbitMQ Exchange Types"
        subgraph "1. Direct Exchange - 直连"
            DE[Direct Exchange]
            DQ1[Queue: error]
            DQ2[Queue: info]
            DQ3[Queue: warning]

            DE -->|routing key: error| DQ1
            DE -->|routing key: info| DQ2
            DE -->|routing key: warning| DQ3
        end

        subgraph "2. Fanout Exchange - 广播"
            FE[Fanout Exchange]
            FQ1[Queue 1]
            FQ2[Queue 2]
            FQ3[Queue 3]

            FE -->|广播| FQ1
            FE -->|广播| FQ2
            FE -->|广播| FQ3
        end

        subgraph "3. Topic Exchange - 主题"
            TE[Topic Exchange]
            TQ1[Queue: *.error.*]
            TQ2[Queue: log.#]
            TQ3[Queue: #.critical]

            TE -->|匹配模式| TQ1
            TE -->|匹配模式| TQ2
            TE -->|匹配模式| TQ3
        end

        subgraph "4. Headers Exchange - 头部"
            HE[Headers Exchange]
            HQ1[Queue: x-match=all]
            HQ2[Queue: x-match=any]

            HE -->|匹配Header| HQ1
            HE -->|匹配Header| HQ2
        end
    end

    style DE fill:#FFB84D
    style FE fill:#FFB84D
    style TE fill:#FFB84D
    style HE fill:#FFB84D
```

### 3. RabbitMQ 消息流转
```mermaid
sequenceDiagram
    participant P as Producer
    participant E as Exchange
    participant Q as Queue
    participant C as Consumer

    P->>E: 1. 发布消息(routing key)
    E->>E: 2. 路由规则匹配
    E->>Q: 3. 消息存入队列
    Q->>Q: 4. 持久化(可选)
    C->>Q: 5. 订阅队列
    Q->>C: 6. 推送消息
    C->>C: 7. 处理消息
    C->>Q: 8. 发送ACK确认
    Q->>Q: 9. 删除消息
```

## RocketMQ 架构详解

### 1. RocketMQ 集群架构
```mermaid
graph TB
    subgraph "生产者集群"
        P1[Producer 1]
        P2[Producer 2]
    end

    subgraph "NameServer 集群"
        NS1[NameServer 1<br/>路由注册]
        NS2[NameServer 2<br/>路由注册]
    end

    subgraph "Broker 集群"
        subgraph "Master-Slave 组1"
            BM1[Broker Master 1]
            BS1[Broker Slave 1]
        end

        subgraph "Master-Slave 组2"
            BM2[Broker Master 2]
            BS2[Broker Slave 2]
        end
    end

    subgraph "消费者集群"
        subgraph "消费者组1"
            C1[Consumer 1]
            C2[Consumer 2]
        end
    end

    P1 -->|1.获取路由| NS1
    P2 -->|1.获取路由| NS2

    P1 -->|2.发送消息| BM1
    P2 -->|2.发送消息| BM2

    BM1 -.心跳注册.-> NS1
    BM2 -.心跳注册.-> NS2

    BM1 -.同步复制.-> BS1
    BM2 -.同步复制.-> BS2

    C1 -->|3.获取路由| NS1
    C2 -->|3.获取路由| NS2

    BM1 -->|4.拉取消息| C1
    BM2 -->|4.拉取消息| C2

    style NS1 fill:#4CAF50
    style NS2 fill:#4CAF50
    style BM1 fill:#FFB84D
    style BM2 fill:#FFB84D
    style BS1 fill:#4ECDC4
    style BS2 fill:#4ECDC4
```

### 2. RocketMQ 消息类型
```mermaid
graph TB
    subgraph "RocketMQ 消息类型"
        subgraph "1. 普通消息"
            NM[Normal Message<br/>普通消息]
            NM1[特点: 异步发送]
            NM2[特点: 高吞吐]
            NM --> NM1
            NM --> NM2
        end

        subgraph "2. 顺序消息"
            OM[Ordered Message<br/>顺序消息]
            OM1[全局顺序]
            OM2[分区顺序]
            OM --> OM1
            OM --> OM2
        end

        subgraph "3. 事务消息"
            TM[Transaction Message<br/>事务消息]
            TM1[Half消息]
            TM2[事务回查]
            TM3[提交/回滚]
            TM --> TM1 --> TM2 --> TM3
        end

        subgraph "4. 延时消息"
            DM[Delay Message<br/>延时消息]
            DM1[18个延时级别]
            DM2[最长2小时]
            DM --> DM1
            DM --> DM2
        end

        subgraph "5. 批量消息"
            BM[Batch Message<br/>批量消息]
            BM1[同一Topic]
            BM2[不支持延时]
            BM --> BM1
            BM --> BM2
        end
    end

    style NM fill:#FFE66D
    style OM fill:#FFB84D
    style TM fill:#FF6B6B
    style DM fill:#4ECDC4
    style BM fill:#A78BFA
```

### 3. RocketMQ 事务消息流程
```mermaid
sequenceDiagram
    participant P as Producer
    participant B as Broker
    participant C as Consumer
    participant DB as Database

    P->>B: 1. 发送Half消息
    B->>P: 2. Half消息发送成功
    P->>DB: 3. 执行本地事务
    DB->>P: 4. 本地事务结果

    alt 本地事务成功
        P->>B: 5a. Commit消息
        B->>C: 6a. 投递消息给消费者
    else 本地事务失败
        P->>B: 5b. Rollback消息
        B->>B: 6b. 删除Half消息
    else 未知状态
        B->>P: 5c. 事务状态回查
        P->>DB: 6c. 查询事务状态
        DB->>P: 7c. 返回状态
        P->>B: 8c. Commit/Rollback
    end
```

## 三大消息队列对比

### 架构对比
```mermaid
graph TB
    subgraph "对比维度"
        subgraph "Kafka - 高吞吐"
            K1[分区顺序消费]
            K2[批量拉取]
            K3[零拷贝技术]
            K4[适合日志收集]
        end

        subgraph "RabbitMQ - 可靠性"
            R1[多种交换机]
            R2[消息确认机制]
            R3[灵活路由]
            R4[适合业务消息]
        end

        subgraph "RocketMQ - 均衡"
            RM1[事务消息]
            RM2[延时消息]
            RM3[高可用架构]
            RM4[适合电商场景]
        end
    end

    style K1 fill:#FFB84D
    style K2 fill:#FFB84D
    style K3 fill:#FFB84D
    style K4 fill:#FFB84D
    style R1 fill:#4ECDC4
    style R2 fill:#4ECDC4
    style R3 fill:#4ECDC4
    style R4 fill:#4ECDC4
    style RM1 fill:#A78BFA
    style RM2 fill:#A78BFA
    style RM3 fill:#A78BFA
    style RM4 fill:#A78BFA
```

## 典型应用场景

### 1. 异步解耦场景
```mermaid
graph LR
    ORDER[订单服务] -->|发送消息| MQ[消息队列]
    MQ -->|异步消费| INVENTORY[库存服务]
    MQ -->|异步消费| LOGISTICS[物流服务]
    MQ -->|异步消费| NOTIFY[通知服务]

    style MQ fill:#FFB84D
```

### 2. 流量削峰场景
```mermaid
graph TB
    subgraph "秒杀场景"
        USER[大量用户请求<br/>10万QPS]
        MQ[消息队列<br/>缓冲]
        PROCESS[订单处理<br/>1000QPS]
    end

    USER -->|写入队列| MQ
    MQ -->|平稳消费| PROCESS

    style MQ fill:#FFB84D
```

### 3. 日志收集场景
```mermaid
graph LR
    APP1[应用1] -->|日志| KAFKA[Kafka]
    APP2[应用2] -->|日志| KAFKA
    APP3[应用3] -->|日志| KAFKA

    KAFKA -->|消费| ES[Elasticsearch]
    KAFKA -->|消费| HDFS[HDFS]

    style KAFKA fill:#FFB84D
```

## 选型建议

| 场景 | 推荐MQ | 理由 |
|------|--------|------|
| 日志收集、大数据 | Kafka | 超高吞吐量，批量处理 |
| 金融交易、订单 | RabbitMQ | 消息可靠性高，支持事务 |
| 电商业务 | RocketMQ | 功能全面，支持事务和延时 |
| 实时计算 | Kafka | 流式处理，低延迟 |
| 复杂路由 | RabbitMQ | 多种交换机类型 |

## 最佳实践

### 1. 消息幂等性处理
- 使用全局唯一消息ID
- 业务侧去重表
- 利用数据库唯一约束

### 2. 消息顺序性保证
- Kafka: 单分区内有序
- RabbitMQ: 单队列有序
- RocketMQ: 使用MessageQueueSelector

### 3. 消息可靠性保证
- 生产者确认机制（ACK）
- 消息持久化
- 消费者手动确认
- 死信队列处理失败消息

### 4. 性能优化
- 批量发送和消费
- 异步发送
- 合理设置分区数
- 消费者并发度调优
