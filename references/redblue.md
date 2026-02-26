# Red Team / Blue Team Playbook

> Phase 3 of the audit. After dissecting (Phase 1) and verifying tables (Phase 2), switch to adversarial mode.
> Don't test endpoints in isolation — build multi-step attack chains combining multiple weaknesses.

## How to Use

1. Identify the project type(s) from Phase 1 dissection
2. Run ALL chains in "Universal Chains" (apply to every project)
3. Run the type-specific chains matching your project
4. For each finding, verify the 4 Blue Team defense layers
5. A project can match multiple types (e.g., a game with WeChat login = 🎮 + 🔧)

---

# PART 1: UNIVERSAL CHAINS (All Projects)

These 4 chains apply to EVERY project regardless of type.

## U-Chain 1: Auth Bypass Escalation
```
Goal: Access unauthorized data or admin functions
1. List all endpoints that DON'T require auth → what data do they leak?
2. Call auth-required endpoints without token → 401 or silent fail?
3. User A's token → access User B's data (IDOR)
4. Replay expired/revoked token → server checks expiry?
5. Admin endpoints: password in GET query? Hardcoded? Brute-forceable?
6. Login response: leaks openid/internal IDs enabling impersonation?
7. Cookie: Secure + SameSite=Lax? HttpOnly?
8. Session: fixed expiry or sliding? Logout actually invalidates?
```

## U-Chain 2: Injection Escalation
```
Goal: Execute code or corrupt data
1. User-controlled strings (nickname, message, filename, search query):
   - Set to <script>alert(1)</script> → check every render point
   - Set to ' OR 1=1-- → check all SQL (even prepare() — dynamic table/column names?)
   - Set to {{7*7}} → template injection?
2. JSON body attacks:
   - {__proto__: {isAdmin: true}} → prototype pollution
   - 10MB body → payload size limit? (express.json limit)
   - Array where string expected: {name: [1,2,3]} → type validation?
   - Deeply nested: {a:{b:{c:{d:...}}}} → recursion depth?
3. File upload (if any):
   - Upload .js/.html → served as executable?
   - Path traversal: filename=../../etc/passwd
   - Oversize file → disk exhaustion?
4. URL/redirect parameters:
   - Open redirect: ?redirect=https://evil.com
   - SSRF: ?url=http://169.254.169.254 (cloud metadata)
```

## U-Chain 3: Rate & Resource Abuse
```
Goal: Denial of service or resource exhaustion
1. Expensive endpoints (search, export, leaderboard, report generation):
   - 100 requests/sec → rate limited?
   - Large result set → pagination enforced?
2. Connection exhaustion:
   - Socket.IO: 1000 connections → max limit?
   - HTTP: keep-alive flood → connection pool limit?
3. Entity creation spam:
   - Create 1000 records/rooms/sessions rapidly → cleanup?
   - Each creation allocates memory → bounded?
4. Storage abuse:
   - Upload repeatedly → disk limit?
   - Log spam → log rotation?
5. CPU abuse:
   - Regex with evil input (ReDoS)
   - Complex query/filter → timeout?
```

## U-Chain 4: Data Leakage & Privacy
```
Goal: Extract sensitive information
1. API responses: do they return full user objects? (password hash, openid, email, phone)
2. Error messages: stack traces? SQL errors? File paths?
3. Debug/config endpoints left open? (/debug, /config, /env, /status with secrets)
4. Logs: sensitive data in nginx access logs? (tokens in GET query)
5. Source maps: .map files accessible in production?
6. Git: /.git/ accessible?
7. CORS: origin:true reflects any origin → credential theft
8. Response headers: x-powered-by leaks framework version?
```

---

# PART 2: TYPE-SPECIFIC CHAINS

## 🎮 Game Projects

