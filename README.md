# Lock-Out-Accounts-if-Not-Currently-in-Use

---

**STEP 1 — Check server uptime and performance_schema status**
```sql
SELECT NOW() - INTERVAL variable_value SECOND AS started_at
FROM information_schema.global_status
WHERE variable_name = 'Uptime';
```
```sql
SHOW VARIABLES LIKE 'performance_schema';
```

| Condition | What it means |
|---|---|
| `performance_schema = ON` | Can use connection counts in Step 5 |
| `performance_schema = OFF` | Skip Step 5 — rely on Steps 4, 6, 7 only |
| Server restarted recently + `performance_schema = ON` | Connection counts unreliable — rely on Steps 4, 6, 7 |

---

**STEP 2 — Run the full benchmark query**
```sql
SELECT CONCAT(user, '@', host, ' => ', JSON_DETAILED(priv))
FROM mysql.global_priv;
```
This is verbose but is the exact benchmark query. Use Step 3 and 4 for cleaner assessment.

---

**STEP 3 — Get raw lock status for all accounts**
```sql
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked,
       JSON_EXTRACT(priv, '$.password_expired') AS password_expired,
       JSON_EXTRACT(priv, '$.password_last_changed') AS password_last_changed
FROM mysql.global_priv
ORDER BY user, host;
```

Note any accounts where:
- `account_locked` is `NULL` — these are roles like `PUBLIC`, not login accounts
- `account_locked` is `false` — these need further investigation
- `account_locked` is `true` — these are correctly disabled

---

**STEP 4a — Get full account assessment with human-readable dates**
```sql
SELECT g.user, g.host,
       JSON_EXTRACT(g.priv, '$.account_locked') AS account_locked,
       JSON_EXTRACT(g.priv, '$.password_expired') AS password_expired,
       FROM_UNIXTIME(JSON_EXTRACT(g.priv, '$.password_last_changed'))
           AS password_last_changed
FROM mysql.global_priv g
ORDER BY g.user, g.host;
```

Assess every row returned:

| Condition | Status |
|---|---|
| `PUBLIC` role with all NULLs | ⚠️ Role not a user — check privileges in Step 4b |
| Reserved account + `account_locked = true` | ✅ PASS |
| Reserved account + `account_locked = false` | ❌ FAIL |
| Any account + `password_last_changed` NULL + unlocked | ❌ FAIL |
| Any account + `password_last_changed` over 1 year ago + unlocked | ❌ FAIL |
| Any account + `account_locked = false` + recent password change | ⚠️ Verify in Steps 7 and 8 |
| Any account + `account_locked = true` | ✅ PASS |

---

**STEP 4b — Check PUBLIC role privileges**
```sql
SHOW GRANTS FOR PUBLIC;
```

| Condition | Status |
|---|---|
| Returns `USAGE` only or no rows | ✅ PASS — no privileges assigned to all users |
| Returns any `SELECT`, `INSERT`, `UPDATE` or other data privilege | ❌ FAIL |
| Returns `SUPER` or any admin privilege | ❌ CRITICAL FAIL |

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
| `total_connections > 0` + `account_locked = true` | ❌ FAIL — active but locked, will cause errors |
| All counts = 0 + server recently restarted | ⚠️ Unreliable — skip and rely on Steps 7 and 8 |

**If performance_schema = OFF skip this step entirely.**

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
| `PUBLIC` shows NULL for `account_locked` | ✅ Expected — it is a role, assess via Step 4a |
| Reserved account missing from output entirely | ⚠️ Verify it exists with `SELECT user FROM mysql.global_priv` |

---

**STEP 7 — Check error log for recent account activity**
```sql
-- Find the log file location first
SHOW VARIABLES LIKE 'log_error';
```

Then on the OS:
```bash
# Use the path returned above, or try these common locations
grep -iE "connect|login|access denied" /var/log/mariadb/mariadb.log 2>/dev/null | tail -100
grep -iE "connect|login" /var/log/mysql/error.log 2>/dev/null | tail -100
grep -iE "connect|login" /var/log/mariadb/*.log 2>/dev/null | tail -100
```

| Condition | Status |
|---|---|
| Username appears in recent logs + account unlocked | ✅ Likely active — confirm in Step 8 |
| Username never appears in any log + account unlocked | ❌ FAIL — lock until verified |

---

**STEP 8 — Manual verification for remaining unlocked accounts**

For every account that is unlocked and not confirmed active from Steps 5 or 7, ask the DBA or infrastructure team:

```
☐ What application or person uses this account?
☐ When was it last used?
☐ Is it still needed?
☐ If unsure — lock it now, unlock only when confirmed needed
```

| Answer | Status |
|---|---|
| Confirmed active + named owner identified | ✅ PASS |
| Cannot confirm active + no named owner | ❌ FAIL — lock immediately |
| Confirmed no longer needed | ❌ FAIL — drop or lock immediately |

---

**STEP 9 — List all high priority failures in one query**
```sql
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
AND g.user NOT IN ('mysql.sys','mysql.session','mysql.infoschema','mariadb.sys','PUBLIC')
ORDER BY g.user, g.host;
```

| Condition | Status |
|---|---|
| No rows returned | ✅ No stale unlocked accounts |
| Any rows returned | ❌ FAIL — each row must be locked or verified |

---

**STEP 10 — Remediate all failures**
```sql
-- Lock inactive or unverified accounts
ALTER USER 'username'@'host' ACCOUNT LOCK;

-- Lock reserved accounts if not already locked
ALTER USER 'mysql.sys'@'localhost' ACCOUNT LOCK;
ALTER USER 'mysql.session'@'localhost' ACCOUNT LOCK;
ALTER USER 'mysql.infoschema'@'localhost' ACCOUNT LOCK;

-- Drop accounts confirmed as no longer needed
DROP USER 'username'@'host';

-- Verify all locks applied correctly
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked
FROM mysql.global_priv
ORDER BY user, host;
```

---

**Overall pass/fail decision tree — work top to bottom:**

```
performance_schema = OFF?                         → Note as a separate finding
PUBLIC role has privileges beyond USAGE?          YES → FAIL
Any reserved account not locked?                  YES → FAIL
Any account with NULL password_last_changed
  and unlocked?                                   YES → FAIL
Any account with password unchanged > 1 year
  and unlocked?                                   YES → FAIL
Any unlocked account not verifiable as active
  by DBA or log evidence?                         YES → FAIL
All reserved accounts locked?                     YES → continue
PUBLIC role has USAGE only?                       YES → continue
All active accounts confirmed by DBA or logs?     YES → continue
All inactive or unknown accounts locked?          YES → PASS
```
