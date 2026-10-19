# System Hardening — Patches, Firewall, Unused Services

Step 6 of Task 4. Where `12-system-hardening.md` documents fixing one specific finding (the vsftpd backdoor), this note covers the three general hardening actions called for in this task and applies each one across Metasploitable2 as a whole — not just the one service that was exploited. The idea is to take everything found during Task 2's Nmap scan and actually work through it: patch what can be patched, firewall what's reachable, and shut off everything that doesn't need to be running at all.

**Target:** Metasploitable2 (`192.168.112.133`)

---

## 1. Apply Security Patches

### Check what's currently installed and out of date
```bash
sudo apt-get update
sudo apt list --upgradable
```

### What actually happens on Metasploitable2
This is worth documenting honestly rather than skipping: Metasploitable2 ships on an Ubuntu 8.04 base with repository URLs pointing at mirrors that have been retired for years. Running the above typically returns connection failures (`Could not resolve host`, `404 Not Found`) rather than a list of available updates — the box is *intentionally* frozen in an unpatched state, since its entire purpose is to stay vulnerable for training.

```bash
$ sudo apt-get update
Err http://security.ubuntu.com hardy-security/main Translation-en_US
  Unable to connect to security.ubuntu.com:http:
...
```

This is itself a legitimate finding worth including in a report: **a system that can no longer receive updates at all is a standing risk regardless of what's currently exploitable**, since any new vulnerability discovered in its software from this point forward can never be patched through normal channels.

### What proper patch management looks like (for the report / in a real environment)
```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
sudo apt-get autoremove -y
```
On a modern, supported system, this sequence pulls and applies all available security updates, including ones that might require new dependencies (`dist-upgrade`), then cleans up packages no longer needed.

### What to document when patching isn't possible
Since this specific box can't actually be patched through its package manager:
- Note the **end-of-life status** of the underlying OS (Ubuntu 8.04 reached EOL in 2013) as a finding on its own.
- Recommend **decommissioning or upgrading** the base OS as the only real long-term fix — no amount of individual patching matters if the OS itself no longer receives security updates.
- In the meantime, document that the remaining two controls (firewalling, disabling services) are **compensating controls** standing in for patching, not a substitute for it.

---

## 2. Configure Firewall to Block Malicious Traffic

Building directly on the iptables work from Task 2, applied here across every risky service identified in the original Nmap scan rather than just port 21.

### Starting point — review what's actually exposed
Pulled straight from the Task 2 scan report:
```
21/tcp   open  ftp         vsftpd 2.3.4            ← known backdoor
23/tcp   open  telnet      Linux telnetd            ← plaintext credentials
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn Samba
445/tcp  open  netbios-ssn Samba
512/tcp  open  exec        netkit-rsh rexecd         ← legacy trust-based auth
513/tcp  open  login
514/tcp  open  tcpwrapped
1099/tcp open  java-rmi
1524/tcp open  bindshell   Metasploitable root shell ← literally a planted backdoor
2049/tcp open  nfs
3306/tcp open  mysql
5432/tcp open  postgresql
5900/tcp open  vnc
```

### Default-deny approach (the proper way to firewall a box like this)
Rather than writing one DROP rule per bad port, the professional approach is to default to denying everything, then explicitly allow only what's actually needed:

```bash
# Allow established/related connections (so return traffic isn't blocked)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback traffic
sudo iptables -A INPUT -i lo -j ACCEPT

# Explicitly allow SSH (needed to administer the box)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Set default policy to DROP everything else
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
```

