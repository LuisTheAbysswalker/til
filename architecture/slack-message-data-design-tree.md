---
type: explanation
tags: [architecture, im, slack, diagram]
created: 2026-04-29
---

# Slack 消息数据结构树（可视化）

配套：[[slack-message-data-design]]

## 顶层结构

```mermaid
graph TD
    Message["📨 Message<br/>(ts, user, channel, type, subtype)"]

    Message --> Content["📝 内容（不可变）"]
    Message --> Meta["🏷️ 元数据（可变）"]
    Message --> Ctx["🌐 上下文"]

    Content --> Text["text<br/>plain-text fallback"]
    Content --> Blocks["blocks[]<br/>Block Kit 富结构"]
    Content --> Attach["attachments[]<br/>legacy 富卡片"]
    Content --> Files["files[]<br/>附件"]

    Meta --> Reactions["reactions[]<br/>{name, users, count}"]
    Meta --> Edited["edited<br/>{user, ts}"]
    Meta --> Thread["threading<br/>thread_ts / reply_count"]

    Ctx --> Team["team / channel"]
    Ctx --> Bot["app_id / bot_id"]

    classDef content fill:#e8f4ff,stroke:#1264A3
    classDef meta fill:#fff4e8,stroke:#C77400
    classDef ctx fill:#f0f0f0,stroke:#666

    class Content,Text,Blocks,Attach,Files content
    class Meta,Reactions,Edited,Thread meta
    class Ctx,Team,Bot ctx
```

## blocks 三层结构

```mermaid
graph TD
    Blocks["blocks[]"] --> RT["rich_text"]
    Blocks --> Sec["section"]
    Blocks --> Div["divider"]
    Blocks --> Img["image"]
    Blocks --> CtxB["context"]
    Blocks --> Act["actions"]
    Blocks --> Hdr["header"]
    Blocks --> Inp["input"]
    Blocks --> Vid["video"]
    Blocks --> Fil["file"]

    RT --> RTSec["elements[]"]
    RTSec --> RTS1["rich_text_section<br/>普通段落"]
    RTSec --> RTS2["rich_text_list<br/>有序/无序列表"]
    RTSec --> RTS3["rich_text_quote<br/>&gt; 引用"]
    RTSec --> RTS4["rich_text_preformatted<br/>代码块"]

    RTS1 --> Elements["elements[]"]
    Elements --> ETxt["text<br/>{text, style}"]
    Elements --> EUsr["user<br/>{user_id}"]
    Elements --> EChn["channel<br/>{channel_id}"]
    Elements --> EUgp["usergroup<br/>{usergroup_id}"]
    Elements --> EBcast["broadcast<br/>{range}"]
    Elements --> EEmoji["emoji<br/>{name, unicode}"]
    Elements --> ELink["link<br/>{url, text}"]
    Elements --> EColor["color<br/>{value}"]

    classDef block fill:#e8f4ff,stroke:#1264A3
    classDef section fill:#fff4e8,stroke:#C77400
    classDef element fill:#e8ffe8,stroke:#2E7D32

    class RT,Sec,Div,Img,CtxB,Act,Hdr,Inp,Vid,Fil block
    class RTS1,RTS2,RTS3,RTS4 section
    class ETxt,EUsr,EChn,EUgp,EBcast,EEmoji,ELink,EColor element
```

## 一条富文本消息的完整解析

> `hey @alice check 🔥 https://x.com`

```mermaid
graph TD
    M["Message"] --> T["text:<br/>'hey &lt;@U02ALICE&gt; check 🔥 https://x.com'"]
    M --> B["blocks[0]"]

    B --> RT["type: rich_text"]
    RT --> S["rich_text_section"]
    S --> E1["text: 'hey '"]
    S --> E2["user: U02ALICE"]
    S --> E3["text: ' check '"]
    S --> E4["emoji: 🔥"]
    S --> E5["text: ' '"]
    S --> E6["link: https://x.com"]

    classDef msg fill:#f9f9f9,stroke:#333
    classDef txt fill:#fff9e8,stroke:#C77400
    classDef block fill:#e8f4ff,stroke:#1264A3
    classDef element fill:#e8ffe8,stroke:#2E7D32

    class M msg
    class T txt
    class B,RT,S block
    class E1,E2,E3,E4,E5,E6 element
```

## threading 的平铺存储

```mermaid
graph LR
    P["父消息<br/>ts: 1714060000<br/>thread_ts: 1714060000<br/>reply_count: 3"]
    R1["回复 1<br/>ts: 1714061000<br/>thread_ts: 1714060000"]
    R2["回复 2<br/>ts: 1714062000<br/>thread_ts: 1714060000"]
    R3["回复 3<br/>ts: 1714063000<br/>thread_ts: 1714060000"]

    P -.thread_ts.-> R1
    P -.thread_ts.-> R2
    P -.thread_ts.-> R3

    Sib["普通消息<br/>ts: 1714065000<br/>(无 thread_ts)"]

    classDef parent fill:#fff4e8,stroke:#C77400
    classDef reply fill:#e8f4ff,stroke:#1264A3
    classDef sibling fill:#f0f0f0,stroke:#666

    class P parent
    class R1,R2,R3 reply
    class Sib sibling
```

> 都在同一个 messages collection，按 `thread_ts` 分组就是 thread。

## 内容 vs 元数据：何时变

```mermaid
stateDiagram-v2
    [*] --> Created: 用户发送
    Created --> Edited: 用户编辑
    Edited --> Edited: 再次编辑（覆盖）
    Edited --> [*]: 删除

    note right of Created
        text / blocks / attachments / files
        ← 写一次，编辑覆盖；不存历史版本
    end note

    state "reactions / replies / pins" as Meta
    Created --> Meta: 增量更新（独立于内容）
    Meta --> Meta: 不断变化
```

## Related

- [[slack-message-data-design]] - 详细文字版
