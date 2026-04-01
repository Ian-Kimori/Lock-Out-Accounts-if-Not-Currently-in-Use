# Lock-Out-Accounts-if-Not-Currently-in-Use

---

**STEP 1 — Check server uptime and performance_schema status**
```sql
-- Check when server last restarted
SELECT NOW() - INTERVAL variable_value SECOND AS started_at
FROM information_schema.global_status
WHERE variable_name = 'Uptime';
```
```sql
-- Check if performance_schema is on or off
SHOW VARIABLES LIKE 'performance_schema';
```

| Condition | What it means for the rest of the assessment |
|---|---|
| `performance_schema = ON` | Can use connection counts in Step 3 |
| `performance_schema = OFF` | Skip Step 3 — rely on Steps 2, 4, 5 only |

---

**STEP 2 — Run the benchmark query**
```sql
SELECT CONCAT(user, '@', host, ' => ', JSON_DETAILED(priv)) 
FROM mysql.global_priv;
```

This returns the full privilege JSON for every account. It is verbose — use Step 2 for a cleaner view.

---

**STEP 3 — Get all accounts with lock status**
```sql
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked,
       JSON_EXTRACT(priv, '$.password_expired') AS password_expired,
       JSON_EXTRACT(priv, '$.password_last_changed') AS password_last_changed
FROM mysql.global_priv
ORDER BY user, host;
```

**STEP 4 — Get all accounts with lock status and last password change**
```sql
SELECT g.user, g.host,
       JSON_EXTRACT(g.priv, '$.account_locked') AS account_locked,
       JSON_EXTRACT(g.priv, '$.password_expired') AS password_expired,
       FROM_UNIXTIME(JSON_EXTRACT(g.priv, '$.password_last_changed'))
           AS password_last_changed
FROM mysql.global_priv g
ORDER BY g.user, g.host;
```

Assess each row:

| Condition | Status |
|---|---|
| Reserved account + `account_locked = true` | ✅ PASS |
| Reserved account + `account_locked = false` | ❌ FAIL |
| Any account + `password_last_changed` is NULL + unlocked | ❌ FAIL |
| Any account + `password_last_changed` over 1 year ago + unlocked | ❌ FAIL |
| Any account + `account_locked = false` + recent `password_last_changed` | ⚠️ Verify in Step 4 |
| Any account + `account_locked = true` | ✅ PASS |

---

**STEP 5 — Check connection counts (only if performance_schema = ON)**
```sql
SELECT g.user, g.host,
       JSON_EXTRACT(g.priv, '$.account_locked') AS account_locked,
       COALESCE(a.total_connections, 0) AS total_connections,
       COALESCE(a.current_connections, 0) AS current_connections
FROM mysql.global_priv g
LEFT JOIN performance_schema.accounts a
       ON g.user = a.user AND g.host = a.host
ORDER BY g.user, g.host;
```

| Condition | Status |
|---|---|
| `total_connections > 0` + `account_locked = false` | ✅ PASS — active and unlocked |
| `total_connections = 0` + `account_locked = true` | ✅ PASS — inactive and locked |
| `total_connections = 0` + `account_locked = false` | ❌ FAIL — inactive but unlocked |
| `total_connections > 0` + `account_locked = true` | ❌ FAIL — active but locked |
| All counts = 0 + server recently restarted | ⚠️ Unreliable — rely on Steps 2, 4, 5 |

**If performance_schema = OFF skip this step entirely and proceed to Step 4.**

---

**STEP 6 — Check reserved accounts specifically**
```sql
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked
FROM mysql.global_priv
WHERE user IN (
  'mysql.sys',
  'mysql.session',
  'mysql.infoschema',
  'mariadb.sys',
  'PUBLIC'
);
```

| Condition | Status |
|---|---|
| All reserved accounts show `account_locked = true` | ✅ PASS |
| Any reserved account shows `account_locked = false` | ❌ FAIL |
| Reserved account missing from list entirely | ⚠️ Check if it exists |

---

**STEP 7 — Check error log for recent account activity**
```bash
# Find log location first
mysql -u root -e "SHOW VARIABLES LIKE 'log_error';"

# Then grep for connection activity
grep -iE "connect|login|access denied" /var/log/mariadb/mariadb.log 2>/dev/null | \
  tail -100

# Alternative locations
grep -iE "connect|login" /var/log/mysql/error.log 2>/dev/null | tail -100
grep -iE "connect|login" /var/log/mariadb/*.log 2>/dev/null | tail -100
```

For each username seen in the logs:

| Condition | Status |
|---|---|
| Username appears in recent logs + account unlocked | ✅ Likely active — verify with DBA |
| Username never appears in logs + account unlocked | ❌ FAIL — lock until verified |

---

**STEP 8 — Manual verification for remaining unlocked accounts**

For every account that is unlocked and cannot be confirmed active from Steps 3 or 5, ask the DBA or infrastructure team:

```
☐ What application or person uses this account?
☐ When was it last used?
☐ Is it still needed?
☐ If unsure — lock it now and unlock when confirmed needed
```

| Answer | Status |
|---|---|
| Confirmed active + named owner identified | ✅ PASS |
| Cannot confirm active + no named owner | ❌ FAIL |
| Confirmed no longer needed | ❌ FAIL — drop or lock immediately |

---

**STEP 9 — Identify and list all failures**
```sql
-- Accounts that are unlocked with no recent password change
-- These are your highest priority failures
SELECT g.user, g.host,
       JSON_EXTRACT(g.priv, '$.account_locked') AS account_locked,
       FROM_UNIXTIME(JSON_EXTRACT(g.priv, '$.password_last_changed'))
           AS password_last_changed
FROM mysql.global_priv g
WHERE JSON_EXTRACT(g.priv, '$.account_locked') = false
AND (
  JSON_EXTRACT(g.priv, '$.password_last_changed') IS NULL
  OR FROM_UNIXTIME(JSON_EXTRACT(g.priv, '$.password_last_changed')) 
     < NOW() - INTERVAL 365 DAY
)
AND g.user NOT IN ('mysql.sys','mysql.session','mysql.infoschema','mariadb.sys')
ORDER BY g.user, g.host;
```

| Condition | Status |
|---|---|
| No rows returned | ✅ No stale unlocked accounts |
| Any rows returned | ❌ FAIL — each row needs locking or verification |

---

**STEP 10 — Remediate all failures found**
```sql
-- Lock inactive or unverified accounts
ALTER USER 'username'@'host' ACCOUNT LOCK;

-- Lock reserved accounts if not already locked
ALTER USER 'mysql.sys'@'localhost' ACCOUNT LOCK;
ALTER USER 'mysql.session'@'localhost' ACCOUNT LOCK;
ALTER USER 'mysql.infoschema'@'localhost' ACCOUNT LOCK;

-- Drop accounts confirmed as no longer needed
DROP USER 'username'@'host';

-- Verify all locks applied
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked
FROM mysql.global_priv
ORDER BY user, host;
```

---

**Overall pass/fail decision:**

```
performance_schema OFF?                          → Note as separate finding
Any reserved account not locked?                 YES → FAIL
Any account with NULL password_last_changed
and unlocked?                                    YES → FAIL
Any account with password unchanged > 1 year
and unlocked?                                    YES → FAIL
Any unlocked account not verifiable as
active by DBA/log evidence?                      YES → FAIL
All reserved accounts locked?                    YES → continue
All active accounts confirmed by DBA/logs?       YES → continue
All inactive/unknown accounts locked?            YES → PASS
```
