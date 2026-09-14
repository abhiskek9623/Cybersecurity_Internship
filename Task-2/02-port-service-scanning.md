# Port & Service Scanning

Once live hosts are identified during reconnaissance, the next step is to find out **what's actually running on them**. This is where Nmap comes in — it probes ports on a target and tells you which ones are open, what service is behind each port, and (when possible) the exact software version and OS of the machine.

Target used for this task: **Metasploitable2** (`192.168.112.133`), an intentionally vulnerable VM built for practicing exactly this kind of scanning in a safe, legal environment.

---

## 1. The Scan Command

```bash
sudo nmap -sS -sV -O -oN scan_report.txt 192.168.112.133
```

Breaking this down flag by flag:

| Flag | What it does |
|---|---|
| `-sS` | TCP SYN scan (aka "half-open" scan). Sends a SYN packet and waits for SYN-ACK, but never completes the handshake. Faster and quieter than a full TCP connect scan, and needs root/privileged access. |
| `-sV` | Service/version detection. Nmap doesn't just say "port open" — it probes the service to figure out exactly which software and version is listening. |
| `-O` | OS detection. Nmap fingerprints the TCP/IP stack behavior to guess the target's operating system. |
| `-oN scan_report.txt` | Saves the output in normal (human-readable) format to a file, so it can be attached as a deliverable instead of copy-pasting terminal output. |

This combination is basically the "give me everything" scan — good for an initial full assessment of a target once you already know it's alive.

---

## 2. Raw Scan Output

```
Nmap scan report for 192.168.112.133
Host is up (0.0025s latency).
Not shown: 977 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
23/tcp   open  telnet      Linux telnetd
25/tcp   open  smtp        Postfix smtpd
53/tcp   open  domain      ISC BIND 9.4.2
80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
111/tcp  open  rpcbind     2 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
512/tcp  open  exec        netkit-rsh rexecd
513/tcp  open  login
514/tcp  open  tcpwrapped
1099/tcp open  java-rmi    GNU Classpath grmiregistry
1524/tcp open  bindshell   Metasploitable root shell
2049/tcp open  nfs         2-4 (RPC #100003)
2121/tcp open  ftp         ProFTPD 1.3.1
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
5432/tcp open  postgresql  PostgreSQL DB 8.3.0 - 8.3.7
5900/tcp open  vnc         VNC (protocol 3.3)
```

`Not shown: 977 closed tcp ports (reset)` — out of the 1000 default ports Nmap checks, 977 sent back a RST (closed, nothing listening). The 23 shown above are the ones actually worth looking at.

---

## 3. Going Through Each Port

This is the important part — not just listing ports, but actually reasoning about what each one means.

**21/tcp – FTP – vsftpd 2.3.4**
This exact version has a well-known backdoor (CVE-2011-2523) that was maliciously planted in the source code for a short period. If a target is running this specific version, it's basically an instant red flag in any real assessment.

**22/tcp – SSH – OpenSSH 4.7p1 (Debian)**
An old build of OpenSSH. Not necessarily broken on its own, but old enough that it's worth checking against known CVEs for that release, and it tells you the box hasn't been patched in a long time.

**23/tcp – Telnet**
Telnet sends everything — including login credentials — in plaintext. Just having this port open is a finding by itself, regardless of what's running behind it.

**25/tcp – SMTP – Postfix**
Mail server. Worth checking for open relay (can it be used to send mail as/for anyone) and doing a `VRFY`/user enumeration check.

**53/tcp – DNS – ISC BIND 9.4.2**
An outdated BIND version. Old BIND releases have had multiple CVEs over the years (cache poisoning, DoS). Worth a version-specific CVE lookup.

**80/tcp – HTTP – Apache 2.2.8 (Ubuntu) DAV/2**
Web server, and the `DAV/2` in the banner means WebDAV is enabled — which can sometimes allow file upload if permissions aren't locked down properly. This is normally where you'd move into web app testing (directory brute-forcing, checking for known vulnerable apps hosted here, etc.).

**111/tcp – rpcbind**
Maps RPC program numbers to ports. Not dangerous by itself, but it tells an attacker what RPC-based services (like NFS) are running underneath, which helps plan the next steps.

**139/tcp & 445/tcp – Samba (SMB) 3.X–4.X**
SMB is one of the most commonly abused services in internal networks — used for everything from enumeration to full remote code execution depending on the version and config. Worth an `enum4linux` or `smbclient -L` pass to check for open shares and null sessions.

**512/513/514/tcp – rexec / rlogin / rsh (the "r-services")**
These are ancient remote-access protocols from before SSH existed. They authenticate based on IP/hostname trust rather than real credentials in a lot of default configs, which makes them very easy to abuse if enabled.

**1099/tcp – Java RMI**
Java RMI registries are a known vector for remote code execution when misconfigured, since a client can sometimes get the server to load and execute arbitrary Java classes.

**1524/tcp – "Metasploitable root shell" (bindshell)**
This one's specific to the lab — Metasploitable2 deliberately leaves a backdoor shell bound to this port left over from an earlier compromise simulation, meant to be found and used for practice.

**2049/tcp – NFS**
Network File System. If exports aren't restricted properly, this can allow reading/writing to shared directories from any host on the network without authentication.

**2121/tcp – FTP – ProFTPD 1.3.1**
A second FTP service on a non-standard port, running a different (and also outdated) FTP daemon. Older ProFTPD builds have had remote code execution vulnerabilities.

**3306/tcp – MySQL 5.0.51a**
Very old MySQL build. Worth checking for weak/default root credentials and whether it's reachable from outside localhost (it shouldn't be, in a properly hardened setup).

**5432/tcp – PostgreSQL 8.3.x**
Same idea as MySQL above — old version, worth checking auth config and default credentials.

**5900/tcp – VNC (protocol 3.3)**
Remote desktop access. VNC protocol 3.3 is old enough that some implementations from that era support very weak (or blank) authentication.

---

## 4. Quick Risk Summary

| Port | Service | Immediate concern |
|---|---|---|
| 21 | vsftpd 2.3.4 | Known backdoor (CVE-2011-2523) |
| 23 | Telnet | Plaintext credentials |
| 139/445 | Samba | Enumeration / possible RCE |
| 512-514 | r-services | Weak host-based trust auth |
| 1099 | Java RMI | Possible RCE via deserialization |
| 1524 | Bindshell | Already an open root shell |
| 2049 | NFS | Possible unauthenticated file access |
| 3306/5432 | MySQL/PostgreSQL | Old versions, check default creds |
| 5900 | VNC | Weak/no auth in old protocol versions |

Basically the entire box is a checklist of "what not to leave running on a production server" — which is exactly the point of Metasploitable2 as a training target.

---

## 5. Follow-Up Commands

Some natural next steps after this initial scan, before moving into vulnerability scanning:

```bash
# Full port range instead of just the default top 1000
nmap -p- 192.168.112.133

# Run default Nmap scripts against the discovered services
nmap -sC -sV 192.168.112.133

# Check for the vsftpd 2.3.4 backdoor specifically
nmap --script ftp-vsftpd-backdoor -p 21 192.168.112.133

# Enumerate SMB shares
enum4linux 192.168.112.133
smbclient -L //192.168.112.133 -N
```

---

## 6. Takeaway

This single scan already gives a strong picture of the attack surface: a mix of ancient, unpatched services, plaintext protocols, and at least one deliberately planted backdoor. In a real engagement, this output alone would be enough to prioritize which services to dig into first (FTP and the bindshell being the obvious starting points here) before moving on to the vulnerability scanning phase with OpenVAS/Nessus.
