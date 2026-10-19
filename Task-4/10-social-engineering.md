# Social Engineering (Simulation Only) — SEToolkit

Part of Task 4. Unlike every other technique in this task, this one doesn't target a software vulnerability at all — it targets the human in front of the keyboard. The Social-Engineer Toolkit (SEToolkit) was used to clone a real login page and set up a credential harvester, demonstrating exactly how a phishing page works from the attacker's side. This was run strictly as a local simulation: the cloned page was only ever accessed from another tab on the same lab machine, no real person's credentials were ever at risk, and nothing was hosted anywhere reachable outside the isolated lab network.

> ⚠️ This technique has real legal and ethical weight attached to it. Cloning a login page and harvesting credentials against anyone who hasn't explicitly consented is illegal in virtually every jurisdiction. Everything documented here stayed entirely within a local, isolated lab — the "victim" was the same operator, submitting test credentials into their own cloned page on their own machine.

---

## 1. Setting Up the Credential Harvester

SEToolkit is launched with root privileges, since it needs to bind to port 80 to serve the cloned page:
```bash
sudo setoolkit
```

Navigating the menu:
1. **1) Social-Engineering Attacks**
2. **2) Website Attack Vectors**
3. **3) Credential Harvester Attack Method**
4. **2) Site Cloner**

