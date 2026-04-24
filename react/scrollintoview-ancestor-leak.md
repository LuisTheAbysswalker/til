---
type: explanation
tags: [react, dom, css, scroll, layout-bug]
created: 2026-04-24
---

# scrollIntoView 会偷偷滚动祖先容器，`overflow-hidden` / `overscroll-contain` 拦不住

## 现象

一个垂直 flex 布局的 chat panel：

```
ChatPanel (flex-col, overflow-hidden)
├── Messages (flex-1 min-h-0 overflow-y-auto)  ← 消息滚动容器
├── Heartbeat (h-3)
└── Input (flex-shrink-0)
```

消息很多时，调用 `messagesEndRef.current.scrollIntoView({ behavior: 'smooth' })` 本意只是让 messages 容器滚到底。**但实际结果是 ChatPanel 整个被"推"上去了几百像素**，header 和 preview 全错位。

DevTools 里看到 `ChatPanel.scrollTop = 711`（ChatPanel 本身被滚了），虽然它 class 上有 `overflow-hidden`。

## 为什么

`Element.scrollIntoView()` 的语义不是"滚动最近的 scroll container"，而是：

> 让目标元素在视口里可见 —— 递归遍历所有**可滚动祖先**，调整它们的 scrollTop/scrollLeft，直到元素可见。

**关键点**：
1. "可滚动"的判定是 `scrollHeight > clientHeight`，**不管 `overflow` 设成什么**。即使 `overflow: hidden`，浏览器仍视它为 scrollable container（用户看不到滚动条，但 JS 可写 `scrollTop`）。
2. `scrollIntoView` 调用的是编程式 scroll，不是用户手势。
3. `overscroll-behavior: contain` 只拦截**用户**滚动链（wheel、touch），对编程式滚动完全无效。

所以只要链上某一级祖先的 `scrollHeight > clientHeight`（比如 messages 容器内容溢出把 ChatPanel 的 scrollHeight 也撑大），浏览器就会把它当目标之一滚动。

## 哪些"保护"是无效的

| 方案 | 拦编程式 scrollIntoView？ |
|---|---|
| `overflow: hidden` | ❌ 只隐藏滚动条，scrollTop 仍可写 |
| `overflow-clip` | ❌ 同上（虽然禁用 scroll，但不拦截 scrollIntoView 的递归） |
| `overscroll-behavior: contain` | ❌ 只管 wheel/touch |
| `overscroll-behavior: none` | ❌ 同上 |
| `pointer-events: none` | ❌ 只拦鼠标事件 |

## 哪些方案有效

### 1. 别用 scrollIntoView，直接写 scrollTop（推荐）

把滚动锁死在目标容器内：

```ts
const scrollMessagesToBottom = () => {
  const el = messagesContainerRef.current;
  if (!el) return;
  el.scrollTo({ top: el.scrollHeight, behavior: 'smooth' });
};
```

优点：100% 不冒泡，`overflow: hidden` 的祖先不会被动。

### 2. 传 `block: 'nearest'`

```ts
el.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
```

`nearest` 只让"最近必要"的 scroll parent 动，不会无脑往上递归。但如果目标元素已经超出最近 scroll parent 的可见区，它**仍会**滚动次近的祖先 —— 所以只能减少冒泡，不能完全杜绝。

### 3. 布局上让祖先 `scrollHeight === clientHeight`

让祖先根本"不可滚"（scrollHeight 等于 clientHeight），浏览器就不会当它候选。手段：
- 子元素总高度严格等于父高度（flex/grid 精确分配）
- 祖先用 `display: grid` + `grid-template-rows: minmax(0, 1fr) ...`，强制 row 不超过 1fr

这是"治标"，能降低触发概率，但一旦子内容打破预期（比如某个组件忘了加 `min-h-0`），scrollIntoView 还是会找到新的可滚祖先。

## 诊断手段

在 DevTools Console 里对嫌疑元素打印：

```js
const el = document.querySelector('SELECTOR');
console.log({
  scrollTop: el.scrollTop,           // 非 0 就是被滚了
  scrollHeight: el.scrollHeight,
  clientHeight: el.clientHeight,
  overflow: getComputedStyle(el).overflow,
});
```

如果看到一个 `overflow: hidden` 的元素 `scrollTop > 0`、且 `scrollHeight > clientHeight`，就是被 scrollIntoView 偷偷滚了。

## 教训

**任何"期望只滚自己容器"的代码，都不要用 `scrollIntoView`**。直接对目标容器写 `scrollTop` / `scrollTo({ top })`，这是唯一能 100% 锁住滚动范围的方法。

## Related
- [[ios-safari-overscroll-textarea-caret]] - 另一个"CSS 拦不住的滚动"场景
