# Web Security Headers — securityheaders.com + Apache

Last part of Task 3. This one's less about exploiting a specific input field and more about checking whether the *server itself* is telling the browser to behave securely — things like whether it can be embedded in an iframe from another site, whether it forces HTTPS, and what it allows scripts to do. Tested against a local Apache + DVWA setup on Kali.

## What security headers actually do

HTTP response headers aren't just metadata — several of them are direct instructions to the browser about how to treat the page. A server can tell the browser things like "don't let this page be framed by another site," "don't guess content types," or "only ever load resources from this domain." Without these headers, the browser falls back to permissive defaults, which opens the door to things like clickjacking, MIME-sniffing attacks, and XSS being easier to pull off even if the application code itself is otherwise fine.

This is why header configuration matters even on an app that's already been hardened against SQLi/XSS/CSRF individually — headers are a site-wide safety net, not a fix for one specific bug.

---

## 1. Baseline scan with securityheaders.com

[securityheaders.com](https://securityheaders.com) is a free scanner that requests a URL, inspects the response headers, and grades the site from F to A+ based on which protective headers are present and how strictly they're configured.

Since securityheaders.com needs to reach the target over the public internet, scanning `localhost` directly won't work — a couple of options:

- Temporarily expose the local DVWA instance with a tunnel tool like `ngrok`:
  ```bash
  ngrok http 80
  ```
  then scan the `https://xxxxx.ngrok.io` URL it gives you.
- Or run the scan against any test site you actually control and can edit the Apache config for.

**Before adding any headers, a typical default Apache + DVWA response looks like this (checked manually first with curl):**
```bash
curl -I http://localhost/dvwa/login.php
```
```
HTTP/1.1 200 OK
Date: Mon, 21 Sep 2026 10:15:02 GMT
Server: Apache/2.4.58 (Ubuntu)
X-Powered-By: PHP/8.1.2
Content-Type: text/html; charset=UTF-8
```

Nothing there about framing restrictions, content type sniffing, HTTPS enforcement, or script sources — which is exactly what securityheaders.com flags. A default Apache install typically grades out around an **F** or **D**, with most or all of the following reported missing:

- Content-Security-Policy
- X-Frame-Options
- X-Content-Type-Options
- Strict-Transport-Security
- Referrer-Policy
- Permissions-Policy

---

## 2. Adding the headers in Apache

### Enable the headers module first
Apache's `Header` directive comes from `mod_headers`, which isn't always enabled by default:
```bash
sudo a2enmod headers
```

### Add the headers to the config
Edit either the main config or a site-specific conf file (cleaner to use the site conf, e.g. `/etc/apache2/sites-available/000-default.conf`, inside the `<VirtualHost>` block):

```bash
sudo nano /etc/apache2/apache2.conf
```

Add:
```apache
Header always set X-Frame-Options "DENY"
Header always set X-Content-Type-Options "nosniff"
Header always set Content-Security-Policy "default-src 'self'"
Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains"
Header always set Referrer-Policy "no-referrer-when-downgrade"
```

### What each one actually does

**`X-Frame-Options: DENY`**
Stops the page from being loaded inside an `<iframe>` on any site at all, including the site's own other pages. This is the direct fix for clickjacking — an attack where a malicious site layers an invisible iframe of your app over a fake button, tricking users into clicking something on the real app without realizing it. `DENY` is the strictest option; `SAMEORIGIN` is the middle ground if the app legitimately needs to frame its own pages.

**`X-Content-Type-Options: nosniff`**
Tells the browser not to try to guess ("sniff") a file's content type based on its content, and to strictly respect the `Content-Type` header the server sends instead. Without this, a browser might decide a file that's labeled as an image is actually HTML/JavaScript based on its content and execute it — which has historically been used to smuggle XSS payloads through file upload features.

**`Content-Security-Policy: default-src 'self'`**
The big one — restricts which sources the browser is allowed to load any kind of resource from (scripts, styles, images, fonts, etc.) to the site's own origin only. This is the same header covered in the XSS notes: even if an attacker manages to inject a `<script>` tag somehow, a CSP like this blocks the browser from loading/executing anything that isn't from the app's own domain, giving a second layer of defense against XSS beyond just output encoding.

**`Strict-Transport-Security: max-age=63072000; includeSubDomains`**
Known as HSTS. Tells the browser "always use HTTPS for this domain from now on, for the next `max-age` seconds (here, 2 years), and apply that to subdomains too." Once a browser has seen this header once, it will automatically upgrade any future `http://` request to `https://` *before* even sending it — which protects against downgrade attacks where someone on the network tries to force a victim onto an unencrypted connection. Only meaningful on a site actually served over HTTPS.

**`Referrer-Policy: no-referrer-when-downgrade`**
Controls how much information gets sent in the `Referer` header when a user clicks a link away from the site. This setting sends the full URL to other HTTPS sites but withholds it entirely when going from HTTPS to plain HTTP, preventing sensitive URL data (session tokens in query strings, internal page paths, etc.) from leaking to an insecure destination. A stricter option worth considering for sensitive apps is `strict-origin-when-cross-origin` or even `no-referrer`.

### Restart Apache to apply
```bash
sudo systemctl restart apache2
```

### Verify headers are actually being sent
```bash
curl -I http://localhost/dvwa/login.php
```
```
HTTP/1.1 200 OK
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Content-Security-Policy: default-src 'self'
Strict-Transport-Security: max-age=63072000; includeSubDomains
Referrer-Policy: no-referrer-when-downgrade
Content-Type: text/html; charset=UTF-8
```

---

## 3. Re-scanning and comparing the grade

Run the same securityheaders.com scan against the same URL again after the restart.

**Expected improvement:** going from the earlier F/D grade up to a B or A-range grade, depending on exactly which headers were added and how strict the CSP is. securityheaders.com will now show the five headers above as present, and may still flag a couple of optional/newer ones as missing if they weren't added:

- `Permissions-Policy` — restricts access to browser features like camera, microphone, geolocation per-origin. Not added above, but easy to include:
  ```apache
  Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
  ```
- A tighter `Content-Security-Policy` — `default-src 'self'` is a solid baseline, but a full A+ grade usually wants more explicit directives (`script-src`, `style-src`, `object-src 'none'`, etc.) rather than relying on `default-src` to cover everything.

The before/after here is the actual proof for this step: same URL, same application, but the grade moves up a full letter or more purely from response headers — no application code changed at all.

---

## 4. A caution on Content-Security-Policy specifically

`default-src 'self'` is a strong baseline but can break functionality if the app loads anything from a CDN, inline `<script>` blocks, or inline event handlers (`onclick="..."` attributes) — all of that gets blocked by default under a strict CSP. Worth testing the app thoroughly after adding this header rather than assuming it "just works," since a half-broken app with a great CSP grade isn't actually a win. DVWA in particular has some inline scripts in its UI that may need the CSP adjusted (e.g. adding `'unsafe-inline'` to `script-src`, or better, moving those scripts to external files) to avoid breaking the app's own interface.

---

## Key Takeaways

- Security headers are a site-wide layer of defense that works independently of whether individual pages have been hardened against specific bugs like XSS or CSRF.
- `X-Frame-Options` stops clickjacking, `X-Content-Type-Options` stops MIME-sniffing abuse, `CSP` restricts what the browser will execute/load, `HSTS` forces HTTPS, and `Referrer-Policy` limits data leakage through outbound links.
- These are config-only changes — no application code was touched, yet the security grade (and real protection) improved significantly.
- A strict CSP needs to be tested against the actual app afterward, since it can silently break inline scripts/styles if the app relies on them.
