# SQL Injection UNION Attack — Determining the Number of Columns

`#UNION` `#SQL-injection` `#application-logic`

## What is a UNION attack?

A type of SQL injection that allows the user to append a second `SELECT` against any table the database user can reach, returning its results through the original query's output.

## UNION attack basics

For a `UNION` query to work, two key requirements must be met:

- The individual queries must return the same number of columns.
- The data types in each column must be compatible between the individual queries.

To carry out a SQL injection `UNION` attack, make sure your attack meets these two requirements. This normally involves finding out:

- How many columns are being returned from the original query.
- Which columns returned from the original query are of a suitable data type to hold the results from the injected query.

## Determining the number of columns required

**`ORDER BY`** — you can submit `ORDER BY` payloads to determine how many columns are in a given SQL table. You'll get an error message when you hit the limit — or you might not, if the app suppresses database errors. That's the blind case, where you can't read the result directly. It gets complicated during a blind SQLi UNION attack, in which case you could use a time-based payload to determine how many columns. There's always a way.

```sql
ORDER BY 1--
ORDER BY 2--
ORDER BY 3--
-- and so on
```

**`NULL` payload** — you can hit it with a `NULL` payload too.

```sql
UNION SELECT NULL--
UNION SELECT NULL, NULL--
UNION SELECT NULL, NULL, NULL--
-- and so on
```

Using `NULL` is useful because the values from the injected `SELECT` must be type-compatible between the original and injected queries. `NULL` is compatible with every data type, so it maximizes the chances the payload succeeds in determining the number of columns.

---

## Lab: SQL injection UNION attack, determining the number of columns returned by the query

**Instructions:** This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. The first step is to determine the number of columns being returned by the query. You'll then use this technique in subsequent labs to construct the full attack.

To solve the lab, determine the number of columns returned by the query by performing a SQL injection UNION attack that returns an additional row containing null values.

### Approach

For this lab, I needed to figure out how many columns were in the query. So I performed a UNION attack that revealed the column count.

1. Opened Burp Suite, loaded the lab URL in the Burp browser so traffic routed through the proxy.
    
2. Found the `GET /filter?category=...` request in **Proxy → HTTP history** and pressed `Ctrl+R` to send it to **Repeater**.
    
3. Modified the `category` filter with a `UNION` injection:
    
    ```
    GET /filter?category=Clothing%2c+shoes+and+accessories%27+UNION+SELECT+NULL--
    ```
    
4. Iterated, adding one more `NULL` to the column list each time, until the `500` error cleared:
    
    ```
    GET /filter?category=Clothing%2c+shoes+and+accessories%27+UNION+SELECT+NULL%2cNULL%2cNULL--
    ```
    

### Result

There were **three columns**. The flip from `500 Internal Server Error` to `200 OK` confirmed the column count matched. Lab solved.

### Encoding notes

|Character|URL-encoded|Role in payload|
|---|---|---|
|`'`|`%27`|breaks out of the string|
|`,`|`%2c`|separates columns|
|space|`+`|separates SQL keywords|
|`+` (literal)|`%2b`|NOT wanted — sends a literal plus|