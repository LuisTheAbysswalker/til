# TIL (Today I Learned)

A collection of concise write-ups on things I learn day to day.

Each note is classified by [Diátaxis](https://diataxis.fr/) type:
- **explanation** — why / how something works internally
- **how-to** — steps to solve a specific problem
- **reference** — facts, parameters, API lookup
- **tutorial** — learning by doing, step-by-step

## Categories

- [TypeScript](typescript/) - Type system, generics, compiler
- [Node.js](node-js/) - Runtime, modules, performance
- [Fastify](fastify/) - Routes, plugins, lifecycle
- [React](react/) - Components, hooks, state management
- [Claude SDK](claude-sdk/) - Agent SDK, MCP, tool calls
- [E2B](e2b/) - Sandbox, filesystem, runtime
- [Multiplayer](multiplayer/) - MQTT, real-time communication, sync
- [Architecture](architecture/) - Design patterns, system design
- [DevOps](devops/) - K8s, Docker, CI/CD
- [Redis](redis/) - Data structures, commands, patterns
- [Codex](codex/) - Codex agent, sandboxing, local agent setup
- [Mobile Web](mobile-web/) - iOS/Android web quirks, viewport, keyboard

## Conventions

Notes are written to render correctly in **both** GitHub and Obsidian.

- **Diagrams** are Mermaid, inlined in the note itself. No companion `-flow.md` files — a diagram belongs to the note it explains.
- **Callouts** use only the five types GitHub and Obsidian share: `[!NOTE]`, `[!TIP]`, `[!IMPORTANT]`, `[!WARNING]`, `[!CAUTION]` — uppercase, no custom title.
- **Links** between notes are relative Markdown links, not `[[wikilinks]]`, because GitHub renders wikilinks as literal text.
- **Frontmatter**: `type`, `tags`, `created` are required; `source` records the URL when a note came from an article.

Opening the vault in Obsidian gives you `TIL.base`, a live index grouped by type and by category. It queries the frontmatter, so it never needs maintaining. GitHub does not render `.base` files — this README is the entry point here.
