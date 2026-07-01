---
layout: post
title: "PortSwigger SQLi — Retrieving Hidden Data with OR 1=1"
date: 2026-06-30 19:00:00 -0700
categories: [web, portswigger]
tags: [sql-injection, web, owasp, portswigger]
---

Starting the **web application** track — where the real bug-bounty money lives. This is
the first lab in PortSwigger's [Web Security Academy](https://portswigger.net/web-security)
SQL injection path: **SQL injection in a WHERE clause, allowing retrieval of hidden data.**
Different from my HTB network boxes, same instinct: find where user input reaches a
trusted context, then abuse it.

> **Class:** SQL injection · **Impact:** unauthorized data disclosure · **Environment:**
> deliberately-vulnerable training lab (legal, authorized).

![SQL injection with OR 1=1 — how the payload rewrites the query](/assets/img/sqli-hidden-data.svg)

## 1. The target

A shopping site with a category filter. Clicking **Gifts** requests
`/filter?category=Gifts`, and the server builds this SQL:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

That `AND released = 1` is a business rule — only show products that are live. The value
`Gifts` comes straight from my URL, unsanitised. That's the door.

## 2. The injection

I changed the `category` parameter to:

```
Gifts' OR 1=1--
```

Which makes the server assemble:

```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1-- ' AND released = 1
```

Three moves in one payload:

| Piece | Effect |
|-------|--------|
| `Gifts'` | closes the string literal early |
| `OR 1=1` | **`1=1` is always true**, so `WHERE (…) OR true` matches **every row** |
| `--` | starts a SQL comment, discarding the rest (`AND released = 1`) |

The `released` filter is gone and the condition is always true, so the database returns
**every product on the site** — including the unreleased ones it was supposed to hide.

> Note on methodology: my first payload, `Gifts'--`, injected correctly but only returned
> the Gifts category, which had nothing hidden — so the lab didn't register a solve.
> `OR 1=1` is the canonical *"return everything"* move and proved impact immediately. A bug
> *executing* and a bug *demonstrating impact* are two different milestones.

## 3. Why this matters

`OR 1=1--` is a toy here, but the exact same flaw on a real application means dumping
user tables, password hashes, or payment records. SQL injection is a perennial
**critical-severity** finding and one of the highest-paying bug classes on bounty
platforms.

## 4. Remediation

| Root cause | Fix |
|------------|-----|
| User input concatenated into SQL | Use **parameterised queries / prepared statements** |
| Trusting the `category` value | Validate/allowlist expected inputs |
| Verbose behaviour on injection | Least-privilege DB accounts; don't expose raw query behaviour |

The one-line takeaway: **never build SQL by string concatenation.** Bind parameters and
the payload above becomes just a category name that doesn't exist.

*Performed against a legal, authorized PortSwigger Web Security Academy lab.*
