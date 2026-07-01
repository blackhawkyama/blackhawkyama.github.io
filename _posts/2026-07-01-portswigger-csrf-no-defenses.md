---
layout: post
title: "PortSwigger CSRF — Forging a Request in the Victim's Browser"
date: 2026-07-01 07:30:00 -0700
categories: [web, portswigger]
tags: [csrf, cross-site-request-forgery, web, owasp, portswigger]
---

Fifth web bug class, and a different mindset from the rest. SQLi/SSRF attack the *server*;
XSS runs code in a browser; **CSRF (Cross-Site Request Forgery)** makes a **victim's own
browser** send a request they never intended — as *them*. PortSwigger lab: *CSRF
vulnerability with no defenses.*

> **Class:** CSRF · **Impact:** perform state-changing actions as a logged-in victim
> (here: change their email → account takeover) · **Environment:** deliberately-vulnerable
> training lab (legal, authorized).

![CSRF — auto-submitting a forged form using the victim's cookie](/assets/img/csrf-basic.svg)

## 1. The vulnerable action

Logged in as my own account (`wiener:peter`), the email-change form sends:

```
POST /my-account/change-email
email=whatever@example.com
```

Inspecting the form, it has **one field (`email`) and no CSRF token** — no unpredictable,
per-request value. The server authorises the change based *only* on the session cookie.

## 2. Why that's exploitable

Browsers **automatically attach a site's cookies to every request to that site** — even
requests triggered by a *different, malicious* page. So if I can get a logged-in victim's
browser to POST to `/my-account/change-email`, their session cookie rides along and the
server accepts it. Without a token the server can't distinguish my forged request from a
genuine one.

## 3. The exploit

I host this on the exploit server. It auto-submits the moment the page loads:

```html
<form action="https://LAB/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="pwned@evil-attacker.net">
</form>
<script>document.forms[0].submit();</script>
```

Then **"Deliver exploit to victim."** The victim's browser loads the page, silently
submits the form with *their* cookie, and their email is changed to mine. They clicked
nothing. Lab solved.

## 4. Why it matters

Change-email is the classic pivot: change the email, request a password reset, own the
account. CSRF has historically been used for fund transfers, privilege changes, and
settings tampering — any **state-changing** request that relies on the cookie alone.

## 5. Remediation

| Root cause | Fix |
|------------|-----|
| No per-request token on a state-changing action | **CSRF tokens** — unpredictable, tied to the session, validated server-side |
| Cookies sent on cross-site requests | **`SameSite=Lax/Strict`** cookies (browser-level defense) |
| Sensitive action via simple form POST | Re-authenticate / confirm for high-value changes |

Same family rule as the others, seen from the client side: **a state change must prove it
was intended** — a token the attacker can't guess is that proof. (This is why modern
frameworks + `SameSite` defaults have made basic CSRF far rarer.)

*Performed against a legal, authorized PortSwigger Web Security Academy lab.*
