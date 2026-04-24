---
type: explanation
tags: [react, dom, scroll, diagram]
created: 2026-04-24
---

# scrollIntoView 祖先冒泡图

伴随 [[scrollintoview-ancestor-leak]] 的可视化。

## 调用时浏览器做了什么

```mermaid
flowchart TD
    Start([el.scrollIntoView 调用]) --> Check1{最近祖先:<br/>scrollHeight &gt; clientHeight?}
    Check1 -->|否| Check2
    Check1 -->|是| Scroll1[写 scrollTop 让 el 可见]
    Scroll1 --> Check2{还有可滚祖先?}
    Check2 -->|否| End([结束])
    Check2 -->|是| Up[向上找下一个祖先]
    Up --> Check1

    style Scroll1 fill:#fdd,stroke:#c00
    style Check1 fill:#eef
```

**关键**：判定可滚只看 `scrollHeight > clientHeight`，**不看 `overflow`**。`overflow: hidden` 的元素也会进入循环。

## 我遇到的真实错位路径

```mermaid
sequenceDiagram
    participant M as messagesEndRef
    participant MC as messagesContainer<br/>(overflow-y: auto)
    participant CP as ChatPanel<br/>(overflow: hidden)
    participant CA as ContentArea<br/>(overflow: hidden)
    participant Main as &lt;main&gt;<br/>(overflow-y: auto)

    Note over M,Main: messages 溢出到 ChatPanel, <br/>ChatPanel.scrollHeight = 4062

    M->>MC: scrollIntoView()
    MC->>MC: scrollTop = 3660 (滚到底)
    Note over MC: messages 还有剩余不可见<br/>(因为内容超出 ChatPanel 盒子)
    MC->>CP: 继续向上滚
    CP->>CP: scrollTop = 711 ❌<br/>(overflow:hidden 挡不住)
    Note over CP: 整个 chat 被推上 711px
    CP->>CA: 继续向上?
    CA-->>CA: scrollHeight == clientHeight<br/>止步
```

## 三种"保护"的真实作用

```mermaid
flowchart LR
    subgraph 用户手势
        Wheel[wheel/touch] --> OB{overscroll-behavior?}
        OB -->|contain/none| Stop1[✅ 拦住]
        OB -->|auto| Bubble1[冒泡]
    end

    subgraph 编程式滚动
        Prog[scrollIntoView<br/>scrollTop = N<br/>scrollTo] --> OH{overflow?}
        OH -->|hidden| Through1[🚫 穿透]
        OH -->|auto/scroll| Through2[穿透]
        OH -->|clip| Through3[🚫 穿透]

        Through1 --> Check{scrollHeight &gt;<br/>clientHeight?}
        Through2 --> Check
        Through3 --> Check
        Check -->|是| DoScroll[✅ 浏览器真的滚了]
        Check -->|否| NoScroll[不滚]
    end

    style Through1 fill:#fdd
    style Through3 fill:#fdd
    style DoScroll fill:#fdd
    style Stop1 fill:#dfd
```

结论：**`overflow-hidden` 和 `overscroll-contain` 是两个不同维度的限制** —— 一个管可见性、一个管用户手势冒泡，两个都不管编程式 scroll。