### G-Chain 1: Skip-Pay-Collect (Economic Exploit)
```
Goal: Get rewards without paying
1. Find "pay" API (buy, start-raid, purchase, bet, enter-dungeon)
2. Find "collect" API (raid-result, quest-complete, claim-reward, settle, cash-out)
3. Call collect WITHOUT calling pay → works?
4. Call pay with cost=0 or cost=-1, then collect with max reward
5. Call pay once, collect multiple times (replay)
6. Check: one-time token linking pay→collect? If not → 🔴 Critical
7. Check: server recomputes reward or trusts client value?
```

### G-Chain 2: Economic Loop
```
Goal: Generate infinite currency/resources
1. Map ALL resource inflows and outflows (from Table 6)
2. Find cycles: buy A for X → sell/convert A for Y where Y > X
3. Quest/achievement rewards: claimable repeatedly? Cooldown enforced server-side?
4. Daily reset: actually resets? Timezone bug = double-claim window
5. Trade between accounts: transfer to alt → rollback main?
6. "Free" inflows: no cooldown, no daily limit, no cost?
7. Negative outflow: sell item for negative price → gain items?
8. Overflow: resource count exceeds MAX_SAFE_INTEGER?
```

### G-Chain 3: State Manipulation
```
Goal: Corrupt game state for advantage
1. Start action A → force-quit → start action B → A's state leaks into B?
2. Game-over: can I still trigger reward events after game ends?
3. Two browser tabs: same action simultaneously → race condition on balance?
4. Rapid duplicate requests: double-click buy → charged once, received twice?
5. visibilitychange during critical transition → state corruption?
6. Timer from round 1 fires during round 2 → stale callback?
7. Reconnect during battle → duplicate entity creation?
```

### G-Chain 4: Anti-Cheat Bypass
```
Goal: Achieve impossible game results
1. Score/kills/damage: client-reported? Server validates against max possible?
2. Quest taskKey: whitelist? Can I submit fake taskKeys?
3. Level/floor skip: can I jump to floor 100 from floor 1?
4. Rarity bypass: use R-grade material as SSR in merge/upgrade?
5. Speed hack: game timer client-side? Server checks elapsed time?
6. Safe-box items: ALL exit points check safe-box flag? (trade, sell, merge, fuse, gift)
7. Cooldown bypass: client-side timer only? Server enforces?
```

---

## 📊 Data Tool / Dashboard Projects

### D-Chain 1: Data Access Control
```
Goal: Access or modify data beyond authorization
1. Multi-tenant: User A can see User B's data? (check all query WHERE clauses)
2. Export/download: can I export other users' data by changing ID in URL?
3. Filter bypass: remove filter params → get ALL data instead of scoped?
4. Aggregation leak: summary stats reveal individual records?
5. Admin panel: accessible without admin role? Role check on every endpoint?
6. Bulk operations: delete/update without ownership check?
```

### D-Chain 2: Data Integrity Attack
```
Goal: Corrupt or manipulate stored data
1. Import/upload: malformed CSV/JSON → crashes parser? Partial import leaves dirty state?
2. Concurrent writes: two users edit same record → last-write-wins data loss?
3. Cascade delete: deleting parent orphans children?
4. Numeric precision: float accumulation → 290402.0000000001?
5. Timezone: daily aggregation uses UTC midnight (= Beijing 8am) → wrong day?
6. SQL: string quotes (SQLite "" = column name, not string → silent wrong query)
7. Cross-DB query: two SQLite DBs can't JOIN → code tries anyway → crash?
```

### D-Chain 3: Scheduled Task Abuse
```
Goal: Disrupt automated data pipelines
1. Cron/scheduler: can external request trigger a scheduled task? (no auth on trigger endpoint)
2. Long-running task: no timeout → blocks worker forever?
3. Failed task: retry logic? Infinite retry loop? Duplicate data on retry?
4. Task overlap: previous run still going when next starts → concurrent corruption?
5. External API dependency: upstream down → task crashes? Partial data saved?
6. Data snapshot: no backup before destructive operation → no rollback?
```

---

## 🔌 API Service Projects

