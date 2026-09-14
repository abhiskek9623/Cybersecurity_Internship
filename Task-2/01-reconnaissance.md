# Reconnaissance

Reconnaissance (recon) is the first phase of penetration testing / ethical hacking. The goal is to gather as much information as possible about a target **before** interacting with it directly (or with minimal, low-noise interaction). Recon is broadly split into two types:

- **Passive Recon** – Gathering information without directly touching/contacting the target system. Leaves no trace on the target.
- **Active Recon** – Directly interacting with the target system to gather information. Can be detected/logged by the target.

> ⚠️ **Disclaimer:** All commands below were run against domains I own/control, public lab targets, or my own isolated lab VMs (Kali + Metasploitable2) for educational purposes only. Do not run active recon against systems you don't have explicit permission to test.

---

## 1. Passive Reconnaissance

Passive recon relies on publicly available information — DNS records, domain registration data, search engines, and public device databases (Shodan) — without sending traffic directly to the target that would reveal the tester's intent.

### 1.1 WHOIS Lookup

`whois` queries domain registration databases to reveal information such as the registrar, creation/expiry dates, name servers, and abuse contact details for a domain.

**Command:**
```bash
whois google.com
```

**Sample Output:**
```
Domain Name: GOOGLE.COM
Registry Domain ID: 2138514_DOMAIN_COM-VRSN
Registrar WHOIS Server: whois.markmonitor.com
Registrar URL: http://www.markmonitor.com
Updated Date: 2019-09-09T15:39:04Z
Creation Date: 1997-09-15T04:00:00Z
Registry Expiry Date: 2028-09-14T04:00:00Z
Registrar: MarkMonitor Inc.
Registrar IANA ID: 292
Registrar Abuse Contact Email: abusecomplaints@markmonitor.com
Registrar Abuse Contact Phone: +1.2086851750
Domain Status: clientDeleteProhibited
Domain Status: clientTransferProhibited
Domain Status: clientUpdateProhibited
Domain Status: serverDeleteProhibited
Domain Status: serverTransferProhibited
Domain Status: serverUpdateProhibited
Name Server: NS1.GOOGLE.COM
Name Server: NS2.GOOGLE.COM
Name Server: NS3.GOOGLE.COM
Name Server: NS4.GOOGLE.COM
DNSSEC: unsigned
```

**What this tells an attacker/analyst:**
| Field | Significance |
|---|---|
| Registrar / Registrar URL | Where the domain was purchased — useful for social engineering or abuse reporting |
| Creation / Expiry Date | Domain age (older domains are usually more "trusted") |
| Name Servers | Reveals the DNS provider, which can hint at hosting infrastructure |
| Domain Status (lock flags) | Shows if the domain is protected against unauthorized transfer/deletion |
| Abuse Contact | Useful for legitimate reporting, not for attacks |

---

### 1.2 Nslookup (DNS Lookup)

`nslookup` resolves a domain name to its IP address using the DNS server configured on the machine. It can also be used to query specific record types (MX, NS, TXT, etc.).

**Command:**
```bash
nslookup bmsit.ac.in
```

**Sample Output:**
```
Server:         192.168.112.2
Address:        192.168.112.2#53

Non-authoritative answer:
Name:   bmsit.ac.in
Address: 13.200.101.137
```

**Explanation:**
- `Server` / `Address` → the DNS resolver used to answer the query (in this case the local/lab DNS server).
- `Non-authoritative answer` → means the response came from a DNS server's cache rather than directly from the domain's authoritative name server.
- `Name` / `Address` → the resolved IP address of the target domain — this becomes a starting point for further recon (e.g., WHOIS on the IP, port scanning).

**Other useful nslookup queries:**
```bash
nslookup -type=mx bmsit.ac.in     # Find mail servers
nslookup -type=ns bmsit.ac.in     # Find name servers
nslookup -type=txt bmsit.ac.in    # Find TXT records (SPF, verification records)
```

---

### 1.3 Google Dorking

Google Dorking uses advanced Google search operators to find sensitive information indexed by search engines — exposed files, login portals, misconfigured directories, etc. — without ever touching the target server directly.

**Query used:**
```
site:nasa.gov intitle:report filetype:pdf
```

**Sample Results:**
- `NASA (.gov) – REPORT DOCUMENTATION PAGE (PDF)`
- `NASA (.gov) – Report From the MPP Working Group for Space Science (PDF)`
- `NASA (.gov) – Advance Confidential Report E5L18 (PDF)`

**Breakdown of operators used:**
| Operator | Purpose |
|---|---|
| `site:nasa.gov` | Restrict results to only the nasa.gov domain |
| `intitle:report` | Only show pages/documents with "report" in the title |
| `filetype:pdf` | Only show PDF documents |

**Other commonly used Google Dork operators:**
```
site:example.com filetype:xlsx                  → exposed spreadsheets
site:example.com inurl:admin                     → admin login pages
site:example.com intext:"index of /"             → open directory listings
site:example.com ext:sql                         → exposed SQL/database dump files
site:example.com inurl:login                     → login pages
```

**Why it matters:** Organizations often unintentionally expose internal documents, backup files, or admin panels that get indexed by search engines. Google Dorking helps identify this exposed attack surface passively.

---

### 1.4 Shodan

Shodan is a search engine for internet-connected devices. Unlike Google (which indexes web pages), Shodan indexes banners/metadata from open ports on servers, IoT devices, webcams, industrial control systems, etc.

