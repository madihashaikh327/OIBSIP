# SQL Injection Payload Log — DVWA (Low Security)

Environment: DVWA running locally via XAMPP, Security Level = Low
Module: SQL Injection

---

## Baseline Test (Normal Input)
**Input:**
```
1
```

**Query (inferred):**
```sql
SELECT first_name, last_name FROM users WHERE user_id = '1';
```

**Result:** Returned a single user's first and last name, as expected.
**Screenshot:** `screenshot_baseline.png`

---

## Payload 1
**Input:**
```
' OR '1'='1
```

**Query (inferred):**
```sql
SELECT first_name, last_name FROM users WHERE user_id = '' OR '1'='1';
```

**Result:** All user records were returned instead of a single one.

**Data exposed:** First and last names of every user stored in the `users` table.

**Analysis:** The injected `OR '1'='1'` clause is always true, so the `WHERE`
condition matched every row in the table. This demonstrates how unsanitized input
can override the intended filtering logic of a query.

**Screenshot:** `screenshot_payload1.png`

---

## Payload 2
**Input:**
```
' UNION SELECT user, password FROM users -- 
```

**Query (inferred):**
```sql
SELECT first_name, last_name FROM users WHERE user_id = ''
UNION SELECT user, password FROM users -- ';
```

**Result:** Usernames and password hashes were displayed instead of the expected
name output.

**Data exposed:** Contents of the `user` and `password` columns from the `users`
table — credential data that this form was never intended to expose.

**Analysis:** `UNION SELECT` allowed a second, attacker-controlled query to be
merged with the original query's output. Because the column count matched, the
database accepted the union and returned data from a completely different set of
columns than intended. The trailing `--` commented out the rest of the original
query to avoid a syntax error.

**Screenshot:** `screenshot_payload2.png`

---

## Summary

| Payload | Technique | Data Exposed |
|---|---|---|
| `' OR '1'='1` | Boolean-based filter bypass | All user names |
| `' UNION SELECT user, password FROM users -- ` | UNION-based extraction | Usernames + password hashes |

## Conclusion
Both payloads succeeded because DVWA (Low) concatenates user input directly into
SQL queries without sanitization or parameterization. The fix is to use prepared
statements, which treat all user input strictly as data and never as executable
SQL syntax (see README.md for details and code example).