### A-Chain 1: API Key / Token Abuse
```
Goal: Use API without authorization or exhaust quotas
1. API key in URL query → logged in nginx/CDN/browser history
2. Key rotation: old keys still work after rotation?
3. Key scope: key for read-only can write?
4. Key sharing: no per-key rate limit → one leaked key = unlimited abuse
5. Free tier bypass: exceed quota → still served? Or 429?
6. Key enumeration: sequential keys? Can I guess valid keys?
```

### A-Chain 2: Upstream Dependency Attack
```
Goal: Exploit trust in upstream/downstream services
1. Upstream timeout: no timeout set → request hangs forever?
2. Upstream returns unexpected format → crash? Or graceful degradation?
3. Upstream returns malicious content → passed through to client unsanitized?
4. Webhook receiver: no signature verification → anyone can POST fake events
5. Callback URL: SSRF via user-provided callback URL?
6. Proxy/tunnel: health check? What if tunnel dies mid-request?
```

### A-Chain 3: Response Manipulation
```
Goal: Extract more data than intended
1. Pagination bypass: page_size=999999 → dump entire DB?
2. Field selection: ?fields=password,secret → returns sensitive fields?
3. Sort/filter injection: ?sort=;DROP TABLE → SQL in sort param?
4. Include/expand: ?include=user.password → nested relation leaks?
5. Error verbosity: invalid request → returns SQL error with table structure?
6. Content-type confusion: request XML when expecting JSON → parser differential?
```

---

## 🤖 Bot Projects

### B-Chain 1: Message Injection
```
Goal: Make bot execute unintended actions
1. Command injection: message contains bot command prefix → triggers action?
2. Prompt injection (AI bots): "ignore previous instructions and..." → bypasses system prompt?
3. Mention/tag abuse: @bot in rapid succession → flood bot's queue?
4. Media message: send image/file → bot crashes on unexpected content type?
5. Unicode/emoji: special chars in message → encoding crash?
6. Long message: 10000 chars → buffer overflow or timeout?
```

### B-Chain 2: Bot State Abuse
```
Goal: Corrupt bot's conversation state
1. Concurrent conversations: bot mixes up context between users?
2. Session timeout: old session data leaks into new conversation?
3. Cancel mid-flow: start multi-step command → cancel → start different command → state leak?
4. Duplicate message: WeChat resends → bot processes twice? (dedup check?)
5. Group vs DM: bot behaves differently? Group command leaks DM data?
6. Rate limit: 100 messages/sec → bot crashes? Queue overflow?
```

---

## 🔧 WeChat Projects

### W-Chain 1: OAuth & Identity Attack
```
Goal: Impersonate users or steal sessions
1. OAuth state parameter: missing → CSRF login attack
2. OAuth callback: accepts any redirect_uri → open redirect → token theft
3. access_token: cached? Auto-refresh on expiry? Leaked in frontend?
4. openid in URL/response: can I use someone else's openid to login as them?
5. JS-SDK signature: uses fixed URL instead of current page URL → signature mismatch or reuse
6. Union ID vs OpenID confusion: wrong ID used for cross-platform identity
```

### W-Chain 2: WebView Compatibility Attack
```
Goal: Find features that break in WeChat's browser
1. ES6+ syntax: optional chaining ?., nullish coalescing ??, template literals `` → crash in old WebView
2. CSS: backdrop-filter without -webkit- prefix → invisible
3. API: fetch() → some old WebView needs XMLHttpRequest
4. Cache: WeChat extremely sticky cache → old JS served after update (need ?v=N)
5. Image upload: large blank-area PNG → WeChat decode failure → use JPEG
6. Payment: wx.chooseWXPay timing → must wait for bridge ready
```

### W-Chain 3: Mini-Program / H5 Hybrid Attack
```
Goal: Exploit the boundary between H5 and native
1. postMessage: origin check? Can malicious page send fake messages?
2. localStorage: shared between H5 pages on same domain → data leak between apps
3. Navigation: onclick JS redirect → fails in WebView → must use <a href>
4. 302 redirect with params → params lost in WeChat → use localStorage instead
5. Mixed content: HTTPS page loads HTTP resource → blocked silently
6. CDN: jsdelivr unreliable in China → local fallback?
```