At the Site Cloner step, SEToolkit asks for the IP to host the cloned page on (the Kali machine's own IP, so the fake page is actually reachable) and which real site to clone. A built-in template option is also available — the Google login template was selected here rather than manually cloning a live URL:

![SEToolkit template selection — choosing the Google credential harvester template](./screenshots/set-template-selection.png)

The tool confirms the clone is in progress ("Cloning the website: http://www.google.com") and then starts the harvester listener on port 80, warning that the technique works best when real username/password field names are detected, but will capture all POST data from the page regardless.

---

## 2. The Cloned Page, From the "Victim's" Side

Browsing to the Kali machine's IP from another tab shows a pixel-for-pixel clone of Google's actual sign-in page — same layout, same branding, same form fields:

![Cloned Google sign-in page served from the attacker's own Kali IP](./screenshots/cloned-google-login.png)

The only thing that gives it away, if you know to look, is the address bar: `192.168.112.135` instead of `accounts.google.com`. Visually, there's nothing else distinguishing it from the real thing — which is exactly the point of this exercise. A real phishing page would typically pair this clone with a lookalike domain (e.g. `accoounts-google-login.com`) to make even the address bar harder to catch at a glance.

Test credentials were submitted into the cloned form (`victim@gmail.com` / a placeholder password) to trigger the harvester.

---

## 3. What the Harvester Captured

The moment the form was submitted, SEToolkit's listener intercepted the POST request and printed every parameter it contained directly to the terminal in real time:

![SEToolkit harvester output showing captured credentials and full POST data](./screenshots/harvester-captured-creds.png)

```
192.168.112.135 - - [10/Oct/2026 16:10:25] "GET / HTTP/1.1" 200 -
192.168.112.135 - - [10/Oct/2026 16:10:26] "GET /favicon.ico HTTP/1.1" 404 -
[*] WE GOT A HIT! Printing the output:
PARAM: GALX=SJLCkfgaqoM
PARAM: continue=https://accounts.google.com/o/oauth2/auth?zt=...
PARAM: service=lso
PARAM: dsh=-7381887106725792428
PARAM: _utf8=&
PARAM: bgresponse=js_disabled
PARAM: pstMsg=1
PARAM: dnConn=
PARAM: checkConnection=
PARAM: checkedDomains=youtube
POSSIBLE USERNAME FIELD FOUND: Email=victim@gmail.com
POSSIBLE PASSWORD FIELD FOUND: Passwd=ViCtImQ777
PARAM: signIn=Sign+in
PARAM: PersistentCookie=yes
[*] WHEN YOU'RE FINISHED, HIT CONTROL-C TO GENERATE A REPORT.
```

**Breaking down what actually happened here:**
- SEToolkit doesn't just grab two fields — it logs **every single parameter** the real Google login form submits, including hidden anti-automation/session tokens like `GALX`, `dsh`, and the OAuth `continue` redirect URL. This is because the cloned page is a byte-for-byte copy of the real form, hidden fields included — the harvester just logs the entire POST body.
- It specifically flags `POSSIBLE USERNAME FIELD FOUND` and `POSSIBLE PASSWORD FIELD FOUND` by pattern-matching common field names (`Email`, `Passwd`, etc.), which is how it surfaces the actual credentials out of the noise of all the other captured parameters.
- All of this output — and a formatted report — is also saved locally for later review:
  ```bash
  ls /root/.set/reports/
  ```
  Hitting `Ctrl+C` in the harvester terminal triggers SEToolkit to generate a formatted HTML/XML report of everything captured during the session.

**Why this matters as a finding:** nothing about the page itself was broken into — this isn't an exploit against Google or against any software vulnerability at all. The entire "attack" succeeded purely because the page looked convincing enough, and the only signal available to catch it was the URL in the address bar. This is precisely why social engineering remains one of the most effective attack categories in real incidents — it bypasses every technical control (firewalls, patched software, strong passwords) by targeting the decision made by a person, not a flaw in code.

---

## 4. Awareness Training — What Someone Should Actually Check

This is the defensive half of the exercise: the signs that would have caught this specific fake page before any credentials were typed in.

- **URL mismatch** — the single most reliable tell here. The cloned page loaded at `192.168.112.135`, not `accounts.google.com`. A real phishing campaign usually hides behind a lookalike domain instead of a bare IP, so the habit to build is checking the *actual* domain in the address bar character-by-character, not just glancing at whether the page "looks right."
- **Missing HTTPS / invalid certificate warnings** — the cloned page in this test was served over plain HTTP on port 80, with no certificate at all. A real Google login always shows a valid HTTPS padlock; a missing padlock, a broken padlock icon, or a certificate warning is an immediate stop sign, not something to click through.
- **Urgency or fear-based language in whatever led someone there** — this demo skipped the delivery step (the email/message that would normally direct a victim to this page), but real phishing almost always pairs a convincing page like this with a message creating pressure: "your account will be suspended," "unusual sign-in detected," "action required within 24 hours." Urgency is a deliberate tactic to short-circuit the moment someone would otherwise pause and check the URL.
- **Generic greetings instead of a real name** — a legitimate security or account notification from a service you actually use typically addresses you by name; "Dear Customer" or no greeting at all is a common tell for mass-sent phishing that hasn't been personalized.
- **Hovering over links before clicking** — on desktop, hovering over a link (without clicking) shows the real destination URL in the browser's status bar. This is the single easiest habit to build and would have surfaced the real destination (the attacker's Kali IP, in this exercise) before ever landing on the fake page at all.

### A simple mental checklist worth including in any awareness material
1. Hover before you click — check where a link actually goes.
2. Check the domain, not just the page design — attackers copy pages perfectly; they can't fake the real domain.
3. Look for HTTPS and a valid certificate — no padlock, no trust.
4. Pause on urgency — legitimate services rarely demand immediate action via a link in an email.
5. When in doubt, navigate directly — type the known, real address into the browser yourself instead of clicking the link at all.

---

## 5. Mitigation (Organizational / Technical Side)

Awareness training catches what a person can be trained to notice, but shouldn't be the only layer of defense:

- **Multi-Factor Authentication (MFA)** — even if a password is successfully harvested, MFA means the captured credential alone isn't enough to log in. This is the single highest-leverage technical control against phishing specifically.
- **Email filtering / anti-phishing gateways** — catching the delivery mechanism (the phishing email itself) before it ever reaches an inbox.
- **Browser-level phishing protection** (Google Safe Browsing, Microsoft SmartScreen, etc.) — flags known-bad domains automatically, though this won't catch a brand-new cloned page on day one.
- **DMARC/DKIM/SPF** on the organization's own domain — prevents attackers from convincingly spoofing the organization's real email domain in a phishing campaign.
- **Regular, realistic phishing simulations** (exactly this kind of exercise, run internally and ethically against employees with consent and a defined program) — measurably improves real-world detection rates over time far more than a one-time slide deck ever does.
- **Credential monitoring** — services that alert when an organization's email addresses appear in known breach/phishing-harvest dumps, catching compromise after the fact if prevention fails.

---

## Key Takeaways

- SEToolkit's Site Cloner doesn't need to "hack" anything — it copies the real page's HTML wholesale, so visually it's indistinguishable from the real thing; the only honest signal left is the URL and certificate.
- The harvester captures the entire POST body, not just the obvious username/password fields — real login forms carry hidden session/anti-automation tokens that get swept up too, which is worth knowing when interpreting captured data in a real incident response scenario.
- This attack class succeeds against perfect software with zero vulnerabilities — it's purely a human-decision exploit, which is exactly why it needs its own category of defense (training + MFA) rather than being treated as something patching can fix.
- Hovering before clicking and checking the actual domain are the two highest-value, lowest-effort habits for defeating this specific attack technique.
