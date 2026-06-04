# SQL Injection Attack, Querying the Database Type and Version on Oracle

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

The goal is to display the database version string.

---

## 1. Detection

- Opened the URL and clicked the `Lifestyle` button under **Refine your search**.
- Captured the `GET` request sent to `/filter?category=Lifestyle`.
- Appended `"` after `Lifestyle` to probe for errors — no response. Tried `'` instead, which returned an `Internal Server Error`, indicating that user input is not sanitized and the backend likely isn't using prepared statements. This confirms the category value is wrapped in single quotes inside the backend query.
- Tested `Lifestyle' AND 1=1--+` — returned results. Switching to `Lifestyle' AND 1=2--+` returned no results, since `1=2` is always false. Further tested `Lifestyle' OR 1=1--+` — returned results, and `Lifestyle' OR 1=2--+` returned none. This confirms SQL injection exists.

---

## 2. Extracting Column Count

- With injection confirmed, the next step was determining how many columns the query returns. Two approaches were used: `ORDER BY` and `UNION`.
- **ORDER BY approach** — `Lifestyle' ORDER BY 1--+` returned results normally. Incrementing the value (`2`, `3`...) until `Lifestyle' ORDER BY 3--+` threw an `Internal Server Error`, narrowing the column count down.
- **UNION approach** — `Lifestyle' UNION SELECT NULL,NULL FROM DUAL--+` returned no error. Adding a third `NULL` caused an error, confirming exactly **2 columns**.

---

## 3. Extracting the Database Version String

- With the column count confirmed, the next step was identifying the data types each column accepts.
- Used the payload `Lifestyle' UNION SELECT 1,2 FROM DUAL--+` — returned an error, meaning one or both columns don't accept integers. Modified to `Lifestyle' UNION SELECT '1',2 FROM DUAL--+` — still errored, ruling out integer for the second column as well.
- Modified the payload to `Lifestyle' UNION SELECT '1','2' FROM DUAL--+` — both values were reflected in the response, confirming both columns are of string type.
- With both columns confirmed as text, extracted the version string using Oracle's `v$VERSION` view:

```sql
Lifestyle' UNION SELECT BANNER,'2' FROM v$VERSION--+
```

- The response returned the version string `CORE 11.2.0.2.0 Production`, solving the lab.
