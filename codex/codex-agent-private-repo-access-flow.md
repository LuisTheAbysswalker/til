---
type: explanation
tags: [codex, local-codex-agent, sandbox, seatbelt, diagram]
created: 2026-07-02
---

# Codex agent 私有 repo 访问 — 决策流程

配套 [[codex-agent-private-repo-access]] 的可视化。核心:**两层**(OS 沙箱层 + Prompt 自律层)必须同时对齐。

```mermaid
flowchart TD
    Start([让沙箱 Codex agent 访问私有 repo]) --> Goal{目的?}

    Goal -->|编辑代码| Edit[用已有本地 checkout<br/>如 ~/work/&lt;repo&gt;]
    Goal -->|只读 PR diff| Diff[gh pr diff]

    %% ---- 编辑路径 ----
    Edit --> L2["② Prompt 层 必须放开<br/>AGENT_LOCAL_FILESYSTEM_ACCESS=full<br/>否则 agent 拒绝 workspace 外路径"]
    L2 --> Mode{"① AGENT_PROCESS_SANDBOX ?"}

    Mode -->|none 默认| WR["codex --sandbox workspace-write<br/>config.toml [sandbox_workspace_write]:<br/>writable_roots = repo 路径<br/>network_access = true"]
    Mode -->|macos-seatbelt| AP["bridge seatbelt 接管<br/>ALLOW_PATHS = repo 路径 逗号分隔<br/>带上默认 ~/.cache/codex-runtimes<br/>网络已开, config.toml 被忽略"]

    %% ---- 读 diff 路径 ----
    Diff --> Tok[".env: GH_TOKEN = 细粒度 PAT<br/>GIT_TERMINAL_PROMPT=0"]
    Tok --> Net{"沙箱有网络?"}
    Net -->|workspace-write| NetW[config.toml network_access=true]
    Net -->|macos-seatbelt| NetS[已开 allow default]

    WR --> Restart([重启 bridge = dotenv 重新加载])
    AP --> Restart
    NetW --> Restart
    NetS --> Restart

    Restart --> Done([在 Slack 指定 repo 路径 / PR 号])

    %% ---- 常见错误 ----
    Trap1["⚠ 保持 ② =workspace<br/>→ agent 自觉拒绝改 repo"]:::warn
    Trap2["⚠ 只加 ALLOW_PATHS 但 ① =none<br/>→ 被完全忽略"]:::warn
    L2 -.纠正.-> Trap1
    AP -.纠正.-> Trap2

    classDef warn fill:#3a1e1e,stroke:#c0392b,color:#f5d5d5;
```

## 一句话记忆

- **② `AGENT_LOCAL_FILESYSTEM_ACCESS`** = agent 被*告知*能否碰本地文件 → 要改 repo 必须 `full`。
- **① `AGENT_PROCESS_SANDBOX`** = OS 真正的边界 → `ALLOW_PATHS` 只在 `macos-seatbelt` 有效;`none` 默认时用 config.toml 的 `writable_roots`。
- `~/.ssh` / Keychain 在沙箱内一律 deny → gh 走 `GH_TOKEN` env,git 操作放沙箱外。
