# Burp Suite Advanced — Intercepting & Fuzzing DVWA Login

Fifth part of the web app security task. This one isn't about exploiting a specific bug in DVWA's code like SQLi/XSS/CSRF/file inclusion — it's about using Burp Suite as a proxy to see and manipulate traffic between the browser and the server, then using that same captured request to automate a login brute-force with Intruder.

## What Burp Suite actually does here

Burp sits as a man-in-the-middle proxy between the browser and the target application. Every request the browser sends passes through Burp first, where it can be inspected, paused, edited, and replayed before it ever reaches the server. This matters for two reasons:

1. It proves that **anything happening in the browser's UI is irrelevant to what the server actually receives** — if a request isn't properly validated server-side, intercepting and editing it client-side bypasses whatever the form/JavaScript was trying to enforce.
2. Once you've got one real, working request captured, you can hand it to Intruder and have Burp replay it hundreds or thousands of times with different payload values automatically — which is exactly how automated brute-forcing works in practice.

---

## 1. Setting up the proxy

**Burp side:**
- Open Burp Suite (pre-installed on Kali).
- Go to **Proxy → Options** and confirm the listener is running on `127.0.0.1:8080` (default, usually already active).

**Browser side (Firefox):**
- Settings → Network Settings → Manual proxy configuration
- HTTP Proxy: `127.0.0.1`, Port: `8080`
- Tick "Also use this proxy for HTTPS"

**Installing Burp's CA certificate** (only needed if the target is served over HTTPS):
- With the proxy active, visit `http://burpsuite` in the browser — this serves Burp's certificate download page.
- Download `cacert.der`.
- Firefox → Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import → select the file → check "Trust this CA to identify websites."
- Without this step, HTTPS sites throw certificate warnings because Burp is presenting its own cert in place of the real one in order to decrypt and inspect the traffic.

**Confirming it works:**
Browse to the DVWA login page, then check Burp's **Proxy → HTTP history** tab — the request should show up there, proving traffic is actually flowing through Burp.

---

## 2. Intercepting and modifying the login request

1. Go to **Proxy** tab, make sure **Intercept is on**.
2. In the browser, open DVWA's login page, type any credentials (e.g. `admin` / `wrongpassword123`), click Login.
3. The request freezes inside Burp instead of reaching the server. Raw request looks something like:
   ```
   POST /dvwa/login.php HTTP/1.1
   Host: localhost
   Content-Type: application/x-www-form-urlencoded
   Content-Length: 44

   username=admin&password=wrongpassword123&Login=Login
   ```
4. Edit the body directly inside Burp — change the password value to DVWA's actual default:
   ```
   username=admin&password=password&Login=Login
   ```
5. Click **Forward** to release the (now-modified) request to the server.
6. Check the response that comes back — login succeeds, even though the browser UI never showed the correct password being typed.

**What this actually demonstrates:** client-side input is just a suggestion. The server only ever sees whatever arrives in the raw request, and anyone able to intercept traffic between the client and server (a malicious proxy, a compromised network, etc.) can rewrite that request before it's processed. It's a good practical argument for why HTTPS, server-side validation, and things like account lockouts after failed attempts all matter — none of them can be bypassed just by editing values in a browser's dev tools, but a raw intercepted request can go further than that.

---

## 3. Automated brute-forcing with Intruder

Once a real login request has been captured, it becomes a template Intruder can replay automatically with different payload values substituted in.

### Sending the request to Intruder
With the captured login request visible in **Proxy → HTTP history**, right-click it → **Send to Intruder** (shortcut: `Ctrl+I`).

### Setting the payload position
Go to **Intruder → Positions**. Burp auto-highlights several fields it guesses might be interesting — clear all of that first (**Clear §** button), since only the password should be targeted here.

Manually select just the password's value and mark it:
```
username=admin&password=§wrongpassword123§&Login=Login
```
The `§...§` markers tell Intruder exactly where each wordlist entry gets substituted in — everything outside the markers stays fixed across every attempt.

**Attack type:** leave it on **Sniper** — it cycles a single payload set through one marked position, which is exactly the setup here (one field, one wordlist).

### Loading the wordlist
Go to **Payloads** tab:
- Payload type: **Simple list**
- Payload Options → **Load** → select a wordlist file, e.g. `/usr/share/wordlists/rockyou.txt`

Note: rockyou.txt has over 14 million entries — fine for a real assessment but painfully slow for a demo. For a video/screenshot walkthrough, a trimmed list of 50–100 common passwords (including DVWA's actual default, `password`) gets the point across in a reasonable time.

### Making success easy to spot
Go to **Options** tab → **Grep - Match** section, and add a string that only shows up on a successful login (something like DVWA's logged-in welcome text). This adds a tick/cross column to the results table so a successful attempt is visible immediately, rather than opening every response individually.

### Running the attack
Click **Start Attack** (top right). Community Edition runs somewhat throttled compared to Pro, but it's enough for a wordlist of reasonable size.

### Reading the results
In the results table, sort by:
- **Length** — a successful login usually returns a page of a different size than a failed-login page (dashboard content vs. an error message), so the one differently-sized row tends to stand out immediately.
- **Status** — not always reliable on its own, since DVWA may return `200 OK` for both success and failure; this is exactly why the Grep-Match column from the previous step is useful as a second signal.

The row with the standout response length (and/or a ticked Grep-Match column) is the correct password. Confirmed by logging into DVWA manually with that value afterward.

---

## 4. Why this matters / mitigation

This isn't a bug in DVWA's code the way SQLi or XSS are — it's a demonstration of what happens when a login form has **no protections against automated attempts**. The fixes that actually stop this kind of attack:

- **Account lockout** after a set number of failed attempts (e.g. 5 failures → 15 minute lock).
- **Rate limiting** on the login endpoint — reject or delay requests coming in faster than a real user could type.
- **CAPTCHA** after a few failed attempts, to block simple scripted replay.
- **Multi-factor authentication** — even a correctly guessed password isn't enough on its own to log in.
- Monitoring/alerting on a high volume of failed logins from a single source, which is usually the actual detection point in a real environment rather than prevention alone.

DVWA's "Brute Force" module (separate page, not covered here) and its "Insecure CAPTCHA" page specifically exist to demonstrate these mitigations if it's worth including a quick before/after comparison in the final report.

---

## Key Takeaways

- Burp works as a proxy, which means it sees traffic *before* it's encrypted/sent and *after* it's decrypted/received — installing its CA cert is what makes HTTPS inspection possible without constant browser warnings.
- Intercepting and editing a request proves that server-side validation is the only validation that actually matters — the browser form is just a UI.
- Intruder turns one captured request into an automated attack by substituting a wordlist into a marked payload position — exactly how real credential-stuffing/brute-force tools work under the hood.
- Sorting by response length (or using Grep-Match) is usually faster than relying on HTTP status codes alone, since many apps return the same status for both success and failure.
