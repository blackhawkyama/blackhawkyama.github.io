---
layout: post
title: "PortSwigger IDOR — Reading Another User's Data by Changing an ID"
date: 2026-06-30 20:30:00 -0700
categories: [web, portswigger]
tags: [idor, access-control, broken-access-control, web, owasp, portswigger]
---

Fourth web bug, third class — and the one you'll reach for most in the real world.
No injection here at all. **Insecure Direct Object Reference (IDOR)**, a flavour of
**broken access control** — OWASP's current #1 web risk. PortSwigger lab:
*Insecure direct object references.*

> **Class:** Broken access control (IDOR) · **Impact:** read another user's private
> data → account takeover · **Environment:** deliberately-vulnerable training lab (legal,
> authorized).

![IDOR — changing an object ID to access another user's data](/assets/img/idor-access-control.svg)

## 1. The target

A shop with a **Live chat** feature. Chat transcripts are stored as files on the server
and fetched with a static URL. My own transcript downloads from:

```
GET /download-transcript/2.txt
```

That `2` is a **direct object reference** — an ID pointing straight at a stored object.
The question a hunter always asks: *what if I change it?*

## 2. The exploit

Request a different ID:

```
GET /download-transcript/1.txt
```

The server hands it over — it **never checks whether that transcript belongs to me**.
That's the entire bug: a missing *"is this yours?"* authorization check.

Transcript `1.txt` turned out to be another user (`carlos`) — who had helpfully typed his
password in plaintext into the chat:

```
You: my password is [redacted]. Is that right?
Hal Pline: Yes it is!
```

## 3. Account takeover

Take that password to the login form:

```
Username: carlos
Password: [the leaked password]
```

Logged in as `carlos`. **"Your username is: carlos."** Full account takeover — from
nothing but decrementing a number in a URL.

## 4. Why this matters

IDOR is everywhere because almost every app fetches objects by ID — invoices, messages,
orders, user profiles, files. Any endpoint like `/api/user/1234` or `?order_id=5501`
that returns data **without verifying the requester owns it** is an IDOR. It's the
single most common bug class on real bounty programs, and it needs **zero** special
payload — just curiosity and a value you can change.

## 5. Remediation

| Root cause | Fix |
|------------|-----|
| No ownership check on the object | Enforce **access control server-side on every request** — verify the session owns the resource |
| Guessable/sequential IDs | Use unpredictable identifiers (UUIDs) as defense-in-depth (not a fix on its own) |
| Sensitive data over static files | Don't store secrets in retrievable transcripts; scope file access |

The lesson: **authentication ≠ authorization.** Being *logged in* doesn't mean you're
allowed to see object #1. Check ownership every time.

*Performed against a legal, authorized PortSwigger Web Security Academy lab.*