---

## 📈 Platform / Multi-Service Projects

### P-Chain 1: Cross-Service Trust Attack
```
Goal: Exploit trust between microservices
1. Internal API: no auth because "only called internally" → but exposed on public port?
2. Service-to-service token: hardcoded? Rotated? Scoped?
3. Shared database: Service A writes, Service B reads → schema mismatch after update?
4. Event bus: can I publish fake events? Subscriber validates sender?
5. Config service: who can update? Change propagation delay → inconsistent state?
6. Health check endpoint: leaks internal topology/versions?
```

### P-Chain 2: Multi-Tenant Isolation
```
Goal: Break tenant boundaries
1. Database: tenant_id in every WHERE clause? Or shared tables without filter?
2. File storage: tenant A can access tenant B's uploads via path guessing?
3. Cache: Redis keys prefixed with tenant_id? Or shared namespace?
4. Background jobs: job for tenant A runs with tenant B's context?
5. Admin: super-admin can impersonate tenant → audit logged?
6. Subdomain/path routing: tenant routing bypass via Host header manipulation?
```

---

# PART 3: BLUE TEAM DEFENSE VERIFICATION

For EVERY red team finding, verify all 4 layers:

## Layer 1: Prevention
| Attack Category | Expected Defense | Verification |
|----------------|-----------------|-------------|
| Brute force | IP lockout after N failures | Try 10 wrong passwords |
| Payload bomb | express.json({limit:'100kb'}) | Send 1MB body |
| Rate abuse | Per-IP/per-key rate limiter | 100 req/sec burst |
| SQL injection | All prepare(), no string concat | grep for concat patterns |
| XSS | esc() output + strip input | Inject `<script>` in all user fields |
| CORS | Whitelist, not origin:true | Check cors() config |
| Token replay | Expiry check + invalidation on logout | Reuse old token |
| IDOR | Ownership check in every query | Access other user's resource by ID |
| Prototype pollution | Object.create(null) for lookup maps | Send __proto__ in body |
| Path traversal | Sanitize filenames, no .. allowed | Upload filename with ../ |

## Layer 2: Detection
```
For each attack surface, verify:
- Failed auth attempts: logged with IP + timestamp?
- Anomalous patterns: logged? (100 req/sec, cost=0, negative amounts, unusual export size)
- Admin dashboard: shows real-time anomalies?
- Error rate spike: monitored? Alerted?
- Data modification: audit trail? (who changed what, when)
```

## Layer 3: Containment
```
For each critical finding, assess blast radius:
- Single user affected? All users? Full server crash?
- Per-user limits exist? (daily caps, transaction maximums)
- Kill switch available? (disable endpoint, block IP, maintenance mode)
- Can damage be isolated without full shutdown?
- Graceful degradation: if one service fails, do others survive?
```

## Layer 4: Recovery
```
For the project as a whole:
- Database backups: exist? Frequency? Tested restore?
- Point-in-time recovery: can rollback to specific timestamp?
- Individual transaction rollback: possible?
- Affected user identification: audit log enables this?
- Incident playbook: documented steps for common scenarios?
- Data export: users can export their data for independent backup?
```

---

# PART 4: EXECUTION GUIDE

## Step-by-step for the auditor:

1. From Phase 1 dissection, identify project type(s)
2. Run ALL 4 Universal Chains — document findings
3. Run type-specific chains matching the project — document findings
4. For each 🔴 Critical finding: verify all 4 Blue Team layers
5. For each 🟡 Medium finding: verify Layer 1 (Prevention) at minimum
6. Compile findings with: chain reference, severity, cause, fix, file location
7. Prioritize fixes: 🔴 first, then 🟡 with easy fixes, then remaining 🟡, then 🟢
