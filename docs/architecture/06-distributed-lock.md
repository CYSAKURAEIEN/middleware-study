# 分布式锁架构

## 分布式锁概述

### 1. 分布式锁应用场景
```mermaid
graph TB
    subgraph "为什么需要分布式锁"
        subgraph "单机环境"
            SINGLE[单JVM进程]
            LOCAL[本地锁<br/>synchronized/Lock]
            SINGLE --> LOCAL
        end

        subgraph "分布式环境"
            MULTI[多个JVM进程]
            DIST[分布式锁]
            MULTI --> DIST
        end

        subgraph "典型场景"
            S1[库存扣减<br/>防止超卖]
            S2[订单号生成<br/>保证唯一]
            S3[定时任务<br/>避免重复执行]
            S4[缓存重建<br/>防止击穿]
        end
    end

    DIST --> S1
    DIST --> S2
    DIST --> S3
    DIST --> S4

    style LOCAL fill:#FFE66D
    style DIST fill:#4CAF50
    style S1 fill:#FF6B6B
    style S2 fill:#FFB84D
    style S3 fill:#4ECDC4
    style S4 fill:#A78BFA
```

### 2. 分布式锁的特性要求
```mermaid
graph TB
    subgraph "分布式锁必备特性"
        LOCK[分布式锁]

        subgraph "基本特性"
            MUTUAL[互斥性<br/>同一时刻只有一个客户端持有锁]
            DEADLOCK[防死锁<br/>必须有超时机制]
            REENTRANT[可重入<br/>同一线程可多次获取]
        end

        subgraph "高级特性"
            AVAILABLE[高可用<br/>避免单点故障]
            PERFORM[高性能<br/>加锁解锁快速]
            BLOCKING[阻塞/非阻塞<br/>支持tryLock]
        end

        subgraph "安全特性"
            OWNER[锁归属<br/>只能由加锁者解锁]
            FAIR[公平性<br/>按申请顺序获取]
        end
    end

    LOCK --> MUTUAL
    LOCK --> DEADLOCK
    LOCK --> REENTRANT
    LOCK --> AVAILABLE
    LOCK --> PERFORM
    LOCK --> BLOCKING
    LOCK --> OWNER
    LOCK --> FAIR

    style MUTUAL fill:#FF6B6B
    style DEADLOCK fill:#FF6B6B
    style OWNER fill:#FF6B6B
    style AVAILABLE fill:#4CAF50
    style PERFORM fill:#4CAF50
```

## Redisson 分布式锁

### 1. Redisson 整体架构
```mermaid
graph TB
    subgraph "应用层"
        APP1[应用实例1]
        APP2[应用实例2]
        APP3[应用实例3]
    end

    subgraph "Redisson 客户端"
        CLIENT[Redisson Client]

        subgraph "锁类型"
            LOCK[RLock<br/>可重入锁]
            FAIR[RFairLock<br/>公平锁]
            MULTI[MultiLock<br/>联锁]
            READWRITE[RReadWriteLock<br/>读写锁]
            SEMAPHORE[RSemaphore<br/>信号量]
            COUNTDOWNLATCH[RCountDownLatch<br/>闭锁]
        end

        CLIENT --> LOCK
        CLIENT --> FAIR
        CLIENT --> MULTI
        CLIENT --> READWRITE
        CLIENT --> SEMAPHORE
        CLIENT --> COUNTDOWNLATCH
    end

    subgraph "Redis 集群"
        subgraph "主从模式"
            MASTER[Redis Master]
            SLAVE1[Slave 1]
            SLAVE2[Slave 2]
        end

        MASTER -.复制.-> SLAVE1
        MASTER -.复制.-> SLAVE2
    end

    APP1 --> CLIENT
    APP2 --> CLIENT
    APP3 --> CLIENT

    CLIENT -->|Lua脚本| MASTER

    style LOCK fill:#4CAF50
    style FAIR fill:#4ECDC4
    style MULTI fill:#FFB84D
    style READWRITE fill:#A78BFA
    style MASTER fill:#FF6B6B
```

