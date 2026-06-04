# SQL Injection Attack, Querying the Database Type and Version on MySQL and Microsoft

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

The goal is to display the database version string.

---

## 1. Detection

- Opened the URL and clicked the `Gifts` button under **Refine your search**.
- Captured the `GET` request sent to `/filter?category=Gifts`.
- Appended `"` after `Gifts` to probe for errors — no response. Tried `'` instead, which returned an `Internal Server Error`, indicating that user input is not sanitized and the backend likely isn't using prepared statements. This confirms the category value is wrapped in single quotes inside the backend query.
- Tested `Gifts' AND true--+` — returned results. Switching to `Gifts' AND false--+` returned no results.

---

## 2. Extracting Column Count

- With injection confirmed, the next step was determining how many columns the query returns. Two approaches were used: `ORDER BY` and `UNION`.
- **ORDER BY approach** — `Gifts' ORDER BY 1--+` returned results normally. Incrementing the value (`2`, `3`...) until `Gifts' ORDER BY 3--+` threw an `Internal Server Error`, narrowing the column count down. The page was also showing a title and description per item, hinting the query returns only **2 columns**.
- **UNION approach** — `Gifts' UNION SELECT NULL,NULL--+` returned no error. Adding a third `NULL` caused an error, confirming exactly **2 columns**.

---

## 3. Extracting the Database Version String

- With the column count confirmed, the next step was identifying the data types each column accepts.
- Used the payload `Gifts' UNION SELECT 1,2--+` — no error, and both values were reflected in the response. Wrapping them in quotes produced the same result, confirming both columns accept either integers or strings.
- Extracted the version string using MySQL's `version()` function:

```sql
Gifts' UNION SELECT version(),NULL--+
```

- The response returned the version string `8.0.42-0ubuntu0.20.04.1`, solving the lab.
