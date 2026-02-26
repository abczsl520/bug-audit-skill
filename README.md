# 🔍 Bug Audit Skill

> Dynamic bug audit for Node.js web projects — profile first, then prescribe.

Built from real-world experience auditing **200+ bugs across 30+ projects** including H5 games, data pipelines, WeChat mini-programs, API services, and bots.

## Why This Skill?

Most audit checklists are static — you run the same 50 items on every project, wasting time on irrelevant checks. This skill takes a different approach:

```
Profile the project → Pick relevant modules → Execute in risk order → Done
```

A game project gets game-specific checks (physics bugs, anti-cheat, WebView compat). A data tool gets data-specific checks (timezone traps, fetch dedup, scheduler reliability). No bloat.

## What's Inside

| File | Size | Content |
|------|------|---------|
| `SKILL.md` | 4.3 KB | Core 5-step flow: Profile → Plan → Execute → Regress → Archive |
| `references/modules.md` | 7.3 KB | 9 audit modules with 100+ check items |
| `references/pitfalls.md` | 4.2 KB | High-frequency pitfall table + remote debugging techniques |

### 9 Audit Modules

| Module | Tag | Focus |
|--------|-----|-------|
| 🔒 Security | All with users | Input validation, auth, CORS, brute force, prototype pollution |
| 📊 Data | All with DB | Atomic ops, timezone traps, float precision, SQLite quirks |
| ⚡ Performance | Large/realtime | Memory leaks, hot path optimization, cache strategies |
| 🎮 Game | Canvas/Phaser | State guards, anti-cheat, rendering bugs, config validation |
| 🔧 WeChat | WeChat WebView | ES6 compat, CDN fallback, JS-SDK, remote debugging |
| 🔌 API | API services | Interface standards, rate limiting, upstream fallback |
| 🤖 Bot | Bot/auto-reply | Timeout, dedup, sensitive word filter |
| 🚀 Deploy | All | PM2, nginx, SSL, SDK overwrite detection |
| 🔄 Regression | All | Self-introduced bugs, cross-file refs after refactor |

## Install

### For [OpenClaw](https://github.com/openclaw/openclaw) Users

```bash
# Clone into your skills directory
git clone https://github.com/abczsl520/bug-audit-skill.git ~/.openclaw/skills/bug-audit
```

Then in any chat, say:

> "对这个项目执行bug排查" or "audit this project for bugs"

The skill triggers automatically.

### Manual Use

Read `SKILL.md` for the flow, `references/modules.md` for check items, `references/pitfalls.md` for common traps.

## Quick Example

**Step 1** — Profile says: Express + SQLite + Phaser + WeChat WebView, 3000 lines, recent modular refactor

**Step 2** — Plan selects: S1 S2 S3 (security) + D1 D2 (data) + G1 G2 G3 G4 (game) + W1 W3 (wechat) + R1 R2 (deploy)

**Step 3** — Execute 8 rounds, find 27 bugs:
- 🔴 5 critical (CORS reflection, quest skip, admin brute force...)
- 🟡 12 medium (timezone mixup, float precision, memory leak...)
- 🟢 10 minor (missing catch, debug logs, cache headers...)

**Step 4** — Regression catches 2 bugs introduced by fixes

## Real-World Pitfalls (Preview)

| Freq | Issue | Projects Hit |
|------|-------|-------------|
| ⭐⭐⭐ | `fetch()` without `.catch()` | 8+ |
| ⭐⭐⭐ | `forEach` + `destroy()` = crash | 3 |
| ⭐⭐⭐ | config.json written but never read | 5 |
| ⭐⭐⭐ | UTC vs Beijing time mixup | 4 |
| ⭐⭐ | CORS `origin:true` reflects any origin | 2 |
| ⭐⭐ | `setInterval` without `clearInterval` | 4 |
| ⭐ | Deploy overwrites SDK init code | 5 |

Full table with 15 entries + WeChat WebView remote debugging techniques in `references/pitfalls.md`.

## Tech Stack Coverage

- **Runtime**: Node.js / Express / Socket.IO
- **Games**: Phaser 3 / Three.js / Canvas / Matter.js
- **Database**: SQLite / MySQL / PostgreSQL
- **Platforms**: WeChat WebView / Standard browsers
- **Infra**: PM2 / Nginx / Let's Encrypt

## Contributing

Found a new pitfall? Open an issue or PR. The more real-world data, the better the audit.

## License

MIT
