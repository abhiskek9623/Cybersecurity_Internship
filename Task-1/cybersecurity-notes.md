## The CIA Triad

Basically every security concept ties back to one of these three things.

**Confidentiality** - keeping data away from people who shouldn't see it. Think encryption, passwords, access controls. If your medical records leak, that's a confidentiality failure.

**Integrity** - making sure data hasn't been tampered with. If someone changes your bank balance without authorization, that's an integrity problem, even if they never actually read the data.

**Availability** - the system/data needs to be there when you need it. A server that's down because of an attack is an availability issue, even if nothing was stolen or changed.

People remember it as "can't read it, can't change it, can't lose it." Not perfect but it works.

One thing that trips people up at first: these three sometimes fight each other. Lock something down too hard for confidentiality (extra logins, extra encryption layers) and you hurt availability - it gets slower or harder to access. Security is usually about finding the right balance for the situation, not maxing out all three.

---

## Threat Types

### Phishing

Fake emails/texts/calls pretending to be someone trustworthy (your bank, your boss, IT support) to get you to click a link or hand over credentials.

Variants worth knowing:
- Spear phishing - targeted at one specific person, usually with info gathered about them beforehand
- Whaling - spear phishing but aimed at executives/high value targets
- Smishing - same idea, over text message
- Vishing - same idea, over phone calls

Red flags: urgency ("act now or your account is suspended"), weird sender addresses that almost match the real domain, links that don't go where they say they go.

Best defense honestly isn't technical - it's just training people to slow down and check before clicking.

### Malware

Umbrella term for anything malicious that runs on your system.

- Virus - needs a host file, spreads when you run that file
- Worm - spreads on its own across a network, no user action needed
- Trojan - pretends to be legit software, hides the bad stuff inside
- Spyware - quietly watches what you do, steals info over time
- Rootkit - buries itself deep, gives attacker persistent access, hard to detect

Patch your systems, don't run random downloads, use decent endpoint protection. Nothing fancy about the defense, just consistency.

### DDoS

Distributed Denial of Service. Flood a target with so much traffic from so many sources it can't serve real users anymore.

Comes in a few flavors:
- Volume-based - just raw traffic overload
- Protocol attacks - abuse how network protocols work (SYN floods etc.)
- Application layer - target a specific expensive operation on the app itself (like hammering a search function)

Mitigation is usually about scale - CDNs, traffic scrubbing, rate limiting, having enough capacity to absorb a spike while filtering out the junk.

### SQL Injection

Classic one. Attacker sneaks SQL code into an input field (login box, search bar, whatever) and the app runs it against the database like it was a normal query.

Old-school example - typing something like `' OR '1'='1` into a login field to bypass authentication because the resulting query always evaluates true.

Fix is simple in concept, annoying in practice for legacy code: use parameterized queries / prepared statements instead of building SQL strings by hand. Also validate input and don't give your app's DB account more privileges than it needs.

### Brute Force

Just trying combinations until something works. Could be guessing passwords one by one, running through a dictionary of common passwords, or trying leaked username/password pairs from other breaches (credential stuffing - this one works surprisingly often because people reuse passwords).

Defenses: lockout after failed attempts, rate limiting, CAPTCHAs, and honestly just MFA - even a correctly guessed password doesn't get you in if there's a second factor.

### Ransomware

Malware that encrypts your files then demands payment for the key. Usually starts with something boring - a phishing email, an exposed RDP port, an unpatched vulnerability - then spreads internally before the encryption actually kicks in.

Newer trend is "double extortion" - they steal your data before encrypting it, so even if you restore from backup they threaten to leak the stolen copy anyway.

Best defense is offline/immutable backups (ones the ransomware itself can't reach and encrypt), network segmentation so one infected machine doesn't take down everything, and just staying patched.

---

## Attack Vectors

### Social Engineering

This is the human side of hacking - manipulating people instead of exploiting code.

- Pretexting - inventing a believable story to get info ("Hi, I'm from IT, I need your password to fix an issue")
- Baiting - leaving something tempting around, like a USB drive labeled "Salaries 2024" in a parking lot
- Tailgating - literally following someone through a secure door without badging in yourself
- Quid pro quo - "I'll help you with X if you just give me Y"

No firewall stops this. It's all training, verification habits, and a culture where it's normal to double check before handing over sensitive stuff.

### Wireless Attacks

Anything that targets Wi-Fi/wireless comms specifically.

- Evil twin - a fake access point set up to look like the real one, so your device connects to the attacker instead
- Wardriving - literally driving around scanning for vulnerable networks
- Packet sniffing - grabbing unencrypted wireless traffic out of the air
- Deauth attacks - forcing devices off a network so they reconnect (sometimes to a fake AP)

Use WPA3 where you can, turn off WPS, don't use default router passwords, watch for rogue access points on your network.

### Insider Threats

Sometimes the risk is already inside the building/network.

- Malicious insider - someone who deliberately steals data or sabotages something (disgruntled employee is the classic case)
- Negligent insider - no bad intent, just careless - clicking the wrong link, misconfiguring a server, losing a laptop
- Compromised insider - an attacker is using a legitimate employee's stolen credentials, so it looks like normal activity

Hardest category to defend against because these people already have legitimate access. Least privilege access, monitoring for unusual behavior, and solid offboarding (revoking access the day someone leaves, not a week later) all help.