### 2. Redisson 可重入锁原理
```mermaid
sequenceDiagram
    participant T as 线程
    participant R as Redisson
    participant Redis as Redis

    T->>R: 1. lock() 加锁请求
    R->>Redis: 2. 执行Lua脚本(SETNX)

    alt 锁不存在
        Redis->>Redis: 3a. 创建Hash结构<br/>field: threadId<br/>value: 1
        Redis->>Redis: 4a. 设置过期时间30s
        Redis->>R: 5a. 加锁成功
        R->>T: 6a. 获得锁
    else 锁已存在且是当前线程
        Redis->>Redis: 3b. 重入次数+1
        Redis->>R: 5b. 加锁成功
        R->>T: 6b. 获得锁(可重入)
    else 锁被其他线程持有
        Redis->>R: 3c. 加锁失败
        R->>R: 4c. 订阅解锁消息
        R->>R: 5c. 自旋等待
    end

    Note over R,Redis: WatchDog看门狗机制<br/>每10s自动续期到30s

    T->>R: 7. unlock() 解锁
    R->>Redis: 8. 执行Lua脚本
    Redis->>Redis: 9. 重入次数-1
    alt 重入次数=0
        Redis->>Redis: 10a. 删除锁
        Redis->>R: 11a. 发布解锁消息
    else 重入次数>0
        Redis->>Redis: 10b. 保留锁
    end
    R->>T: 12. 解锁完成
```

### 3. Redisson 数据结构
```mermaid
graph TB
    subgraph "Redis 存储结构"
        subgraph "可重入锁 - Hash结构"
            HASH[Key: myLock]
            FIELD1[Field: UUID:ThreadID]
            VALUE1[Value: 重入次数]
            TTL[TTL: 30秒]

            HASH --> FIELD1
            FIELD1 --> VALUE1
            HASH -.过期时间.-> TTL
        end

        subgraph "示例数据"
            EXAMPLE["myLock: {<br/>  '8743c9c0-uuid:thread-1': 2,<br/>  TTL: 30s<br/>}"]
        end

        subgraph "WatchDog机制"
            WATCHDOG[看门狗线程]
            RENEW[每10秒续期]
            EXPIRE[重置过期时间为30秒]

            WATCHDOG --> RENEW --> EXPIRE
        end
    end

    HASH -.示例.-> EXAMPLE
    TTL -.自动续期.-> WATCHDOG

    style HASH fill:#4ECDC4
    style EXAMPLE fill:#FFE66D
    style WATCHDOG fill:#FFB84D
```

### 4. RedLock 算法（多节点）
```mermaid
graph TB
    subgraph "RedLock 多节点算法"
        CLIENT[客户端]

        subgraph "独立Redis节点"
            R1[Redis实例1<br/>独立节点]
            R2[Redis实例2<br/>独立节点]
            R3[Redis实例3<br/>独立节点]
            R4[Redis实例4<br/>独立节点]
            R5[Redis实例5<br/>独立节点]
        end

        subgraph "加锁流程"
            STEP1[1. 获取当前时间T1]
            STEP2[2. 依次向5个节点请求锁]
            STEP3[3. 计算总耗时]
            STEP4[4. 判断是否成功]
            SUCCESS[成功: >=3个节点 AND<br/>耗时 < 锁有效期]
            FAIL[失败: 释放所有节点的锁]

            STEP1 --> STEP2 --> STEP3 --> STEP4
            STEP4 --> SUCCESS
            STEP4 --> FAIL
        end
    end

    CLIENT --> STEP1

    STEP2 -.请求.-> R1
    STEP2 -.请求.-> R2
    STEP2 -.请求.-> R3
    STEP2 -.请求.-> R4
    STEP2 -.请求.-> R5

    style SUCCESS fill:#4CAF50
    style FAIL fill:#FF6B6B
    style R1 fill:#4ECDC4
    style R2 fill:#4ECDC4
    style R3 fill:#4ECDC4
    style R4 fill:#4ECDC4
    style R5 fill:#4ECDC4
```

## Zookeeper 分布式锁

