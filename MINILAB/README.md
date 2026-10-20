# GOAD MINILAB — Active Directory Penetration Test Writeup

**Lab:** GOAD MINILAB · **Domain:** `mini.lab` · **Result:** ✅ Full domain compromise

A complete attack chain against a GOAD (Game Of Active Directory) MINILAB environment — from anonymous SMB enumeration to full Domain Controller compromise via credential reuse.

## Read the full writeup

| Language | Link |
|---|---|
| 🇬🇧 English | [WRITEUP.en.md](./WRITEUP.en.md) |
| 🇺🇦 Українська | [WRITEUP.ua.md](./WRITEUP.ua.md) |
|  русский | [WRITEUP.ru.md](./WRITEUP.ru.md) |

## Quick Summary

```
Anonymous SMB enumeration
        │
        ▼
Password disclosure via SMB description field (bob:superman)
        │
        ▼
Local admin on WS01
        │
        ▼
SAM / LSA secrets dump → vagrant:vagrant (plaintext)
        │
        ▼
Credential reuse on DC01
        │
        ▼
vagrant found in local Administrators group on DC01
        │
        ▼
Full Domain Controller compromise
```

**Tools used:** nmap, NetExec (nxc), Impacket, smbclient, Samba `net rpc`

**Lab environment:** [GOAD](https://github.com/Orange-Cyberdefense/GOAD) by Orange Cyberdefense
