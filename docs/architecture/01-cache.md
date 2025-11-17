# 缓存层架构

## Redis 架构详解

### 1. Redis 单机架构
```mermaid
graph TB
    subgraph "应用层"
        APP1[应用实例1]
        APP2[应用实例2]
        APP3[应用实例3]
    end

    subgraph "Redis 单机"
        REDIS[Redis Server<br/>端口:6379]
        subgraph "数据结构"
            STRING[String<br/>字符串]
            HASH[Hash<br/>哈希表]
            LIST[List<br/>列表]
            SET[Set<br/>集合]
            ZSET[ZSet<br/>有序集合]
        end
        REDIS --> STRING
        REDIS --> HASH
        REDIS --> LIST
        REDIS --> SET
        REDIS --> ZSET
    end

    APP1 -->|读写| REDIS
    APP2 -->|读写| REDIS
    APP3 -->|读写| REDIS

    style REDIS fill:#FF6B6B
    style STRING fill:#FFE66D
    style HASH fill:#FFE66D
    style LIST fill:#FFE66D
    style SET fill:#FFE66D
    style ZSET fill:#FFE66D
```

### 2. Redis 主从复制架构
```mermaid
graph TB
    subgraph "应用层"
        WRITE[写操作]
        READ1[读操作1]
        READ2[读操作2]
        READ3[读操作3]
    end

    subgraph "Redis 集群"
        MASTER[Redis Master<br/>主节点<br/>端口:6379]
        SLAVE1[Redis Slave1<br/>从节点<br/>端口:6380]
        SLAVE2[Redis Slave2<br/>从节点<br/>端口:6381]
        SLAVE3[Redis Slave3<br/>从节点<br/>端口:6382]
    end

    WRITE -->|写入| MASTER
    READ1 -->|读取| SLAVE1
    READ2 -->|读取| SLAVE2
    READ3 -->|读取| SLAVE3

    MASTER -.异步复制.-> SLAVE1
    MASTER -.异步复制.-> SLAVE2
    MASTER -.异步复制.-> SLAVE3

    style MASTER fill:#FF6B6B
    style SLAVE1 fill:#4ECDC4
    style SLAVE2 fill:#4ECDC4
    style SLAVE3 fill:#4ECDC4
```

### 3. Redis Sentinel 高可用架构
```mermaid
graph TB
    subgraph "客户端"
        CLIENT[应用客户端]
    end

    subgraph "Sentinel 监控集群"
        S1[Sentinel 1<br/>端口:26379]
        S2[Sentinel 2<br/>端口:26380]
        S3[Sentinel 3<br/>端口:26381]
    end

    subgraph "Redis 主从集群"
        MASTER[Redis Master]
        SLAVE1[Redis Slave1]
        SLAVE2[Redis Slave2]
    end

    CLIENT -->|1.获取主节点地址| S1
    CLIENT -->|2.读写数据| MASTER

    S1 -.监控.-> MASTER
    S2 -.监控.-> MASTER
    S3 -.监控.-> MASTER

    S1 -.监控.-> SLAVE1
    S2 -.监控.-> SLAVE2

    S1 <-.互相监控.-> S2
    S2 <-.互相监控.-> S3
    S3 <-.互相监控.-> S1

    MASTER -.复制.-> SLAVE1
    MASTER -.复制.-> SLAVE2

    style MASTER fill:#FF6B6B
    style SLAVE1 fill:#4ECDC4
    style SLAVE2 fill:#4ECDC4
    style S1 fill:#A78BFA
    style S2 fill:#A78BFA
    style S3 fill:#A78BFA
```

### 4. Redis Cluster 分片架构
```mermaid
graph TB
    subgraph "应用层"
        APP[应用客户端<br/>Smart Client]
    end

    subgraph "Redis Cluster - 槽位分配"
        subgraph "节点1组 - 槽位 0-5460"
            M1[Master 1]
            S1[Slave 1]
        end

        subgraph "节点2组 - 槽位 5461-10922"
            M2[Master 2]
            S2[Slave 2]
        end

        subgraph "节点3组 - 槽位 10923-16383"
            M3[Master 3]
            S3[Slave 3]
        end
    end

    APP -->|Hash Slot路由| M1
    APP -->|Hash Slot路由| M2
    APP -->|Hash Slot路由| M3

    M1 -.复制.-> S1
    M2 -.复制.-> S2
    M3 -.复制.-> S3

    M1 <-.Gossip协议.-> M2
    M2 <-.Gossip协议.-> M3
    M3 <-.Gossip协议.-> M1

    style M1 fill:#FF6B6B
    style M2 fill:#FF6B6B
    style M3 fill:#FF6B6B
    style S1 fill:#4ECDC4
    style S2 fill:#4ECDC4
    style S3 fill:#4ECDC4
```

## Memcached 架构详解

