# 🛡️ Web Security Cheat Sheet

```bash
░▒▓█▓▒░░▒▓█▓▒░▒▓████████▓▒░▒▓███████▓▒░       ░▒▓█▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓███████▓▒░░▒▓█▓▒░▒▓████████▓▒░▒▓████████▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓███████▓▒░ ░▒▓███████▓▒░      
░▒▓█▓▒░░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓█▓▒░░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░             
░▒▓█▓▒░░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓█▓▒░░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░             
░▒▓████████▓▒░  ░▒▓█▓▒░   ░▒▓███████▓▒░       ░▒▓█▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓███████▓▒░░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓██████▓▒░ ░▒▓█▓▒░░▒▓█▓▒░▒▓███████▓▒░ ░▒▓██████▓▒░       
░▒▓█▓▒░░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓█▓▒░░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░             ░▒▓█▓▒░      
░▒▓█▓▒░░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓█▓▒░░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓█▓▒░      ░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░             ░▒▓█▓▒░      
░▒▓█▓▒░░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓███████▓▒░        ░▒▓█████████████▓▒░░▒▓█▓▒░░▒▓█▓▒░▒▓█▓▒░  ░▒▓█▓▒░   ░▒▓████████▓▒░░▒▓██████▓▒░░▒▓█▓▒░      ░▒▓███████▓▒░       

    A curated collection of web security vulnerabilities and cheat sheets for Bug Bounty & Penetration Testing

📂 Vulnerability Categories
🚀 Core Vulnerabilities (الأساسيات)
Vulnerability	Difficulty	Link
IDOR	Easy	IDOR.md
XSS (Cross-Site Scripting)	Easy	XSS.md
SQL Injection	Medium	sql-injection.md
Authentication Bypass	Easy	Auth-Bypass.md
File Upload	Medium	File-Upload.md
🎯 Common & Critical (الشائعة والحرجة)
Vulnerability	Difficulty	Link
SSRF (Server-Side Request Forgery)	Medium	SSRF.md
OS Command Injection	Medium	OS-CMD-Injection.md
NoSQL Injection	Medium	NoSQLi.md
JWT Vulnerabilities	Medium	JWT.md
Business Logic Flaws	Hard	Business-Logic.md
🧩 Advanced & API (المتقدمة و API)
Vulnerability	Difficulty	Link
GraphQL Injection	Hard	GraphQL.md
Prototype Pollution	Hard	Prototype-Pollution.md
HTTP Request Smuggling	Hard	Request-Smuggling.md
Web Cache Poisoning	Hard	Cache-Poisoning.md
CORS Misconfiguration	Medium	CORS.md
🛠️ Additional (إضافية)
Vulnerability	Difficulty	Link
Clickjacking	Easy	Clickjacking.md
XXE (XML External Entity)	Medium	XXE.md
CSRF (Cross-Site Request Forgery)	Easy	CSRF.md
WebSockets	Medium	WebSockets.md
Host Header Attacks	Medium	Host-Header.md
🗂️ Folder Structure
text

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

🛠️ Tools & Resources

    Burp Suite - الـ Proxy الأساسي لاختبار الويب

    FFUF - لتخمين الملفات والمجلدات

    Param Miner - لاكتشاف البارامترات المخفية

    PortSwigger Labs - تدريب عملي مجاني

    OWASP Top 10 - قائمة أهم الثغرات

📊 Progress Tracker
Category	Completed	Total
Core Vulnerabilities	5	5
Common & Critical	5	5
Advanced & API	5	5
Additional	5	5
Total	20	20

⭐ Happy Hacking!
