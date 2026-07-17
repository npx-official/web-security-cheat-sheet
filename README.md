![GitHub repo size](https://img.shields.io/github/repo-size/npx-official/web-security-cheat-sheet)
![GitHub stars](https://img.shields.io/github/stars/npx-official/web-security-cheat-sheet)
![GitHub last commit](https://img.shields.io/github/last-commit/npx-official/web-security-cheat-sheet)

# 🛡️ Web Security Cheat Sheet

```
 ░▒▓██████▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓████████▓▒░░▒▓██████▓▒░▒▓████████▓▒░       ░▒▓███████▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓████████▓▒░▒▓████████▓▒░▒▓████████▓▒░ 
░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░ ░▒▓█▓▒░          ░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░      ░▒▓█▓▒░         ░▒▓█▓▒░     
░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░ ░▒▓█▓▒░          ░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░      ░▒▓█▓▒░         ░▒▓█▓▒░     
░▒▓█▓▒░      ░▒▓████████▓▒░▒▓██████▓▒░ ░▒▓████████▓▒░ ░▒▓█▓▒░           ░▒▓██████▓▒░░▒▓████████▓▒░▒▓██████▓▒░ ░▒▓██████▓▒░    ░▒▓█▓▒░     
░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░ ░▒▓█▓▒░                 ░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░      ░▒▓█▓▒░         ░▒▓█▓▒░     
░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░ ░▒▓█▓▒░                 ░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░      ░▒▓█▓▒░         ░▒▓█▓▒░     
 ░▒▓██████▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓████████▓▒░▒▓█▓▒░░▒▓█▓▒░ ░▒▓█▓▒░          ░▒▓███████▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓████████▓▒░▒▓████████▓▒░  ░▒▓█▓▒░     
                                                                                                                                          
```                                                                                                                                      

> A curated collection of web security vulnerabilities and cheat sheets for Bug Bounty & Penetration Testing

---

## 📂 Vulnerability Categories

### 🚀 Core Vulnerabilities (الأساسيات)

| Vulnerability | Difficulty | Link |
|:--------------|:-----------|:-----|
| **IDOR** | Easy | [IDOR.md](./cheat-sheet/IDOR.md) |
| **XSS (Cross-Site Scripting)** | Easy | [XSS.md](./cheat-sheet/XSS.md) |
| **SQL Injection** | Medium | [sql-injection.md](./cheat-sheet/sql-injection.md) |
| **Authentication Bypass** | Easy | [Auth-Bypass.md](./cheat-sheet/Auth-Bypass.md) |
| **File Upload** | Medium | [File-Upload.md](./cheat-sheet/File-Upload.md) |

---

### 🎯 Common & Critical (الشائعة والحرجة)

| Vulnerability | Difficulty | Link |
|:--------------|:-----------|:-----|
| **SSRF (Server-Side Request Forgery)** | Medium | [SSRF.md](./cheat-sheet/SSRF.md) |
| **OS Command Injection** | Medium | [OS-CMD-Injection.md](./cheat-sheet/OS-CMD-Injection.md) |
| **NoSQL Injection** | Medium | [NoSQLi.md](./cheat-sheet/NoSQLi.md) |
| **JWT Vulnerabilities** | Medium | [JWT.md](./cheat-sheet/JWT.md) |
| **Business Logic Flaws** | Hard | [Business-Logic.md](./cheat-sheet/Business-Logic.md) |

---

### 🧩 Advanced & API (المتقدمة و API)

| Vulnerability | Difficulty | Link |
|:--------------|:-----------|:-----|
| **GraphQL Injection** | Hard | [GraphQL.md](./cheat-sheet/GraphQL.md) |
| **Prototype Pollution** | Hard | [Prototype-Pollution.md](./cheat-sheet/Prototype-Pollution.md) |
| **HTTP Request Smuggling** | Hard | [Request-Smuggling.md](./cheat-sheet/Request-Smuggling.md) |
| **Web Cache Poisoning** | Hard | [Cache-Poisoning.md](./cheat-sheet/Cache-Poisoning.md) |
| **CORS Misconfiguration** | Medium | [CORS.md](./cheat-sheet/CORS.md) |

---

### 🛠️ Additional (إضافية)

| Vulnerability | Difficulty | Link |
|:--------------|:-----------|:-----|
| **Clickjacking** | Easy | [Clickjacking.md](./cheat-sheet/Clickjacking.md) |
| **XXE (XML External Entity)** | Medium | [XXE.md](./cheat-sheet/XXE.md) |
| **CSRF (Cross-Site Request Forgery)** | Easy | [CSRF.md](./cheat-sheet/CSRF.md) |
| **WebSockets** | Medium | [WebSockets.md](./cheat-sheet/WebSockets.md) |
| **Host Header Attacks** | Medium | [Host-Header.md](./cheat-sheet/Host-Header.md) |

---

## 🗂️ Folder Structure

web-security-cheat-sheet/
├── README.md
└── cheat-sheet/
├── Auth-Bypass.md
├── Business-Logic.md
├── Cache-Poisoning.md
├── Clickjacking.md
├── CORS.md
├── CSRF.md
├── File-Upload.md
├── GraphQL.md
├── Host-Header.md
├── IDOR.md
├── JWT.md
├── NoSQLi.md
├── OS-CMD-Injection.md
├── Prototype-Pollution.md
├── README.md
├── Request-Smuggling.md
├── sql-injection.md
├── SSRF.md
├── WebSockets.md
├── XSS.md
└── XXE.md



---

## 🛠️ Tools & Resources

| Tool | Description |
|:-----|:------------|
| **Burp Suite** | الـ Proxy الأساسي لاختبار الويب |
| **FFUF** | لتخمين الملفات والمجلدات |
| **Param Miner** | لاكتشاف البارامترات المخفية |
| **PortSwigger Labs** | تدريب عملي مجاني |
| **OWASP Top 10** | قائمة أهم الثغرات |

---

## 📊 Progress Tracker

| Category | Completed | Total |
|:---------|:----------|:------|
| Core Vulnerabilities | 5 | 5 |
| Common & Critical | 5 | 5 |
| Advanced & API | 5 | 5 |
| Additional | 5 | 5 |
| **Total** | **20** | **20** |

---

⭐ **Happy Hacking!**
