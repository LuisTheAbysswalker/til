---
type: explanation
tags: [architecture, race-condition, distributed-lock, redis, state-machine]
created: 2026-03-20
---

# 并发 API 调用导致提前发布的 Bug 分析

## 背景

`POST /sessions/:id/game` 接口负责创建和发布游戏。它有一个 `completed` 参数：

| 调用方 | completed | 意图 |
|--------|-----------|------|
| 客户端 confirm | `false` | 只创建 GENERATING 状态的 game 记录 |
| Sandbox game_ready | `true` | 游戏构建完成，发布并发送通知 |

## 问题

Native 客户端和 Web (Safari) 端同时调用 confirm，两个 `completed=false` 的请求几乎同时到达：

```
T+0ms   请求A: completed=false, 没有 existing game → 创建 GENERATING game ✓
T+50ms  请求B: completed=false, 有 existing game → 直接 publishGame() ✗
```

请求 B 发现已经有 game 了（请求 A 刚创建的），走到了 `else if (game)` 分支，**直接 publish**。此时 sandbox 还没开始 build，但 `game_published` MQTT 通知已经发出，Native 收到后跳转到游戏页面 — 用户看到一个空游戏。

## 问题时间线（从客户端日志还原）

```
01:59:08.090  前端 confirm → Promise.all([createSandbox, processImages, createGame])
01:59:08.939  后端 createGame → existing game found → publishGame() → MQTT game_published
01:59:08.953  Native 收到通知 → session status 6→9 → 跳转 game room
01:59:12.844  sandbox 初始化完成（但已经没人连了）
```

从 confirm 到误跳转只用了 **850ms**。

## 修复方案

### 1. 业务逻辑守卫

`completed=false` + existing game 时，检查 session 是否真的有游戏代码文件（`.html`/`.js`），而不是无条件 publish：

```typescript
} else if (game) {
  const hasGameCode = session.files?.some(f =>
    f.path.endsWith('.html') || f.path.endsWith('.js')
  );
  if (hasGameCode) {
    game = await publishGame(game);
  }
}
```

只有 sandbox 真正 build 过（产生了代码文件）才允许 publish。

### 2. Redis 分布式锁

用 `withRedisLock` 防止并发执行，第二个请求直接返回已有 game：

```typescript
const result = await withRedisLock(`create-game:${sessionId}`, async () => {
  // 核心逻辑
}, {
  ttlMs: 10000,
  onContention: async () => {
    // 返回已有的 game record
    const existing = await gameStore.getGame(session.output.gameId);
    return reply.send({ code: 0, data: { game: existing } });
  },
});
```

### 3. 双重保护

两层防护缺一不可：
- **锁**：防止并发执行（但 Redis 不可用时会降级为无锁）
- **文件检查**：即使没锁，也不会发布空游戏（业务逻辑兜底）

## 教训

1. **多端并发**：同一个操作可能从多个客户端同时触发（Native + Web），API 设计时必须考虑幂等性
2. **状态驱动 vs 参数驱动**：不能只看"有没有 existing game"就决定行为，要看 game 的实际状态和内容
3. **通知的副作用**：`publishGame` 里包含发送 MQTT 通知的副作用，一旦误触发就会导致客户端状态跳转，不可逆
4. **调试方法**：客户端日志 + 服务端日志 + sandbox 日志三方对照还原时间线，精确到毫秒级

## Related
- [[bullmq-internals]] - 另一个涉及并发和原子操作的场景