This single change closes off all 20 other open ports from the earlier scan at once, rather than needing a DROP rule written individually for each one — a default-deny posture is both more secure (nothing is exposed unless explicitly allowed) and easier to maintain (a newly-started service doesn't become reachable by accident).

### Targeted rules (if default-deny isn't appropriate for the exercise)
For a demo where you want to show individual before/after comparisons per service, explicit DROP rules work too:
```bash
sudo iptables -A INPUT -p tcp --dport 21 -j DROP    # vsftpd backdoor
sudo iptables -A INPUT -p tcp --dport 23 -j DROP    # Telnet — plaintext creds
sudo iptables -A INPUT -p tcp --dport 512 -j DROP   # rexec
sudo iptables -A INPUT -p tcp --dport 513 -j DROP   # rlogin
sudo iptables -A INPUT -p tcp --dport 1524 -j DROP  # planted bindshell
sudo iptables -A INPUT -p tcp --dport 2049 -j DROP  # NFS
sudo iptables -A INPUT -p tcp --dport 5900 -j DROP  # VNC
```

### Verify and persist
```bash
sudo iptables -L -v
sudo apt-get install iptables-persistent -y
sudo netfilter-persistent save
```
(As noted in `12-system-hardening.md`, the persistence package may fail to install on this box due to the same dead-repo issue — worth noting the rules would need re-applying after reboot, or added to `/etc/rc.local`, in that case.)

### Confirm from the attacker's side
```bash
nmap -sS 192.168.112.133
```
Compare directly against the original Task 2 scan — the port list should shrink dramatically, ideally down to just SSH (22) if default-deny was used.

---

## 3. Disable Unused Services

Firewalling blocks network access to a service; disabling it removes the risk entirely, including any local/non-network attack surface. Both matter — a firewalled-but-still-running service is still a risk if the firewall rule is ever removed or misconfigured later.

### List what's actually running
```bash
sudo netstat -tulnp
```
or, cross-referencing against installed packages:
```bash
dpkg -l | grep -Ei "vsftpd|telnet|rsh|rlogin|nfs-kernel|samba|java-rmi"
```

### Stop and disable each unnecessary service
Working through the list from the Task 2 scan, one at a time:

```bash
# vsftpd — backdoored FTP (also covered in 12-system-hardening.md)
sudo service vsftpd stop
sudo update-rc.d vsftpd remove

# Telnet — plaintext credentials, no legitimate reason to keep it running
sudo service openbsd-inetd stop     # telnetd is typically launched via inetd on this box
sudo update-rc.d openbsd-inetd remove

# rexec/rlogin — legacy host-trust based remote access ("r-services")
sudo service rexec stop
sudo service rlogin stop

# NFS — unauthenticated file sharing if exports aren't locked down
sudo service nfs-kernel-server stop
sudo update-rc.d nfs-kernel-server remove

# Samba — only needed if Windows file sharing is actually required
sudo service smbd stop
sudo update-rc.d smbd remove

# VNC — remote desktop with weak legacy auth
sudo service vnc4server stop
```

### Verify each one actually stopped
```bash
sudo netstat -tulnp | grep -E "21|23|512|513|2049|139|445|5900"
```
Should return nothing (or dramatically less) for the services just disabled.

### A general principle worth stating in the report
Every open port represents something that has to be defended; a port that isn't open doesn't need defending at all. The guiding question for any running service should be **"does this specific host need to provide this specific service to anyone?"** — if the honest answer is no, or "not right now," disabling it outright is strictly better than trying to secure something nobody actually needs exposed.

---

## 4. Putting It Together — Before/After Comparison

| | Before Hardening | After Hardening |
|---|---|---|
| Open ports (Nmap) | 20 ports open, including FTP backdoor, Telnet, r-services, NFS, VNC | Only SSH (22) reachable, assuming default-deny firewall policy |
| vsftpd 2.3.4 backdoor | Exploitable — grants instant root shell | Service stopped, disabled, port blocked |
| Telnet | Plaintext credential exposure | Service disabled |
| OS patch status | Unpatchable (dead repos, EOL Ubuntu 8.04) | Documented as a standing risk; compensating controls applied |
| Firewall policy | None — default ACCEPT on all traffic | Default DROP, explicit allow-list only |

---





