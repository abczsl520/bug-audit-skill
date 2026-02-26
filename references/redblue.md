# Red Team / Blue Team Playbook

## Red Team: Attack Chains

Don't test endpoints in isolation. Build multi-step attack chains that combine weaknesses.

### Chain 1: Skip-Pay-Collect
```
Goal: Get rewards without paying
1. Find the "pay" API (buy, start-raid, purchase)
2. Find the "collect" API (raid-result, quest-complete, claim-reward)
3. Try calling collect WITHOUT calling pay first
4. Try calling pay with cost=0 or cost=-1, then collect with max reward
5. Try calling pay once, then collect multiple times
6. Check: is there a one-time token linking pay→collect?
```
If no token: 🔴 Critical. Attacker can farm infinite resources.

### Chain 2: Auth Bypass Escalation
```
Goal: Access admin or other users' data
1. List all endpoints that DON'T require auth → what data do they leak?
2. Try calling auth-required endpoints without token → does it 401 or silently fail?
3. Try using User A's token to access User B's data (IDOR)
4. Try replaying an expired token → does the server check expiry?
5. Check admin endpoints: is the password in GET query? In code? Brute-forceable?
6. Check: does guest-login response leak openid/internal IDs that enable impersonation?
```

### Chain 3: Economic Loop
```
Goal: Generate infinite currency
1. Map all resource inflows and outflows (from Table 6)
2. Find any cycle: buy item A for X coins → sell/convert for Y coins where Y > X
3. Check quest/achievement rewards: can they be claimed repeatedly?
4. Check daily reset: does it actually reset? Timezone bugs = double-claim window
5. Check trade between accounts: can I transfer to alt, then rollback main?
6. Check: are there any "free" inflows with no cooldown or daily limit?
```

### Chain 4: State Confusion
```
Goal: Corrupt game state for advantage
1. Start action A (raid/battle/quest) → force-quit → start action B
2. Does state from A leak into B? (timers, flags, temporary buffs)
3. Open two browser tabs → perform same action simultaneously
4. Send rapid duplicate requests (double-click buy, race condition on balance check)
5. Trigger visibilitychange during critical state transition
6. Check: after game-over, can I still trigger reward-granting events?
```

### Chain 5: Injection Escalation
```
Goal: Execute code or corrupt data
1. Set nickname to <script>alert(1)</script> → check leaderboard, chat, admin panel
2. Set nickname to ' OR 1=1-- → check if any query uses string concat
3. Send {__proto__: {isAdmin: true}} in JSON body → check prototype pollution
4. Send 10MB JSON body → check payload size limit
5. Send array where string expected: {cost: [1,2,3]} → check type validation
6. Send deeply nested object: {a:{b:{c:{d:...}}}} → check recursion depth
```

### Chain 6: Rate & Resource Abuse
```
Goal: Denial of service or resource exhaustion
1. Call expensive endpoint (leaderboard, search, export) 100x/sec → rate limited?
2. Create 1000 Socket.IO connections → max connection limit?
3. Send 1000 chat messages/sec → message rate limit?
4. Create entities rapidly (AI snakes, game rooms, tokens) → cleanup?
5. Upload large files repeatedly → disk space limit?
6. Check: are there any endpoints that allocate memory proportional to input size?
```

## Blue Team: Defense Verification

For each red team finding, verify all 4 layers:

### Layer 1: Prevention
```
| Attack | Expected Defense | Check |
|--------|-----------------|-------|
| Brute force login | IP lockout after N failures | Try 10 wrong passwords from same IP |
| Payload bomb | express.json({limit:'100kb'}) | Send 1MB body |
| Rate abuse | Per-IP or per-key rate limiter | 100 requests in 1 second |
| SQL injection | All queries use prepare() | grep for string concatenation in SQL |
| XSS | esc() on output + strip on input | Set nickname with <script> tag |
| CORS bypass | Whitelist, not origin:true | Check cors() config |
| Token replay | Expiry check + one-time tokens | Reuse old token after logout |
```

### Layer 2: Detection
```
- Are failed auth attempts logged with IP?
- Are unusual patterns logged? (100 requests/sec, cost=0, negative amounts)
- Is there an admin dashboard showing anomalies?
- Are error rates monitored?
```

### Layer 3: Containment
```
- If an exploit succeeds, what's the blast radius?
  - Single user? All users? Server crash?
- Can the damage be isolated? (per-user limits, transaction caps)
- Is there a kill switch? (disable endpoint, block IP, maintenance mode)
```

### Layer 4: Recovery
```
- Database backups exist? How often? Tested restore?
- Can individual transactions be rolled back?
- Can affected user accounts be identified and restored?
- Is there an audit log of all state-changing operations?
```

## Attack Priority by Project Type

| Project Type | Priority Chains | Why |
|-------------|----------------|-----|
| 🎮 Game | Chain 1, 3, 4 | Economy exploits and state bugs are game-breaking |
| 📊 Data Tool | Chain 2, 5, 6 | Auth bypass leaks data; injection corrupts DB |
| 🔌 API Service | Chain 2, 5, 6 | Auth and rate abuse are primary threats |
| 🤖 Bot | Chain 5, 6 | Injection via messages; rate abuse via spam |
| 📈 Dashboard | Chain 2, 6 | Data access control; query amplification |
