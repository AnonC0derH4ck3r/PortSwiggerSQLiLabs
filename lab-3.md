# SQL injection attack, querying the database type and version on Oracle

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

---

## 1. Detection

- Opened the URL and clicked the `Lifestyle` button under **Refine your search**.
- Captured the `GET` request sent to `/filter?category=Lifestyle`.
- Appended `"` after `Lifestyle` to probe for errors — no response. Tried `'` instead, which returned an `Internal Server Error`, indicating that user input is not sanitized and the backend likely isn't using prepared statements. This confirms the category value is wrapped in single quotes inside the backend query.
- Tested `Lifestyle' AND 1=1--+` — returned results. Switching to `Lifestyle' AND 1=2--+` returned no results, since `1=2` is always false. Further tested `Lifestyles' OR 1=1--+` — returned results, and `Lifestyle' OR 1=2--+` returned none. This confirms SQL injection exists.

## 2. Extracting Column Count

- With injection confirmed, the next step was determining how many columns the query returns. Two approaches were used: `ORDER BY` and `UNION`.
- **ORDER BY approach** — `Lifestyle' ORDER BY 1--+` returned results normally. Incrementing the value (`2`, `3`...) until `Lifestyle' ORDER BY 3--+` threw an `Internal Server Error`, narrowing the column count down.
- **UNION approach** — `Lifestyle' UNION SELECT NULL,NULL FROM DUAL--+` returned no error. Adding a third `NULL` caused an error, confirming exactly **2 columns**.
---

## 2. Extracting Database version string

- Since, now we know that the backend query is returning **2 columns**.
- It's time to identify what exact data-types the query is returning.
- I used the following payload to enumerate the columns data-types - `Lifestyle' UNION SELECT 1,2 FROM DUAL--+`. This query returned an error, means one or both the columns aren't INTEGERS. Modifying the payload to `Lifestyle' UNION SELECT '1',2 FROM DUAL--+`, still results an error, means the second column is also not an INTEGER type.
- Furthermore, I modified the payload to `Lifestyle' UNION SELECT '1','2' FROM DUAL--+` which results in both `1` and `2` being printed in the response, this means both the columns are getting reflected and both of them are TEXT.
- Without Further ado, I wrote a payload to extract the Database Version string. The final payload - `Lifestyle' UNION SELECT BANNER,'2' FROM v$VERSION--+`.
- This query results in the database version string `CORE	11.2.0.2.0	Production`. Hence, the challenge has been solved.