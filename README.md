# 🔍 Bug Audit Skill

> Don't run a checklist. Dissect the project, then exhaustively verify every entity.

Built from a hard lesson: a project took **21 rounds** to find 172 bugs using generic checklists. Post-mortem revealed that building project-specific check matrices first would have caught most bugs in **3-4 rounds**.

## The Problem with Checklists

Generic checklists catch "known pattern" bugs (CORS, XSS, timezone). But most critical bugs are **project-specific logic vulnerabilities**:
- `buy` API accepts `cost=0` → free purchases (not in any checklist)
- `raid-result` callable without calling `buy` first → infinite money exploit
- Search completion doesn't verify distance → remote looting

These bugs live in the **relationships between APIs**, not in individual code patterns.

## The Solution: Dissect → Verify → Supplement

```
Phase 1: Dissect — Read code, build 6 project-specific tables (10-15 min)
Phase 2: Verify  — Exhaustively check every row in every table
Phase 3: Supplement — Run generic modules as safety net
Phase 4: Regress — Check fixes didn't introduce new bugs
Phase 5: Archive — Record pitfalls for next audit
```

### The 6 Tables

| Table | Extracts | Key Question |
|-------|----------|-------------|
| API Endpoints | Every route: method, path, auth, params | Can I bypass? What if I send garbage? |
| State Machines | Every state variable: setter, reader, lifecycle | Does it leak across lifecycles? |
| Timers | Every setTimeout/setInterval | Does it fire after cleanup? |
| Numeric Values | Every user-influenceable number | What if 0? Negative? Huge? |
| Data Flows | Every related API pair (buy→use) | Can I skip Step 1 and call Step 2 directly? |
| Resource Ledger | Every resource: all inflows, all outflows | Is there an infinite loop? |

**Data Flows table is the most critical.** The biggest bugs (buy bypass, missing raid tokens) hide in the links between APIs.

## Install

```bash
git clone https://github.com/abczsl520/bug-audit-skill.git ~/.openclaw/skills/bug-audit
```

Then say: "对这个项目执行bug排查" or "audit this project for bugs"

## What's Inside

| File | Content |
|------|---------|
| `SKILL.md` | Core methodology: 6 tables + 5 phases |
| `references/modules.md` | 9 generic audit modules for Phase 3 |
| `references/pitfalls.md` | 200+ real-world pitfalls + debugging techniques |

## Documentation

Full docs on the [Wiki](https://github.com/abczsl520/bug-audit-skill/wiki):
- [解剖流程详解](https://github.com/abczsl520/bug-audit-skill/wiki/解剖流程详解) — How to build each table, what to ask
- [排查模块一览](https://github.com/abczsl520/bug-audit-skill/wiki/排查模块一览) — 9 generic modules for supplementary checks
- [实战踩坑速查](https://github.com/abczsl520/bug-audit-skill/wiki/实战踩坑速查) — High-frequency pitfalls + remote debugging

## License

MIT
