# Lock-Out-Accounts-if-Not-Currently-in-Use

Here is the complete test procedure with explicit pass/fail conditions:

---

**STEP 1 — Run the benchmark query**
```sql
SELECT CONCAT(user, '@', host, ' => ', JSON_DETAILED(priv)) 
FROM mysql.global_priv;
```

This returns the full privilege JSON for every account. It is verbose — use Step 2 for a cleaner view.

---

**STEP 2 — Get all accounts with lock status**
```sql
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked,
       JSON_EXTRACT(priv, '$.password_expired') AS password_expired,
       JSON_EXTRACT(priv, '$.password_last_changed') AS password_last_changed
FROM mysql.global_priv
ORDER BY user, host;
```

---

**STEP 3 — Get connection activity for all accounts**
```sql
SELECT user, host,
       current_connections,
       total_connections
FROM performance_schema.accounts
WHERE user IS NOT NULL
ORDER BY total_connections DESC;
```

---

**STEP 4 — Combine lock status and activity in one view**
```sql
SELECT g.user, g.host,
       JSON_EXTRACT(g.priv, '$.account_locked') AS account_locked,
       JSON_EXTRACT(g.priv, '$.password_expired') AS password_expired,
       COALESCE(a.total_connections, 0) AS total_connections,
       COALESCE(a.current_connections, 0) AS current_connections
FROM mysql.global_priv g
LEFT JOIN performance_schema.accounts a
       ON g.user = a.user AND g.host = a.host
ORDER BY g.user, g.host;
```

---

**STEP 5 — Specifically check reserved accounts are locked**
```sql
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked
FROM mysql.global_priv
WHERE user IN ('mysql.sys','mysql.session','mysql.infoschema',
               'mariadb.sys','PUBLIC');
```

---

**STEP 6 — Identify accounts that should be locked but are not**
```sql
SELECT g.user, g.host,
       JSON_EXTRACT(g.priv, '$.account_locked') AS account_locked,
       COALESCE(a.total_connections, 0) AS total_connections
FROM mysql.global_priv g
LEFT JOIN performance_schema.accounts a
       ON g.user = a.user AND g.host = a.host
WHERE JSON_EXTRACT(g.priv, '$.account_locked') = false
AND COALESCE(a.total_connections, 0) = 0
AND g.user NOT IN ('root')
ORDER BY g.user;
```

---

**How to assess the output of each step:**

**STEP 1 — Lock status assessment:**

| Condition | Status |
|---|---|
| Reserved account has `account_locked = true` | ✅ PASS |
| Reserved account has `account_locked = false` | ❌ FAIL |
| Active user account has `account_locked = false` | ✅ PASS — if justified |
| Unused account has `account_locked = false` | ❌ FAIL |

---

**STEP 2 — Connection activity assessment:**

| Condition | Status |
|---|---|
| `total_connections > 0` and account unlocked | ✅ PASS — account in use |
| `total_connections = 0` and account unlocked | ❌ FAIL — never used, should be locked |
| `total_connections = 0` and account locked | ✅ PASS — correctly disabled |
| `current_connections > 0` and account unlocked | ✅ PASS — actively in use right now |

---

**STEP 3 — Combined assessment per account:**

| account_locked | total_connections | Assessment | Status |
|---|---|---|---|
| `false` | Greater than 0 | Active and unlocked | ✅ PASS |
| `true` | 0 | Inactive and locked | ✅ PASS |
| `false` | 0 | Inactive but unlocked | ❌ FAIL |
| `true` | Greater than 0 | Active but locked | ❌ FAIL — will cause app errors |

---

**STEP 4 — Reserved accounts assessment:**

| Account | Expected | Condition | Status |
|---|---|---|---|
| `mysql.sys` | Locked | `account_locked = true` | ✅/❌ |
| `mysql.session` | Locked | `account_locked = true` | ✅/❌ |
| `mysql.infoschema` | Locked | `account_locked = true` | ✅/❌ |
| `mariadb.sys` | Locked | `account_locked = true` | ✅/❌ |
| `PUBLIC` | Locked | `account_locked = true` | ✅/❌ |
| Any of the above with `false` | | | ❌ FAIL |

---

**STEP 5 — Unlocked inactive accounts:**

| Condition | Status |
|---|---|
| No rows returned | ✅ PASS — no unlocked inactive accounts |
| Any rows returned | ❌ FAIL — each row is an account that needs locking |

---

**Overall pass/fail logic:**

```
Any reserved account not locked?                YES → FAIL
Any account unlocked with zero connections?      YES → FAIL
Any account locked but actively connecting?      YES → FAIL — investigate
Any account with no identifiable owner unlocked? YES → FAIL
All reserved accounts locked?                    YES → continue
All active accounts unlocked and justified?      YES → continue
All inactive accounts locked?                    YES → PASS
```

---

**Remediation for any failures found:**
```sql
-- Lock an inactive account
ALTER USER 'username'@'host' ACCOUNT LOCK;

-- Lock a reserved account
ALTER USER 'mysql.sys'@'localhost' ACCOUNT LOCK;

-- Verify lock applied
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked
FROM mysql.global_priv
WHERE user = 'username';

-- Drop accounts with no current owner
DROP USER 'username'@'host';
```

---

**Run the steps in order — Step 3 and Step 5 will give you the clearest picture of where the failures are.**