### 1. Zookeeper 锁架构
```mermaid
graph TB
    subgraph "应用层"
        APP1[应用1]
        APP2[应用2]
        APP3[应用3]
    end

    subgraph "Zookeeper 集群"
        LEADER[ZK Leader]
        FOLLOWER1[ZK Follower]
        FOLLOWER2[ZK Follower]

        LEADER <-.同步.-> FOLLOWER1
        LEADER <-.同步.-> FOLLOWER2
    end

    subgraph "ZNode 树结构"
        LOCKS[/locks<br/>持久节点]

        subgraph "锁节点 - 临时顺序节点"
            NODE1[/lock-0000000001<br/>应用1持有]
            NODE2[/lock-0000000002<br/>应用2等待]
            NODE3[/lock-0000000003<br/>应用3等待]
        end

        LOCKS --> NODE1
        LOCKS --> NODE2
        LOCKS --> NODE3
    end

    APP1 -->|创建节点| LEADER
    APP2 -->|创建节点| LEADER
    APP3 -->|创建节点| LEADER

    LEADER --> LOCKS

    APP2 -.watch.-> NODE1
    APP3 -.watch.-> NODE2

    style NODE1 fill:#4CAF50
    style NODE2 fill:#FFE66D
    style NODE3 fill:#FFE66D
    style LEADER fill:#FF6B6B
```

### 2. Zookeeper 加锁流程
```mermaid
sequenceDiagram
    participant C1 as 客户端1
    participant C2 as 客户端2
    participant ZK as Zookeeper

    C1->>ZK: 1. 创建临时顺序节点<br/>/locks/lock-0000000001
    ZK->>C1: 2. 节点创建成功

    C1->>ZK: 3. 获取/locks所有子节点
    ZK->>C1: 4. 返回: [lock-0000000001]
    C1->>C1: 5. 判断自己是最小节点
    Note over C1: 获得锁，执行业务

    C2->>ZK: 6. 创建临时顺序节点<br/>/locks/lock-0000000002
    ZK->>C2: 7. 节点创建成功

    C2->>ZK: 8. 获取/locks所有子节点
    ZK->>C2: 9. 返回: [lock-0000000001, lock-0000000002]
    C2->>C2: 10. 判断不是最小节点

    C2->>ZK: 11. watch前一个节点<br/>/locks/lock-0000000001
    Note over C2: 等待锁释放

    C1->>ZK: 12. 删除节点(业务完成)
    ZK->>C2: 13. 触发watch事件
    C2->>ZK: 14. 重新获取子节点
    ZK->>C2: 15. 返回: [lock-0000000002]
    C2->>C2: 16. 判断自己是最小节点
    Note over C2: 获得锁
```

### 3. Zookeeper 锁特性
```mermaid
graph TB
    subgraph "Zookeeper 锁的优缺点"
        subgraph "优点"
            P1[强一致性<br/>CP架构]
            P2[自动释放<br/>临时节点特性]
            P3[公平锁<br/>顺序节点保证]
            P4[可靠性高<br/>集群容错]
        end

        subgraph "缺点"
            C1[性能较低<br/>网络开销大]
            C2[复杂度高<br/>运维成本]
            C3[不适合高并发<br/>吞吐量有限]
        end

        subgraph "适用场景"
            S1[对一致性要求高]
            S2[并发量不大]
            S3[需要公平锁]
            S4[已有ZK环境]
        end
    end

    style P1 fill:#4CAF50
    style P2 fill:#4CAF50
    style P3 fill:#4CAF50
    style P4 fill:#4CAF50
    style C1 fill:#FF6B6B
    style C2 fill:#FF6B6B
    style C3 fill:#FF6B6B
```

## 数据库分布式锁

### 1. 基于数据库的悲观锁
```mermaid
sequenceDiagram
    participant T1 as 线程1
    participant T2 as 线程2
    participant DB as MySQL数据库

    T1->>DB: 1. BEGIN TRANSACTION
    T1->>DB: 2. SELECT ... FOR UPDATE<br/>WHERE resource_id = 1

    DB->>DB: 3. 加排他锁(X锁)
    DB->>T1: 4. 返回记录

    T2->>DB: 5. BEGIN TRANSACTION
    T2->>DB: 6. SELECT ... FOR UPDATE<br/>WHERE resource_id = 1

    Note over T2,DB: 阻塞等待锁释放

    T1->>DB: 7. UPDATE...执行业务
    T1->>DB: 8. COMMIT
    DB->>DB: 9. 释放锁

    DB->>T2: 10. 获得锁，返回记录
    T2->>DB: 11. UPDATE...执行业务
    T2->>DB: 12. COMMIT
```

