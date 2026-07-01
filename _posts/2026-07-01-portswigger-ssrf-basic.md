---
layout: post
title: "PortSwigger SSRF — Making the Server Attack Itself"
date: 2026-07-01 07:00:00 -0700
categories: [web, portswigger]
tags: [ssrf, server-side-request-forgery, web, owasp, portswigger]
---

Fourth web bug class. After [SQLi](https://blackhawkyama.github.io/posts/portswigger-sqli-retrieve-hidden-data/),
[XSS](https://blackhawkyama.github.io/posts/portswigger-xss-reflected/), and
[IDOR](https://blackhawkyama.github.io/posts/portswigger-idor-access-control/), this one
turns the server into your proxy: **Server-Side Request Forgery (SSRF)**. PortSwigger
lab: *Basic SSRF against the local server.*

> **Class:** SSRF · **Impact:** reach internal-only services from outside → in this case,
> the admin panel → account deletion · **Environment:** deliberately-vulnerable training
> lab (legal, authorized).

![SSRF — hijacking a server-side fetch to reach localhost](/assets/img/ssrf-basic.svg)

## 1. The target

A shop where each product page has a **"Check stock"** button. Clicking it makes the
**server** fetch a stock-status URL — and that URL is supplied **in my own request**, as
a `stockApi` parameter:

```
POST /product/stock
stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=1&storeId=1
```

The server blindly fetches whatever URL I put there. That's the whole bug.

## 2. Reach the internal admin panel

I can't browse to the server's `localhost` — it's firewalled off from the internet. But
the **server** can reach it. So I swap the parameter:

```
stockApi=http://localhost/admin
```

The response comes back with the internal admin interface — including its user-delete
links:

```
/admin/delete?username=wiener
/admin/delete?username=carlos
```

I borrowed the server's position on the network to read something I was never meant to
see.

## 3. Weaponize it

The lab wants `carlos` gone. Point the SSRF at the delete endpoint:

```
stockApi=http://localhost/admin/delete?username=carlos
```

The server carries out the request against its own admin panel. `carlos` is deleted —
lab solved.

*(Note: the delete endpoint returned a `401` status, but the action still executed
server-side — a good reminder to verify impact, not just trust the response code.)*

## 4. Why SSRF is a top-tier finding

`localhost/admin` is the lab version. In the cloud, the killer target is the **instance
metadata service**:

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

An SSRF that reaches that hands you the server's **cloud credentials** — which is exactly
how the **Capital One breach** (100M+ records) happened. SSRF is consistently rated
high/critical and pays accordingly on bounty programs.

## 5. Remediation

| Root cause | Fix |
|------------|-----|
| Server fetches a user-supplied URL | **Allowlist** exact destinations; reject everything else |
| Internal services reachable from the app server | Network-segment them; require auth on internal endpoints |
| Raw metadata endpoint exposed | Enforce IMDSv2 / block `169.254.169.254` egress |

Core rule, same family as the others: **never let untrusted input choose what a trusted
component does** — for SQL it's the query, for HTML it's the markup, for SSRF it's the
*destination of a request.*

*Performed against a legal, authorized PortSwigger Web Security Academy lab.*
