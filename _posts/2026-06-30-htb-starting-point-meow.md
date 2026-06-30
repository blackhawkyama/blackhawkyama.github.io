---
layout: post
title: "HTB Starting Point — Meow: Telnet & Missing Authentication"
date: 2026-06-30 18:00:00 -0700
categories: [htb, starting-point]
tags: [telnet, enumeration, misconfiguration, linux]
---

First box of the Hack The Box **Starting Point** track. Meow is a Tier 0 machine —
no exploit code required — but it's a clean lesson in why **enumeration first** and
**legacy services are dangerous**. Full compromise came from a single misconfigured
service exposing a passwordless `root` login.

> **Difficulty:** Very Easy · **OS:** Linux (Ubuntu 20.04.2 LTS) · **Key idea:**
> exposed Telnet with missing authentication.

## 1. Recon

Start by confirming the host is alive, then map its attack surface.

```bash
ping -c 3 10.129.x.x
```

The host replied with **TTL 63** — one hop off the default Linux TTL of 64 — an early
hint we're dealing with a Linux target.

Next, a service/version scan with `nmap`:

```bash
nmap -sC -sV -T4 -oN nmap_initial.txt 10.129.x.x
```

```
PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- `-sC` runs nmap's default safe scripts
- `-sV` fingerprints service versions
- `-oN` saves the output (you want this for your notes/report)

**Finding:** exactly one open port — **23/tcp, Telnet.** That's the entire attack
surface.

## 2. Analysis

Telnet is a legacy remote-login protocol from before encryption was standard. Two
problems make it a serious finding on its own:

1. **No encryption** — credentials and session data travel in plaintext, trivially
   sniffable on the network. It was superseded by SSH ~25 years ago.
2. **Exposed to the network** — a Telnet service reachable like this is almost always a
   misconfiguration. The classic failure is an account that logs in with **no
   password**.

So the plan is simple: connect and try common privileged usernames.

## 3. Foothold

```bash
telnet 10.129.x.x
```

At the `Meow login:` prompt, the username **`root`** was accepted with **no password**
— dropping straight into a root shell:

```
Meow login: root
...
root@Meow:~# id
uid=0(root) gid=0(root) groups=0(root)
```

`uid=0` confirms full root access. No privilege escalation needed — the foothold *is*
root.

## 4. Loot

```bash
root@Meow:~# cat /root/flag.txt
[REDACTED — 32-char hash]
```

Flag submitted, machine owned. ✅

## 5. Why it worked & how to fix it

| Root cause | Remediation |
|---|---|
| Telnet exposed to the network | Disable Telnet; use **SSH** with key-based auth |
| `root` login with no password | Enforce strong passwords; **never** allow passwordless or direct root login |
| Plaintext protocol | Use encrypted transport for all remote administration |

**Takeaways for the methodology:** always enumerate *before* reaching for exploits — a
single port told the whole story here. And legacy services (Telnet, FTP, rsh, SMBv1)
are worth checking first; they're frequently left misconfigured.

*Performed in a legal, authorized environment (HTB Starting Point).*
