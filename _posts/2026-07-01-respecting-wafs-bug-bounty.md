---
layout: post
title: "Respecting WAFs — Vet the Target Before You Burn the Day"
date: 2026-07-01 09:30:00 -0700
categories: [methodology, recon]
tags: [waf, bot-detection, datadome, recon, target-selection, bug-bounty]
---

Here's a lesson I paid for in time rather than reading about: **not every in-scope target is a
good target to *start* on.** Some sit behind aggressive bot-detection that fights the exact tools
you need — and if you don't check for that first, you can spend hours banging on a wall you could
have spotted in thirty seconds. This is the pre-check I now run before I invest in *anything*.

> **Scope note:** generic methodology, no specific target named. Recon and testing only ever
> against assets you're authorized to touch.

![Vet for a WAF before you invest a day](/assets/img/waf-vetting.svg)

## What a bot-WAF actually does to you

A Web Application Firewall with bot-detection (DataDome, Cloudflare Bot Management, Akamai,
PerimeterX/HUMAN, Imperva, and friends) isn't just "a firewall." It fingerprints *how* traffic
arrives and blocks anything that smells automated. In practice it breaks the things a hunter
leans on:

- **Automated tooling** — `httpx`, `ffuf`, `nuclei`, probing scanners get served `403`s and
  challenge pages instead of real responses.
- **Injected JavaScript** — hooking `fetch`, poking around in the console: some WAFs literally
  flag *"use of developer or inspection tools."*
- **Intercepting proxies** — and this is the painful one. The proxy is the *right* tool for
  API testing, but a good WAF can fingerprint proxied traffic and re-challenge you **even in a
  real browser** routed through it. The tool you need becomes the thing that gets you blocked.
- **Your whole IP** — the restriction usually lands on your public IP address, not one browser.
  So every browser on your network is walled until it expires (which can be a while). Switching
  browser doesn't help; a new IP or patience does.

None of this means the target is unhackable. It means beating the WAF is its own cat-and-mouse
project — residential proxies, TLS-fingerprint work, careful pacing, reusing a valid challenge
cookie. That's a real skill set, but it is *not* where a first bug comes from.

## The 30-second pre-check

So before committing to a program, I look at the front door and ask "is there a bouncer?" Two
quick commands on the main asset:

```bash
curl -sI https://target | grep -iE 'server|datadome|cf-|akamai|x-px|set-cookie'
httpx -u https://target -status-code -title -tech-detect -server
```

What I'm reading:

- **Clean `200`, ordinary headers, no bot cookies** → good candidate. My tools will work.
- **`403` / a challenge page, or cookies like `datadome`, `__cf_bm`, `_px`, Akamai markers** →
  heavy bot-WAF. As someone still early in this, that's my cue to move on.

The tell is also behavioural: if the site loads fine when you click by hand but breaks the moment
anything automated touches it, you've found the bouncer.

## The discipline (the real point)

The mistake isn't hitting a WAF — you can't always know in advance, and running into one taught
me exactly what these controls look like from the attacker's side. The mistake is *not checking,
then sinking a day into a fortress.* So the habit now:

1. Prefer **wide-scope public VDPs** — government, education, open-source. They tend to be
   WAF-light and patient with newcomers.
2. **Run the WAF pre-check before investing real time.**
3. If it's DataDome/Akamai-hardened and I'm early-stage, **choose a different target.** The
   methodology travels; the specific site doesn't matter.

There's no shame in walking away from a target — there's only wasted time in refusing to. Vet
first, then dig.
