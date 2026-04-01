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

**STEP 2 — Run a focused query easier to assess**
```sql
-- Clean view of lock status for all accounts
SELECT user, host,
       JSON_EXTRACT(priv, '$.account_locked') AS account_locked,
       JSON_EXTRACT(priv, '$.password_expired') AS password_expired,
       JSON_EXTRACT(priv, '$.password_last_changed') AS password_last_changed
FROM mysql.global_priv
ORDER BY user, host;
```

---

**STEP 3 — Check last login activity per account**
```sql
-- Check when each user last connected (requires performance_schema)
SELECT user, host, 
       MAX(event_time) AS last_login
FROM performance_schema.events_statements_history_long
GROUP BY user, host
ORDER BY last_login DESC;
```

Or alternatively:
```sql
SELECT user, host, last_login 
FROM information_schema.user_statistics 2>/dev/null;
```

---

**How to assess each account:**

**MariaDB reserved accounts** — must be locked:

| Account | Expected status | Condition | Status |
|---|---|---|---|
| `mysql.sys@localhost` | Must be locked | `account_locked = true` | ✅ PASS |
| `mysql.session@localhost` | Must be locked | `account_locked = true` | ✅ PASS |
| `mysql.infoschema@localhost` | Must be locked | `account_locked = true` | ✅ PASS |
| Any reserved account with `account_locked = false` | | | ❌ FAIL |

---

**Active accounts — must be unlocked but justified:**

| Condition | Status |
|---|---|
| Account is unlocked AND actively used AND maps to current person/app | ✅ PASS |
| Account is unlocked AND has recent login activity | ✅ PASS |
| Account is unlocked AND has no login activity in 30+ days | ❌ FAIL — should be locked |
| Account is unlocked AND owner has left organisation | ❌ FAIL — should be dropped |
| Account is unlocked AND cannot be attributed to any person/app | ❌ FAIL — lock immediately |

---

**Inactive accounts — must be locked:**

| Condition | Status |
|---|---|
| Account is locked AND not currently needed | ✅ PASS |
| Account is locked AND `password_expired = true` | ✅ PASS |
| Account exists with no activity and is NOT locked | ❌ FAIL |

---

**STEP 4 — Cross-check with your earlier user list**

Take the 12 users you found earlier (10 weak SHA1 + 2 ed25519) and for each one ask:

```
☐ Is this account currently in active use?
☐ Is there a known person or application using it right now?
☐ Has it been used in the last 30 days?
☐ If not in use — is it locked?
```

| Answer | Status |
|---|---|
| In use + unlocked + identifiable owner | ✅ PASS |
| Not in use + locked | ✅ PASS |
| Not in use + unlocked | ❌ FAIL |
| In use + cannot identify owner | ❌ FAIL |

---

**Overall pass/fail logic:**

```
Any reserved account not locked?              YES → FAIL
Any inactive account not locked?              YES → FAIL
Any account with no login in 30+ days unlocked? YES → FAIL
Any account with no identifiable owner unlocked? YES → FAIL
All active accounts have justified unlock?    YES → PASS
All inactive accounts locked?                 YES → PASS
```

---

**Remediation:**
```sql
-- Lock an inactive account
ALTER USER 'username'@'host' ACCOUNT LOCK;

-- Verify it is locked
SELECT user, host, 
       JSON_EXTRACT(priv, '$.account_locked') AS locked
FROM mysql.global_priv
WHERE user = 'username';

-- Unlock when account becomes active again
ALTER USER 'username'@'host' ACCOUNT UNLOCK;

-- Drop accounts with no current owner
DROP USER 'username'@'host';
```