### 2. 基于数据库的乐观锁
```mermaid
graph TB
    subgraph "乐观锁 - 版本号机制"
        subgraph "表结构"
            TABLE["distributed_lock表<br/>id | resource_name | version"]
            INIT[初始: 1 | stock-001 | 0]
            TABLE --> INIT
        end

        subgraph "更新流程"
            STEP1[1. 查询: SELECT version<br/>WHERE resource = 'stock-001']
            STEP2[2. 业务处理]
            STEP3[3. 更新: UPDATE ... <br/>SET version = version + 1<br/>WHERE version = 旧version]
            STEP4{影响行数?}
            SUCCESS[= 1: 更新成功]
            FAIL[= 0: 版本冲突，重试]

            STEP1 --> STEP2 --> STEP3 --> STEP4
            STEP4 --> SUCCESS
            STEP4 --> FAIL
            FAIL -.重试.-> STEP1
        end
    end

    style SUCCESS fill:#4CAF50
    style FAIL fill:#FF6B6B
    style TABLE fill:#4ECDC4
```

## 分布式锁对比分析

### 1. 三种实现方式对比
```mermaid
graph TB
    subgraph "分布式锁对比"
        subgraph "Redis - Redisson"
            R1[性能: ⭐⭐⭐⭐⭐]
            R2[可靠性: ⭐⭐⭐⭐]
            R3[复杂度: ⭐⭐⭐]
            R4[适合高并发场景]
        end

        subgraph "Zookeeper"
            Z1[性能: ⭐⭐⭐]
            Z2[可靠性: ⭐⭐⭐⭐⭐]
            Z3[复杂度: ⭐⭐⭐⭐]
            Z4[适合强一致性场景]
        end

        subgraph "数据库"
            D1[性能: ⭐⭐]
            D2[可靠性: ⭐⭐⭐]
            D3[复杂度: ⭐]
            D4[适合低并发场景]
        end
    end

    style R1 fill:#4CAF50
    style R4 fill:#4CAF50
    style Z2 fill:#4CAF50
    style Z4 fill:#4CAF50
    style D3 fill:#4CAF50
```

### 2. CAP 理论视角
```mermaid
graph LR
    subgraph "CAP权衡"
        REDIS[Redis分布式锁<br/>AP架构<br/>高可用+性能]
        ZK[Zookeeper锁<br/>CP架构<br/>一致性+分区容错]
        DB[数据库锁<br/>CA架构<br/>一致性+可用性]
    end

    REDIS -.牺牲.-> C[Consistency]
    ZK -.牺牲.-> A[Availability]
    DB -.牺牲.-> P[Partition Tolerance]

    style REDIS fill:#4CAF50
    style ZK fill:#FFB84D
    style DB fill:#4ECDC4
    style C fill:#FF6B6B
    style A fill:#FF6B6B
    style P fill:#FF6B6B
```

## 典型应用场景

### 1. 库存扣减防超卖
```mermaid
sequenceDiagram
    participant U as 用户请求
    participant S as 订单服务
    participant L as Redisson锁
    participant Redis as Redis
    participant DB as 数据库

    U->>S: 1. 下单请求
    S->>L: 2. 尝试获取锁<br/>key: lock:stock:product-123
    L->>Redis: 3. 执行加锁Lua脚本

    alt 获取锁成功
        Redis->>L: 4a. 加锁成功
        L->>S: 5a. 返回锁对象

        S->>DB: 6. 查询库存
        DB->>S: 7. 返回库存=10

        alt 库存充足
            S->>DB: 8a. 扣减库存 stock-1
            S->>S: 9a. 创建订单
        else 库存不足
            S->>S: 8b. 返回库存不足
        end

        S->>L: 10. 释放锁
        L->>Redis: 11. 删除锁
        S->>U: 12. 返回结果
    else 获取锁失败
        Redis->>L: 4b. 加锁失败
        L->>S: 5b. 等待或返回
        S->>U: 6b. 系统繁忙
    end
```

### 2. 定时任务防重复执行
```mermaid
graph TB
    subgraph "分布式定时任务"
        SCHEDULER[定时调度器]

        subgraph "应用集群"
            APP1[应用实例1<br/>定时任务]
            APP2[应用实例2<br/>定时任务]
            APP3[应用实例3<br/>定时任务]
        end

        subgraph "分布式锁"
            LOCK[Redisson分布式锁<br/>key: lock:schedule:task-daily]
        end

        subgraph "执行逻辑"
            TRY[尝试获取锁]
            SUCCESS{获取成功?}
            EXEC[执行任务]
            SKIP[跳过执行]
            RELEASE[释放锁]

            TRY --> SUCCESS
            SUCCESS -->|是| EXEC --> RELEASE
            SUCCESS -->|否| SKIP
        end
    end

    SCHEDULER -.触发.-> APP1
    SCHEDULER -.触发.-> APP2
    SCHEDULER -.触发.-> APP3

    APP1 --> TRY
    APP2 --> TRY
    APP3 --> TRY

    TRY -.竞争.-> LOCK

    style EXEC fill:#4CAF50
    style SKIP fill:#FFE66D
    style LOCK fill:#FF6B6B
```

