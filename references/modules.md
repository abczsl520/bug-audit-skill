# Audit Modules

Read only the sections matching the project's type tags from Phase 1 dissection.

## 🔒 S — Security (projects with user systems)

### S1 Input Validation
- SQL: all `prepare()` parameterized, no string concatenation
- XSS: frontend `esc()` + server-side strip `<>`
- Numbers: `parseInt` + `Math.max(0,...)` + upper clamp
- Array/object type check (prevent array params crashing server)
- Prototype pollution: lookup maps use `Object.create(null)`

### S2 Auth & Permissions
- Cookie: `Secure` + `SameSite=Lax`
- CORS: whitelist array, never `origin:true` (reflects any origin = critical vuln)
- Admin password via header, not GET query (query appears in nginx logs + browser history)
- Brute force protection: N failures → lock IP
- Password comparison: `crypto.timingSafeEqual`
- Admin grant endpoints: reject negative amounts

### S3 Infrastructure
- `express.json({limit:'100kb'})`
- `app.disable('x-powered-by')`
- Global error middleware (no stack traces to client)
- Socket.IO: `maxHttpBufferSize: 16384`
- Max connection limit (prevent DDoS entity creation)

---

## 🔐 C — Cryptographic Failures (all projects) [OWASP 2025 A04]

### C1 Secrets & Credentials
- Hardcoded passwords/API keys/tokens in source code (grep for `password`, `secret`, `apiKey`, `token` in .js files)
- Secrets in config.json committed to git (should be in .env or environment variables)
- Admin password stored as plaintext or weak hash (MD5/SHA1 without salt)
- Password comparison: `===` is vulnerable to timing attacks → use `crypto.timingSafeEqual`
- Session tokens: generated with `Math.random()`? → use `crypto.randomBytes(32).toString('hex')`
- JWT: using `none` algorithm? Secret too short? Stored in localStorage (XSS-accessible)?
- API keys in frontend code (visible in browser DevTools)
- Sensitive data in URL query parameters (logged by nginx, browser history, referrer headers)

---

## 📊 D — Data Consistency (projects with databases)

### D1 Atomic Operations
- Resource deduction: `SET x=x-? WHERE x>=?` (SQLite naturally prevents double-spend)
- Dedup: UNIQUE INDEX + INSERT OR IGNORE
- Float precision: `Math.floor()` safety net (JS float accumulation → 290402.0000000001)
- Cross-SQLite DB: cannot JOIN, query separately then merge
- SQLite strings: single quotes `''` (double quotes `""` = column identifiers, causes crashes)

### D2 Timezone
- `toISOString()` returns UTC! Chinese time: use `getFullYear/getMonth/getDate`
- Daily reset: Beijing midnight (not UTC midnight = Beijing 8am)
- China servers (Asia/Shanghai): no manual +8h needed
- SQLite: `datetime('now','localtime')` vs `datetime('now')`

### D3 Data Tool Specifics (📊 only)
- Feishu sheets: find by name, not index (write-back sheets push Sheet1 to higher index)
- DOU+ stat_cost: unit is fen (÷100 for yuan); is spend within specified date range, not cumulative
- fetch-report only updates orders with data in date range; use `_rfaBefore` timestamp to filter
- After fetch-report, run fetch-all once to catch missed new orders
- Data snapshot/backup mechanism for rollback

---

## ⚡ P — Performance (large or realtime projects)

### P1 Memory Leaks
- Every `setInterval` has matching `clearInterval`
- No destroy/splice during forEach (use `.slice()` snapshot — hit in 3+ projects)
- Game mode end: clean up all AI entities (timed/elimination/royale)
- Socket reconnect: don't stack timers (store ref + clear on disconnect)
- AudioNode: disconnect after stop (onended auto-disconnect)

