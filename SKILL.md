---
name: bug-audit
description: Comprehensive bug audit for Node.js web projects. Activate when user asks to audit, review, check bugs, find vulnerabilities, or do security/quality review on a project. Works by dissecting the project's actual code to build project-specific check matrices, then exhaustively verifying each item — not by running a generic checklist. Supports games, data tools, WeChat apps, API services, bots, and dashboards.
---

# Bug Audit — Dissect, Then Verify

Do NOT run a generic checklist. Instead: read the code, extract every auditable entity, then exhaustively question each one.

## Phase 1: Dissect (10-15 min)

Read all project files. Build 6 tables. These tables ARE the audit — everything found here gets verified in Phase 2.

### Table 1: API Endpoints

For every route in server-side code:

```
| # | Method | Path | Auth? | Params validated? | Precondition | Returns | Attack vector |
```

For each endpoint, ask:
- Can I call this without authentication?
- Can I pass 0, negative, NaN, huge numbers, arrays, objects?
- Can I skip a prerequisite API and call this directly?
- What happens if I call this 100 times per second?
- Does the response leak sensitive data (openid, internal IDs, full user objects)?

### Table 2: State Machines

For every boolean/enum state variable (isGameOver, battleState, Game.running, phase, mode...):

```
| # | Variable | Set by | Read by | Init value | Reset when? | Can it leak across lifecycles? |
```

For each variable, ask:
- If the game/session ends, does this get reset?
- If I start a new round immediately, will stale state from the previous round affect it?
- Are there race conditions between setters?

### Table 3: Timers

For every setTimeout/setInterval:

```
| # | Type | Delay | Created in | Cleared in | What if lifecycle ends before it fires? |
```

For each timer, ask:
- Is the handle stored for cleanup?
- If the game ends / user disconnects / page navigates, does this still fire?
- If it fires after cleanup, does it reference destroyed objects?

### Table 4: Numeric Values

For every user-influenceable number (cost, score, damage, lootValue, kills, quantity...):

```
| # | Name | Source (client/server/config) | Validated? | Min | Max | What if 0? | What if negative? |
```

For each value, ask:
- Is the server-side cap realistic? (kills cap 200 but max enemies is 50?)
- Can the client send a value the server trusts without verification?
- Float precision issues? (accumulated math → 290402.0000000001)

### Table 5: Data Flows (Critical!)

For every pair of related APIs (buy→use, start→complete, pay→deliver, login→action):

```
| # | Step 1 API | Step 2 API | Token/link between them? | Can skip Step 1? | Can replay Step 1? |
```

This is where the biggest bugs hide. For each flow, ask:
- Can I call Step 2 without ever calling Step 1? (raid-result without buy)
- Can I call Step 1 once but Step 2 many times? (buy once, submit results 10 times)
- Is there a one-time token linking them? If not, this is a critical vulnerability.
- Can I call Step 1 with cost=0 then Step 2 with high reward?

### Table 6: Resource Ledger

For every in-game resource (coins, gems, items, XP, energy...):

```
| # | Resource | All INFLOWS (APIs/events that add) | All OUTFLOWS (APIs/events that subtract) | Daily limits? | Can any inflow be infinite? |
```

For each resource, ask:
- Is there any inflow without a corresponding cost? (free coins from quest with no cooldown)
- Can any outflow go negative? (sell item → coins, but what if coins overflow?)
- Are items in safe-box excluded from ALL outflows? (trade, sell, merge, fuse, gift)
- Is there a loop? (buy item A → sell for more than cost → repeat)

## Phase 2: Verify (main audit)

Go through every row in every table. For each row, determine:
- 🔴 Critical: exploitable security hole, data loss, crash
- 🟡 Medium: logic error, inconsistency, performance issue  
- 🟢 Minor: code quality, edge case, UX issue
- ✅ OK: verified correct

Output format:
```
Bug N: [🔴/🟡/🟢] Brief description
- Row: Table X, #Y
- Cause: ...
- Fix: ...
- File: ...
```

**Do NOT stop when you run out of "inspiration".** You stop when every row in every table has been verified ✅ or flagged 🔴/🟡/🟢. This is exhaustive, not heuristic.

## Phase 3: Supplement

After exhausting all tables, run these generic checks as a safety net. Read `references/modules.md` and pick sections matching the project:

- 🔒 Security (S1-S3): CORS, XSS, SQLi, brute force — if project has users
- 📊 Data (D1-D2): Timezone, atomic ops, float precision — if project has DB
- ⚡ Performance (P1-P2): Memory leaks, hot paths — if project is large/realtime
- 🔧 WeChat (W1-W3): ES6 compat, CDN, debugging — if runs in WeChat WebView
- 🔌 API (A1-A3): Interface standards, rate limiting — if project is an API service
- 🤖 Bot (B1): Timeout, dedup, sensitive words — if project is a bot
- 🚀 Deploy (R1-R2): PM2, nginx, SSL, SDK overwrite — all projects

These catch "known pattern" bugs that the tables might miss (like CORS misconfiguration or timezone traps).

## Phase 4: Regression + Verify

- Check that fixes didn't introduce new bugs
- After modular split: verify cross-file variable/function reachability
- Live smoke test: homepage 200, key APIs return JSON, login works, core feature functional

## Phase 5: Archive

Update project docs with: date, tables built, bugs found/fixed, key pitfalls for next audit.

## Key Principles

1. **Tables first, checking second.** Building the tables IS the hard work. Once you have them, verification is mechanical.
2. **Exhaustive, not heuristic.** Don't stop when you "feel done." Stop when every row is verified.
3. **Think like an attacker.** For every API: "How would I exploit this?" For every value: "What if I send garbage?"
4. **Data flows are where critical bugs hide.** The link (or lack thereof) between related APIs is the #1 source of exploitable vulnerabilities.
5. **Generic checklists are supplements, not the main event.** They catch known patterns; the tables catch project-specific logic bugs.

## Reference Files

- `references/modules.md` — Generic audit modules (Security, Data, Performance, Game, WeChat, API, Bot, Deploy) for Phase 3 supplementary checks.
- `references/pitfalls.md` — Real-world pitfall lookup table from 200+ bugs, plus WeChat WebView remote debugging techniques.