### 3. 缓存重建防击穿
```mermaid
sequenceDiagram
    participant U1 as 用户1
    participant U2 as 用户2
    participant S as 服务
    participant L as 分布式锁
    participant C as Redis缓存
    participant DB as 数据库

    U1->>S: 1. 请求热点数据
    S->>C: 2. 查询缓存
    C->>S: 3. 缓存过期/不存在

    S->>L: 4. 尝试获取锁<br/>lock:rebuild:key-123
    L->>S: 5. 获取锁成功

    U2->>S: 6. 请求相同数据
    S->>C: 7. 查询缓存
    C->>S: 8. 缓存不存在

    S->>L: 9. 尝试获取锁
    L->>S: 10. 获取锁失败

    S->>S: 11. 等待100ms后重试
    S->>C: 12. 再次查询缓存

    S->>DB: 13. 查询数据库
    DB->>S: 14. 返回数据
    S->>C: 15. 写入缓存
    S->>L: 16. 释放锁

    C->>S: 17. 返回缓存数据(用户2)
    S->>U1: 18. 返回数据
    S->>U2: 19. 返回数据

    Note over S,C: 只有一个请求查询数据库<br/>其他请求等待后从缓存读取
```

## 最佳实践建议

### 1. 选型建议

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 高并发秒杀 | Redis + Redisson | 性能高，支持高并发 |
| 金融交易 | Zookeeper | 强一致性保证 |
| 定时任务 | Redis + Redisson | 简单高效 |
| 配置变更 | Zookeeper | 可靠性高 |
| 低并发场景 | 数据库乐观锁 | 实现简单 |

### 2. Redisson 最佳实践

```java
// 1. 设置合理的锁超时时间
RLock lock = redisson.getLock("myLock");
lock.lock(30, TimeUnit.SECONDS); // 明确超时时间

// 2. 使用 tryLock 避免无限等待
boolean locked = lock.tryLock(5, 30, TimeUnit.SECONDS);

// 3. 确保锁一定会被释放
try {
    // 业务逻辑
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}

// 4. 使用 MultiLock 实现更可靠的锁
RLock lock1 = redisson.getLock("lock1");
RLock lock2 = redisson.getLock("lock2");
RedissonMultiLock multiLock = new RedissonMultiLock(lock1, lock2);
```

### 3. 常见问题与解决方案

```mermaid
graph TB
    subgraph "常见问题"
        subgraph "死锁问题"
            Q1[问题: 进程挂了锁未释放]
            A1[解决: 设置超时时间<br/>使用临时节点ZK]
        end

        subgraph "锁续期问题"
            Q2[问题: 业务执行时间长]
            A2[解决: WatchDog自动续期<br/>适当延长超时时间]
        end

        subgraph "锁误删问题"
            Q3[问题: A的锁被B删除]
            A3[解决: 加锁时设置UUID<br/>解锁时校验]
        end

        subgraph "Redis主从切换"
            Q4[问题: 主从切换导致锁丢失]
            A4[解决: 使用RedLock<br/>多Redis实例]
        end
    end

    Q1 --> A1
    Q2 --> A2
    Q3 --> A3
    Q4 --> A4

    style Q1 fill:#FF6B6B
    style Q2 fill:#FF6B6B
    style Q3 fill:#FF6B6B
    style Q4 fill:#FF6B6B
    style A1 fill:#4CAF50
    style A2 fill:#4CAF50
    style A3 fill:#4CAF50
    style A4 fill:#4CAF50
```

### 4. 性能优化建议

- 减少锁粒度：细化锁的范围
- 使用分段锁：将资源分段，降低竞争
- 读写分离：使用读写锁
- 降级方案：锁获取失败时的兜底策略
- 监控告警：锁等待时间、持有时间

### 5. 安全性建议

- 避免长时间持有锁
- 设置合理的超时时间
- 实现锁的可重入性
- 防止死锁（超时机制）
- 记录锁的持有者和操作日志