### 1. Memcached 集群架构
```mermaid
graph TB
    subgraph "应用层"
        APP1[应用实例1]
        APP2[应用实例2]
        APP3[应用实例3]
    end

    subgraph "客户端路由层"
        HASH[一致性哈希算法<br/>Consistent Hashing]
    end

    subgraph "Memcached 集群"
        MC1[Memcached 1<br/>端口:11211]
        MC2[Memcached 2<br/>端口:11211]
        MC3[Memcached 3<br/>端口:11211]
        MC4[Memcached 4<br/>端口:11211]
    end

    APP1 --> HASH
    APP2 --> HASH
    APP3 --> HASH

    HASH -->|分布式路由| MC1
    HASH -->|分布式路由| MC2
    HASH -->|分布式路由| MC3
    HASH -->|分布式路由| MC4

    style MC1 fill:#FFB84D
    style MC2 fill:#FFB84D
    style MC3 fill:#FFB84D
    style MC4 fill:#FFB84D
    style HASH fill:#A78BFA
```

### 2. Memcached 内存结构
```mermaid
graph TB
    subgraph "Memcached Server"
        subgraph "Slab分配器"
            SLAB1[Slab Class 1<br/>小对象 96B]
            SLAB2[Slab Class 2<br/>中对象 120B]
            SLAB3[Slab Class 3<br/>大对象 152B]
            SLABN[Slab Class N<br/>超大对象]
        end

        subgraph "LRU淘汰策略"
            LRU[LRU链表<br/>最近最少使用]
        end

        HASH_TABLE[哈希表<br/>快速查找]
    end

    HASH_TABLE --> SLAB1
    HASH_TABLE --> SLAB2
    HASH_TABLE --> SLAB3
    HASH_TABLE --> SLABN

    SLAB1 --> LRU
    SLAB2 --> LRU
    SLAB3 --> LRU
    SLABN --> LRU

    style SLAB1 fill:#FFE66D
    style SLAB2 fill:#FFE66D
    style SLAB3 fill:#FFE66D
    style SLABN fill:#FFE66D
    style LRU fill:#FF6B6B
    style HASH_TABLE fill:#4ECDC4
```

## Redis vs Memcached 对比

### 缓存穿透、击穿、雪崩解决方案

```mermaid
graph TB
    subgraph "缓存问题与解决方案"
        subgraph "1. 缓存穿透"
            CP1[问题: 查询不存在的数据]
            CP2[解决: 布隆过滤器]
            CP3[解决: 空值缓存]
        end

        subgraph "2. 缓存击穿"
            CB1[问题: 热点key过期]
            CB2[解决: 互斥锁]
            CB3[解决: 永不过期]
        end

        subgraph "3. 缓存雪崩"
            CA1[问题: 大量key同时过期]
            CA2[解决: 过期时间随机化]
            CA3[解决: 多级缓存]
        end
    end

    style CP1 fill:#FF6B6B
    style CB1 fill:#FF6B6B
    style CA1 fill:#FF6B6B
    style CP2 fill:#4CAF50
    style CP3 fill:#4CAF50
    style CB2 fill:#4CAF50
    style CB3 fill:#4CAF50
    style CA2 fill:#4CAF50
    style CA3 fill:#4CAF50
```

## 典型应用场景

### 1. 数据库缓存架构
```mermaid
graph LR
    APP[应用] -->|1.查询| REDIS[Redis缓存]
    REDIS -->|2.缓存未命中| APP
    APP -->|3.查询数据库| DB[(MySQL)]
    DB -->|4.返回数据| APP
    APP -->|5.写入缓存| REDIS
    APP -->|6.返回结果| USER[用户]

    style REDIS fill:#FF6B6B
    style DB fill:#4ECDC4
```

### 2. 会话存储
```mermaid
graph TB
    USER[用户登录] -->|1.认证| APP[应用服务器]
    APP -->|2.生成Session| REDIS[Redis]
    REDIS -->|3.返回SessionID| APP
    APP -->|4.设置Cookie| USER

    USER2[用户请求] -->|5.携带SessionID| APP2[应用服务器]
    APP2 -->|6.验证Session| REDIS
    REDIS -->|7.返回用户信息| APP2
    APP2 -->|8.处理请求| USER2

    style REDIS fill:#FF6B6B
```

## 最佳实践建议

### Redis 使用场景
- ✅ 数据结构丰富的场景（List、Set、ZSet等）
- ✅ 需要持久化的缓存数据
- ✅ 发布订阅模式
- ✅ 分布式锁
- ✅ 高可用要求（支持主从、Sentinel、Cluster）

### Memcached 使用场景
- ✅ 简单的 Key-Value 缓存
- ✅ 纯内存缓存，不需要持久化
- ✅ 多线程模型，适合多核 CPU
- ✅ 更少的内存开销

### 性能优化建议
1. **合理设置过期时间**：避免内存溢出
2. **使用连接池**：减少连接开销
3. **避免大 Key**：影响性能和内存
4. **批量操作**：使用 pipeline 或 mget/mset
5. **监控告警**：内存使用率、命中率、慢查询
