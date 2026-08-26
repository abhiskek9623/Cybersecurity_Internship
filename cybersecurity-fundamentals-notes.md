# Cybersecurity Fundamentals — Study Notes

> A structured reference covering the CIA Triad, common threat types, and key attack vectors.

---

## Table of Contents

1. [The CIA Triad](#1-the-cia-triad)
2. [Threat Types](#2-threat-types)
   - [Phishing](#21-phishing)
   - [Malware](#22-malware)
   - [DDoS (Distributed Denial of Service)](#23-ddos-distributed-denial-of-service)
   - [SQL Injection](#24-sql-injection)
   - [Brute Force](#25-brute-force)
   - [Ransomware](#26-ransomware)
3. [Attack Vectors](#3-attack-vectors)
   - [Social Engineering](#31-social-engineering)
   - [Wireless Attacks](#32-wireless-attacks)
   - [Insider Threats](#33-insider-threats)
4. [Quick Reference Summary](#4-quick-reference-summary)

---

## 1. The CIA Triad

The **CIA Triad** is the foundational model for information security. Every control, policy, or defense mechanism generally maps back to protecting one (or more) of these three properties.

### 🔒 Confidentiality
Ensures information is accessible only to authorized individuals.

- **Goal:** Prevent unauthorized disclosure of data.
- **Mechanisms:**
  - Encryption (at rest & in transit)
  - Access control lists (ACLs)
  - Multi-factor authentication (MFA)
  - Data classification & least privilege
- **Violation example:** A data breach exposing customer records.

### ✅ Integrity
Ensures information is accurate, consistent, and unaltered except by authorized action.

- **Goal:** Prevent unauthorized modification or corruption of data.
- **Mechanisms:**
  - Hashing (SHA-256, etc.)
  - Digital signatures
  - Checksums
  - Version control & audit logs
- **Violation example:** An attacker modifying financial transaction records.

### ⚡ Availability
Ensures systems and data are accessible to authorized users when needed.

- **Goal:** Prevent disruption of access to services/data.
- **Mechanisms:**
  - Redundancy & failover systems
  - Load balancing
  - Regular backups
  - DDoS protection
- **Violation example:** A DDoS attack taking a website offline.

### CIA Triad Summary Table

| Property        | Protects Against         | Common Controls                     |
|------------------|--------------------------|--------------------------------------|
| Confidentiality  | Unauthorized disclosure  | Encryption, ACLs, MFA               |
| Integrity        | Unauthorized modification| Hashing, signatures, checksums      |
| Availability     | Service disruption       | Redundancy, backups, DDoS mitigation|

---

## 2. Threat Types

### 2.1 Phishing

A **social engineering** attack where attackers impersonate trusted entities to trick victims into revealing sensitive information or installing malware.

- **Common forms:**
  - **Email phishing** — mass fraudulent emails
  - **Spear phishing** — targeted at a specific individual/organization
  - **Whaling** — targets high-profile executives
  - **Smishing** — via SMS
  - **Vishing** — via voice calls
- **Indicators:**
  - Urgent/threatening language
  - Mismatched sender domains
  - Suspicious links/attachments
- **Defenses:**
  - Email filtering & anti-phishing gateways
  - Security awareness training
  - DMARC/SPF/DKIM email authentication

### 2.2 Malware

Malicious software designed to damage, disrupt, or gain unauthorized access to systems.

| Type       | Description                                       |
|------------|----------------------------------------------------|
| Virus      | Attaches to files, spreads when executed           |
| Worm       | Self-replicates across networks without user action|
| Trojan     | Disguised as legitimate software                   |
| Spyware    | Secretly monitors user activity                    |
| Rootkit    | Hides presence, grants persistent privileged access|
| Adware     | Delivers unwanted advertisements                    |

- **Defenses:**
  - Endpoint Detection & Response (EDR)
  - Regular patching
  - Application whitelisting
  - Antivirus/anti-malware tools

### 2.3 DDoS (Distributed Denial of Service)

Overwhelms a target system/network with traffic from multiple sources, rendering it unavailable.

- **Types:**
  - **Volumetric attacks** — flood bandwidth (e.g., UDP floods)
  - **Protocol attacks** — exploit protocol weaknesses (e.g., SYN floods)
  - **Application-layer attacks** — target specific app functions (e.g., HTTP floods)
- **Defenses:**
  - Content Delivery Networks (CDNs)
  - Rate limiting
  - Web Application Firewalls (WAF)
  - Traffic scrubbing services

### 2.4 SQL Injection

An injection attack where malicious SQL statements are inserted into input fields to manipulate a database.

- **Example (conceptual):**
  ```sql
  SELECT * FROM users WHERE username = 'admin' -- ' AND password = ''
  ```
- **Impact:**
  - Data theft or leakage
  - Authentication bypass
  - Data manipulation or deletion
- **Defenses:**
  - Parameterized queries / prepared statements
  - Input validation & sanitization
  - Least-privilege database accounts
  - Web Application Firewalls (WAF)

### 2.5 Brute Force

An attack method that systematically tries all possible password/key combinations until the correct one is found.

- **Variants:**
  - **Simple brute force** — tries all combinations
  - **Dictionary attack** — uses common word lists
  - **Credential stuffing** — reuses leaked username/password pairs
  - **Hybrid attack** — combines dictionary + brute force logic
- **Defenses:**
  - Account lockout policies
  - Rate limiting & CAPTCHA
  - Multi-factor authentication (MFA)
  - Strong password policies

### 2.6 Ransomware

Malware that encrypts a victim's files/systems and demands payment (ransom) for the decryption key.

- **Attack lifecycle:**
  1. Initial access (phishing, RDP exploitation, vulnerable software)
  2. Lateral movement across the network
  3. Data exfiltration (double extortion)
  4. Encryption of files/systems
  5. Ransom demand (often in cryptocurrency)
- **Defenses:**
  - Regular, isolated (offline/immutable) backups
  - Network segmentation
  - Endpoint protection & patch management
  - Incident response plan

---

## 3. Attack Vectors

### 3.1 Social Engineering

The psychological manipulation of people into performing actions or divulging confidential information.

- **Techniques:**
  - **Pretexting** — fabricating a scenario to extract info
  - **Baiting** — luring victims with something enticing (e.g., infected USB drives)
  - **Tailgating/Piggybacking** — following authorized personnel into secure areas
  - **Quid pro quo** — offering a service in exchange for information
- **Defenses:**
  - Security awareness training
  - Verification protocols for sensitive requests
  - Physical access controls

### 3.2 Wireless Attacks

Attacks that exploit vulnerabilities in Wi-Fi and other wireless communication protocols.

- **Common attacks:**
  - **Evil Twin** — rogue access point mimicking a legitimate one
  - **Wardriving** — searching for vulnerable wireless networks
  - **Packet sniffing** — intercepting unencrypted wireless traffic
  - **Deauthentication attacks** — forcibly disconnecting devices from a network
  - **WPA/WPA2 cracking** — exploiting weak encryption/handshakes
- **Defenses:**
  - Use WPA3 encryption
  - Disable WPS
  - Strong, unique Wi-Fi passwords
  - Network monitoring for rogue access points

### 3.3 Insider Threats

Security risks originating from within an organization (employees, contractors, partners).

- **Categories:**
  - **Malicious insider** — intentionally causes harm (theft, sabotage)
  - **Negligent insider** — causes harm through carelessness or error
  - **Compromised insider** — legitimate credentials used by an external attacker
- **Defenses:**
  - Least privilege & role-based access control (RBAC)
  - User activity monitoring & behavior analytics
  - Data Loss Prevention (DLP) tools
  - Strong offboarding processes

---

## 4. Quick Reference Summary

| Category         | Item              | Primary CIA Impact         |
|------------------|-------------------|------------------------------|
| Threat           | Phishing          | Confidentiality              |
| Threat           | Malware           | Confidentiality/Integrity    |
| Threat           | DDoS              | Availability                 |
| Threat           | SQL Injection     | Confidentiality/Integrity    |
| Threat           | Brute Force       | Confidentiality               |
| Threat           | Ransomware        | Availability/Confidentiality |
| Attack Vector    | Social Engineering| Confidentiality               |
| Attack Vector    | Wireless Attacks  | Confidentiality/Integrity    |
| Attack Vector    | Insider Threats   | All three (C/I/A)             |

---

## 📚 Suggested Further Study

- OWASP Top 10 (web application security risks)
- MITRE ATT&CK Framework (adversary tactics & techniques)
- NIST Cybersecurity Framework (CSF)
- CompTIA Security+ objectives

---

*Notes compiled for personal study and reference purposes.*