### P2 Hot Path
- Cache DOM queries (not `getElementById` every frame — 12/frame = 720/sec)
- Config file: mtime cache (not readFileSync per request)
- Leaderboard: 60s memory cache (58 users × 6 SQL = 348 queries/request)
- Large lists: pagination

---

## 🎮 G — Game Logic (game projects only)

### G1 State Guards
- gameOver/endBattle: dedup flag + **reset flag in init** (otherwise 2nd battle never triggers!)
- battleState: correctly set back to `player_turn` in spirit-defeated→switch chain
- visibilitychange pause: needs state lock to prevent duplicate calls
- Physics engine: delta doesn't accumulate across tab switches

### G2 Anti-Cheat
- Rewards computed server-side (never trust client values: quest rewards, shop quantities, seal levels)
- Quest taskKey: whitelist validation (attacker fills progress with fake keys to bypass completion check)
- quest/start: check prerequisite quests (prevents chapter-skipping for high-tier rewards)
- Star-up/merge: validate rarity (R-grade used as SSR material)
- Tower/level skip: `floor > currentMax + 1` check
- Trade/gift/wish: daily limit (unlimited wish-wall = infinite gold exploit)
- Safe-box items: check ALL exit points (trade, gift, merge, sell, fuse)

### G3 Rendering & Interaction
- Phaser canvas as pure background: `pointer-events:none` + disable input (otherwise intercepts game touches)
- After display toggle: `requestAnimationFrame` before reading clientWidth/Height (may be 0 before reflow)
- Overlay: mutual exclusion (open A → close B) + click-outside to close
- Animation overlays: `pointer-events:none` (don't block game controls)
- Resize: rebuild physics walls
- Global touchmove preventDefault: whitelist selectors must match actual HTML
- Phaser camera: don't follow physics body directly (engine-level jitter bug, use lerp + separate position)
- Phaser container + setScrollFactor(0): click offset bug → use standalone elements + setDepth
- Multiple panels sharing DOM ID: clear innerHTML on switch
- `<img src="">`: browser requests current page URL → use inline SVG placeholder

### G4 Config Validation
- config.json values actually read in code (not just written — admin changes have no effect otherwise)
- Module load-time config reads get defaults (fetch not complete yet) → read at runtime
- Frontend hardcoded cost must match backend config (fragment exchange: frontend 50 ≠ backend 150)
- Admin config save: preserve password field (prevent loss if frontend omits it)

---

## 🔧 W — WeChat Compatibility (wechat projects only)

### W1 Syntax
- No ES6+: optional chaining `?.`, nullish coalescing `??`, computed property `{[key]:val}`
- No JS template literals (backticks cause issues with file-write tools)
- `backdrop-filter`: add `-webkit-` prefix
- `safe-area-inset` support (notch screens)

### W2 WeChat APIs
- OAuth callback URL + state parameter (distinguish sources like v2/v3)
- access_token: cache + auto-refresh (2h expiry)
- JS-SDK signature: use current page URL (not fixed URL)
- Image upload: use JPEG (large blank-area PNGs fail WeChat decode)

### W3 Environment
- CDN libraries: download to server (jsdelivr unreliable in China — curl returns HTTP 000/0 bytes)
- After replacing Phaser version: verify Matter.js API compatibility (3.80.1 removed Runner module)
- WeChat cache: add `?v=N` to all JS references (extremely sticky cache)
- Debugging: `navigator.sendBeacon` remote logging > screenshot debug panel (10x more efficient)

---

## 🔌 A — API Service (api-service projects only)

### A1 Interface Standards
- Unified error format: `{ok:false, error:"..."}`
- Parameter type validation
- Proper HTTP status codes (not all 200)
- Large file: streaming response

### A2 Auth & Rate Limiting
- API Key middleware
- Per-key or per-IP rate limiting
- Key revocation mechanism (whitelist)

### A3 External Dependencies
- Upstream API timeout fallback
- Degraded response (cache/defaults) when upstream is down
- Tunnel/proxy health check (e.g., Gemini image generation tunnel)

---

## 🤖 B — Bot (bot projects only)

### B1 Message Handling
- AI reply timeout fallback
- Duplicate message dedup (WeChat may resend)
- Sensitive word filter
- Friendly error replies (no technical details to users)

---

## 🚀 R — Deploy (all projects)

### R1 Basics
- PM2 online, no restart loop
- nginx proxy correct (sub-path prefix strip)
- HTTPS certificate valid
- Static assets: version `?v=N`

### R2 Deploy Safety
- SDK/init code not overwritten by deployment (5 games lost SDK init from deploy overwrite)
- Local vs server file SHA match
- After replacing dependency: verify API compatibility (`grep -c "Runner" phaser.min.js`)

---

## 🧪 E — Error Handling (projects with frontend or network calls)

### E1 Network Error Paths
- Every `fetch()` has `.catch()` with user-visible feedback (not silent failure)
- Login failure: show error message (not blank screen with default values)
- Action submission failure (raid-result, quest-complete): show toast/alert (not "earned $0" confusion)
- Timeout handling: what if server takes 30 seconds? Is there a loading indicator + timeout?
- Offline/disconnect: does the UI degrade gracefully or freeze?

### E2 Server Error Paths
- Every DB query wrapped in try-catch (not unhandled rejection crash)
- Every file operation has error handling (readFileSync on missing file)
- External API call failure: retry? fallback? or crash?
- Malformed request body: does it 400 or crash with TypeError?
- Global error handler: catches unhandled exceptions, logs, returns 500 (not stack trace)

---

## 📱 U — UX Robustness (projects with user interface)

### U1 User-Facing Error States
- Login fails → clear error message (not silent redirect or blank state)
- Empty data → "no results" placeholder (not blank page)
- Loading → spinner or skeleton (not frozen UI)
- Action succeeds → confirmation feedback (not "did it work?")
- Action fails → specific error (not generic "something went wrong")

### U2 Edge Case UX
- First-time user: does the default state make sense? (not "$ 500" when not logged in)
- Rapid double-click: does it trigger twice? (buy button, submit button)
- Back button: does it break state? (game in progress → back → forward)
- Screen rotation / resize: does layout survive?
- Very long text input: does it overflow or break layout? (nickname, chat message)
- Concurrent tabs: does action in tab A break tab B?

---

## 📦 SC — Supply Chain Security (all Node.js projects) [OWASP 2025 A03]

### SC1 Dependency Audit
- Run `npm audit` — any critical/high vulnerabilities?
- `package-lock.json` exists and committed? (reproducible builds)
- Dependencies pinned to exact versions or using `^` (allows minor bumps with potential breaking changes)?
- Any dependencies with 0 maintainers or abandoned (no updates in 2+ years)?
- `postinstall` scripts in dependencies: do any run arbitrary code?
- CDN-loaded libraries: integrity hash (`integrity="sha384-..."`) present? Or downloaded to server?
- Are you using `eval()`, `new Function()`, or `child_process.exec()` with user input?
- Node.js version: is it a supported LTS release? (EOL versions have unpatched CVEs)

---

## 📝 L — Security Logging & Monitoring [OWASP 2025 A09]

### L1 Audit Trail
- Failed login attempts: logged with IP + timestamp + username?
- Successful logins: logged? (detect account takeover)
- High-value transactions (purchase, transfer, delete): logged with user + amount + timestamp?
- Admin actions: logged? (config change, user ban, data export)
- API errors (4xx, 5xx): logged with request details?
- Rate limit triggers: logged?
- Logs stored securely? (not world-readable, not in public/ directory)
- Log rotation: configured? (prevent disk exhaustion from log growth)
- Sensitive data NOT in logs: no passwords, tokens, full credit card numbers in log output
- Logs include enough context to reconstruct an incident: who, what, when, from where, result
