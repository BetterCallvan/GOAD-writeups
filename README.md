<div align="center">

# 🛡️ GOAD-writeups

**Active Directory Penetration Testing Writeups**

*Self-hosted labs (GOAD) · Enumeration · Privilege Escalation · Exploitation*

[![English](https://img.shields.io/badge/lang-English-blue)](#-english)
[![Українська](https://img.shields.io/badge/мова-Українська-yellow)](#-українська)
[![Русский](https://img.shields.io/badge/язык-Русский-red)](#-русский)

![GOAD](https://img.shields.io/badge/Lab-GOAD-black?logo=windows)
![NetExec](https://img.shields.io/badge/Tool-NetExec-informational)
![Impacket](https://img.shields.io/badge/Tool-Impacket-informational)
![Status](https://img.shields.io/badge/status-active-success)

</div>

---

## 🇬🇧 English

Writeups from Active Directory penetration testing labs — built on [GOAD](https://github.com/Orange-Cyberdefense/GOAD) (Game Of Active Directory), a self-hosted, intentionally vulnerable AD environment used for offensive security practice.

Each writeup documents a full attack chain: reconnaissance → enumeration → initial access → privilege escalation → full domain compromise, with real commands, outputs, and root-cause analysis.

### 📂 Labs

| Lab | Description | Status |
|---|---|---|
| [MINILAB](./MINILAB) | Single-domain AD lab — anonymous SMB enum, credential disclosure, lateral movement, domain compromise | ✅ Complete |

### 🧰 Tools & Techniques

`nmap` · `NetExec (nxc)` · `Impacket` · `smbclient` · `BloodHound` · `rpcclient` · Kerberoasting · AS-REP Roasting · Pass-the-Hash · LSA/SAM dumping · Credential reuse

### ⚠️ Disclaimer

All testing documented here was performed exclusively against **isolated, self-hosted, intentionally vulnerable lab environments**. None of this content targets real systems or production infrastructure.

---

## 🇺🇦 Українська

Writeup'и з лабораторій пентесту Active Directory — побудовані на [GOAD](https://github.com/Orange-Cyberdefense/GOAD) (Game Of Active Directory), власноруч розгорнутому, навмисно вразливому AD-середовищі для практики offensive security.

Кожен writeup документує повний ланцюжок атаки: розвідка → енумерація → первинний доступ → ескалація прав → повна компрометація домену, з реальними командами, виводом і аналізом корінних причин.

### 📂 Лабораторії

| Лаб | Опис | Статус |
|---|---|---|
| [MINILAB](./MINILAB) | Однодоменна AD-лабораторія — анонімна SMB-енумерація, розкриття кредів, lateral movement, компрометація домену | ✅ Завершено |

### 🧰 Інструменти та техніки

`nmap` · `NetExec (nxc)` · `Impacket` · `smbclient` · `BloodHound` · `rpcclient` · Kerberoasting · AS-REP Roasting · Pass-the-Hash · дамп LSA/SAM · повторне використання кредів

### ⚠️ Застереження

Усе тестування, задокументоване тут, виконувалось виключно проти **ізольованих, власноруч розгорнутих, навмисно вразливих лабораторних середовищ**. Жоден з цих матеріалів не стосується реальних систем чи продакшн-інфраструктури.

---

##  русский

Writeup'ы из лабораторий пентеста Active Directory — построенные на [GOAD](https://github.com/Orange-Cyberdefense/GOAD) (Game Of Active Directory), самостоятельно развёрнутой, намеренно уязвимой AD-среде для практики offensive security.

Каждый writeup документирует полную цепочку атаки: разведка → энумерация → первоначальный доступ → эскалация привилегий → полная компрометация домена, с реальными командами, выводом и анализом первопричин.

### 📂 Лаборатории

| Лаб | Описание | Статус |
|---|---|---|
| [MINILAB](./MINILAB) | Однодоменная AD-лаборатория — анонимная SMB-энумерация, раскрытие кредов, lateral movement, компрометация домена | ✅ Завершено |

### 🧰 Инструменты и техники

`nmap` · `NetExec (nxc)` · `Impacket` · `smbclient` · `BloodHound` · `rpcclient` · Kerberoasting · AS-REP Roasting · Pass-the-Hash · дамп LSA/SAM · повторное использование кредов

### ⚠️ Дисклеймер

Всё тестирование, задокументированное здесь, выполнялось исключительно против **изолированных, самостоятельно развёрнутых, намеренно уязвимых лабораторных сред**. Ни один из этих материалов не касается реальных систем или продакшн-инфраструктуры.

---

<div align="center">

**🔗 [GOAD Project](https://github.com/Orange-Cyberdefense/GOAD)** by Orange Cyberdefense

</div>