**Typical usage (via [shodan.io](https://www.shodan.io)):**
```
apache city:"Bangalore"          → Apache servers in Bangalore
org:"NASA"                       → Devices/services belonging to NASA's ASN
port:21 country:"IN"             → FTP servers (port 21) hosted in India
```

**Shodan CLI (optional, if API key configured):**
```bash
shodan search apache country:IN
shodan host <ip_address>
```

**What Shodan reveals:**
- Open ports and the service/banner running on them
- Software/firmware versions (helps identify outdated, vulnerable software)
- Geolocation and ISP/organization info
- Sometimes default credentials or unauthenticated access on exposed devices

> Used responsibly for research/awareness — never to access systems without authorization.

---

## 2. Active Reconnaissance

Active recon involves direct interaction with the target system — this generates traffic/logs on the target and can be detected by firewalls/IDS.

### 2.1 Ping Sweep (Host Discovery)

`ping` checks whether a host is alive by sending ICMP Echo Request packets and measuring the response.

**Command:**
```bash
ping -c 5 192.168.112.133
```

**Sample Output:**
```
PING 192.168.112.133 (192.168.112.133) 56(84) bytes of data.
64 bytes from 192.168.112.133: icmp_seq=1 ttl=64 time=3.11 ms
64 bytes from 192.168.112.133: icmp_seq=2 ttl=64 time=3.08 ms
64 bytes from 192.168.112.133: icmp_seq=3 ttl=64 time=0.657 ms
64 bytes from 192.168.112.133: icmp_seq=4 ttl=64 time=0.532 ms
64 bytes from 192.168.112.133: icmp_seq=5 ttl=64 time=0.684 ms

--- 192.168.112.133 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4059ms
rtt min/avg/max/mdev = 0.532/1.612/3.106/1.210 ms
```

**Explanation:**
- `-c 5` → send exactly 5 ICMP packets (instead of pinging indefinitely).
- `ttl=64` → Time To Live value; can hint at the target's OS (Linux typically starts at 64, Windows at 128).
- `time=` → round-trip time in milliseconds — lower values mean the host is close/responsive on the network.
- `0% packet loss` → confirms the host is up and reachable.

**Ping sweep across a subnet (to find all live hosts):**
```bash
nmap -sn 192.168.112.0/24
for ip in 192.168.112.{1..254}; do ping -c 1 -W 1 $ip | grep "bytes from"; done
```

---

### 2.2 Nmap Host Discovery (`-sn`)

`nmap -sn` performs a **ping scan only** — it discovers which hosts on a network are up, without scanning any ports. Faster and stealthier than a full port scan when the goal is just to map live hosts.

**Command:**
```bash
nmap -sn 192.168.112.0/24
```

**Expected Output (example):**
```
Nmap scan report for 192.168.112.1
Host is up (0.00050s latency).
Nmap scan report for 192.168.112.133
Host is up (0.00061s latency).
Nmap scan report for 192.168.112.254
Host is up (0.00048s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.14 seconds
```

**Explanation:**
- `-sn` = "no port scan" — disables port scanning, uses ICMP echo, TCP SYN/ACK, and ARP requests (on local networks) to determine if a host is alive.
- Output lists every responsive IP in the target range — this becomes the host list for the next phase (port scanning with `-sS`, `-sV`, etc.).
- On a local subnet, Nmap also uses **ARP requests**, which are more reliable than ICMP since many hosts block ping but still respond to ARP.

---

### 2.3 Netcat (`nc`) – Banner Grabbing

`nc` (netcat) is a versatile networking utility used here to connect to an open port and read the **service banner** — text a server sends when a client connects, often revealing the software name and version.

**Command:**
```bash
nc -nv 192.168.112.133 21
```

**Explanation of flags:**
| Flag | Meaning |
|---|---|
| `-n` | Skip DNS resolution (use raw IP only, faster) |
| `-v` | Verbose output (shows connection status) |
| `21` | Target port (FTP in this example) |

**Sample Output:**
```
(UNKNOWN) [192.168.112.133] 21 (ftp) open
220 (vsFTPd 2.3.4)
```

**Why banner grabbing matters:**
- Reveals the exact service and version running (`vsFTPd 2.3.4` in this case).
- That version can then be checked against known CVEs / exploit databases (e.g., `vsftpd 2.3.4` has a well-known backdoor vulnerability — CVE-2011-2523).
- Can be done against any port: `nc -nv <ip> 80` for HTTP banners, `nc -nv <ip> 22` for SSH version, etc.

**Sending a manual HTTP request via nc:**
```bash
nc -nv 192.168.112.133 80
HEAD / HTTP/1.1
Host: 192.168.112.133

```
(Press Enter twice after typing the request to send it.)

---

## Summary Table

| Type | Technique | Tool | Touches Target? |
|---|---|---|---|
| Passive | Domain registration info | `whois` | ❌ No |
| Passive | DNS resolution | `nslookup` | ❌ No (queries DNS resolver, not target) |
| Passive | Exposed files/pages via search engine | Google Dorking | ❌ No |
| Passive | Exposed devices/services | Shodan | ❌ No |
| Active | Host discovery (single host) | `ping -c` | ✅ Yes |
| Active | Host discovery (subnet) | `nmap -sn` | ✅ Yes |
| Active | Banner grabbing / service ID | `nc` | ✅ Yes |

---

## Key Takeaways

- Always start with **passive recon** to build a picture of the target with zero footprint before moving to active techniques.
- Active recon (ping, Nmap, netcat) generates traffic on the target network and can be logged, so it should only be performed in an authorized lab environment or with explicit written permission.
- Information gathered here (IP addresses, live hosts, service banners, DNS records) directly feeds into the next phase: **Port & Service Scanning** (see `02-scanning.md`).
