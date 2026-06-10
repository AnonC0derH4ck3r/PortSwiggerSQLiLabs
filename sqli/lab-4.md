# SQL injection attack, querying the database type and version on MySQL and Microsoft

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

---

## 1. Detection

- Opened the URL and clicked the `Gifts` button under **Refine your search**.
- Captured the `GET` request sent to `/filter?category=Gifts`.
- Appended `"` after `Gifts` to probe for errors - no response. Tried `'` instead, which returned an `Internal Server Error`, indicating that user input is not sanitized and the backend likely isn't using prepared statements. This confirms the category value is wrapped in single quotes inside the backend query.
- Tested `Gifts' AND true--+` - returned results. Switching to `Gifts' AND false--+` returned no results.

## 2. Extracting Column Count
- With injection confirmed, the next step was determining how many columns the query returns. Two approaches were used: `ORDER BY` and `UNION`.
- **ORDER BY approach** - `Gifts' ORDER BY 1--+` returned results normally. Incrementing the value (`2`, `3`...) until `Gifts' ORDER BY 3--+` threw an `Internal Server Error`, narrowing the column count down. Also, the page was showing title and description in Gifts's category, hinting the query is returning only **2 columns**.
- **UNION approach** — `Gifts' UNION SELECT NULL,NULL FROM DUAL--+` returned no error. Adding a third `NULL` caused an error, confirming exactly **2 columns**.

## 3. Extracting Database version string

- Since, now we know that the backend query is returning **2 columns**.
- It's time to identify what exact data-types the query is returning.
- I used the following payload to enumerate the columns data-types - `Gifts' UNION SELECT 1,2 FROM DUAL--+`. This query does not returned an error, and displayed the number `1` and `2`. I tried switching the data-types to TEXT/STRING by wrapping them in quotes, still no change in response.
- Without Further ado, I wrote a payload to extract the Database Version string. The final payload - `Gifts' UNION SELECT version(),0x00--+`.
This query results in the database version string `8.0.42-0ubuntu0.20.04.1`. Hence, the challenge has been solved.