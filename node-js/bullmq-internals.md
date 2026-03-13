---
type: explanation
tags: [node-js, bullmq, redis, queue]
created: 2026-03-13
---

# BullMQ 底层实现原理

## 核心数据结构（全部基于 Redis）

| Redis 结构 | 用途 | Key 示例 |
|-----------|------|----------|
| **Hash** | 存储 job 数据（payload、状态、时间戳） | `bull:myqueue:jobId` |
| **Sorted Set (ZSET)** | 延迟队列、优先级队列、限流（按时间戳/优先级排序） | `bull:myqueue:delayed`、`bull:myqueue:prioritized` |
| **List** | 等待队列和处理中队列（FIFO） | `bull:myqueue:wait`、`bull:myqueue:active` |
| **Set** | 已完成/已失败的 job 集合 | `bull:myqueue:completed` |
| **Stream** | 事件通知（job 状态变化） | `bull:myqueue:events` |

## 队列操作原理

### 入队（addJob）
- 用 Lua 脚本原子操作：写 job 数据到 Hash + RPUSH 到 wait 列表（或 ZADD 到 delayed/prioritized ZSET）
- Lua 脚本保证原子性，不会出现半写状态

### 消费（processJob）
- Worker 用 `BRPOPLPUSH`（阻塞式）从 `wait` 列表弹出，同时推入 `active` 列表
- 原子操作——job 不会丢也不会被两个 worker 抢到

### 延迟任务
- ZADD 到 `delayed` ZSET，score 是执行时间戳
- 定时器轮询 ZRANGEBYSCORE 找到期的 job，移到 `wait` 列表

### 重试
- 失败后把 job 从 `active` 移回 `wait`（或 `delayed`，如果有 backoff）
- 重试次数记在 job 的 Hash 里

### 限流（Rate Limiting）
- 用 ZSET 记录时间窗口内的处理数量
- 超限时 job 暂存在 `limiter` ZSET，等窗口过了再放回

## 为什么全用 Lua 脚本

BullMQ 几乎所有操作都封装成 Redis Lua 脚本（`commands/` 目录下有几十个 `.lua` 文件）：
- **原子性**：多步操作不会被其他命令插入
- **性能**：一次网络往返完成整个操作，减少 RTT
- **一致性**：不需要分布式锁

## 一句话总结

BullMQ = Redis List（FIFO队列）+ Sorted Set（延迟/优先级）+ Hash（job数据）+ Lua脚本（原子操作），本质上是在 Redis 上实现了一个可靠的状态机。
