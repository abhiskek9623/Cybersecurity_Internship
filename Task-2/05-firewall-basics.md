# Firewall Basics — iptables

After scanning Metasploitable2 and seeing just how much is left wide open on that box (FTP backdoors, Telnet, old SMB, you name it), the last part of this task was to actually do something about it — write some basic firewall rules with `iptables` and prove they work by scanning the target again and watching the difference.

`iptables` is the packet filtering tool built into the Linux kernel (via netfilter). It works by matching rules against a chain — for incoming traffic, that's the `INPUT` chain — and deciding whether to `ACCEPT`, `DROP`, or `REJECT` a packet based on things like port, protocol, and source.

Ran all of this on the Kali VM, applying rules to control traffic and then scanning from a second VM to confirm the effect from the attacker's point of view.

---

## 1. Checking the current rules first

Before touching anything, it's worth seeing what's already there:

```bash
sudo iptables -L -v
```

On a fresh setup this usually comes back empty (default ACCEPT policy on all chains), which is exactly the problem — no filtering at all means every port Nmap found earlier is reachable from anywhere.

---

## 2. Allowing SSH explicitly

```bash
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

- `-A INPUT` → append this rule to the INPUT chain (traffic coming into the machine)
- `-p tcp` → only match TCP traffic
- `--dport 22` → only match traffic going to port 22 (SSH)
- `-j ACCEPT` → jump to the ACCEPT target, i.e. let it through

This doesn't do much on its own since the default policy already accepts everything, but it matters once a DROP-everything policy gets added later — SSH needs to be allowed *before* you lock the door, otherwise you lock yourself out of your own machine.

---

## 3. Blocking Telnet

```bash
sudo iptables -A INPUT -p tcp --dport 23 -j DROP
```

Same structure as above, except the target is `DROP` instead of `ACCEPT`. DROP silently discards the packet — no response goes back to whoever sent it, which is different from REJECT (which sends back an explicit "connection refused"). Silently dropping is generally the better choice from a security standpoint since it doesn't confirm to a scanner that something is actively filtering that port.

Telnet was an obvious pick here since it sends credentials in plaintext — one of the easier wins from the earlier scan.

---

## 4. Verifying the rules

```bash
sudo iptables -L -v
```

**Output after adding the two rules above:**

```
Chain INPUT (policy ACCEPT)
target     prot opt in     out     source               destination
ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh
DROP       tcp  --  any    any     anywhere             anywhere             tcp dpt:telnet
```

The `-v` flag adds packet/byte counters and interface info, so you can also see how many packets have actually hit each rule — useful for confirming a rule is doing something rather than just sitting there unused.

---

## 5. Demonstrating a blocked port scan

This is the part that actually proves the firewall is working, rather than just trusting the config.

**Step 1 — scan the target before adding any blocking rule**, from the second VM:

```bash
nmap -p 23 192.168.112.133
```

Result: port 23 shows up as `open` (Telnet responding normally).

**Step 2 — add a rule to drop scan-style traffic on the target**, on the Kali/target VM:

```bash
sudo iptables -A INPUT -p tcp --dport 23 -j DROP
```

**Step 3 — re-run the exact same scan** from the other VM:

```bash
nmap -p 23 192.168.112.133
```

Result: port 23 now shows as `filtered` instead of `open`.

That change from `open` to `filtered` is the actual proof. Nmap reports a port as `filtered` when it sends a probe and gets no response at all — which is exactly what happens when a firewall silently drops the packet instead of the service replying. If the rule had used `REJECT` instead of `DROP`, Nmap would typically show `closed` instead, since it would receive an explicit RST back.

| Before rule | After rule | Nmap sees |
|---|---|---|
| No firewall | — | `23/tcp open telnet` |
| — | `DROP` rule active | `23/tcp filtered` |
| — | `REJECT` rule active | `23/tcp closed` |

---

## 6. Blocking scan-style traffic more broadly

The task also asks to block "scan-style" traffic specifically, not just a single port. A couple of ways to approach that:

**Drop all traffic to a range of commonly scanned/unused ports:**
```bash
sudo iptables -A INPUT -p tcp --dport 135:139 -j DROP
```

**Rate-limit new connections to catch fast port scans (Nmap's default SYN scan hits many ports quickly):**
```bash
sudo iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP
```

This second approach allows a normal, slow trickle of new connections through but drops anything coming in faster than 1 new SYN packet per second per matching rule — which is roughly what a scanner rapid-firing SYN packets across hundreds of ports would trigger. A legit user opening a single connection wouldn't notice; Nmap sweeping the host would get mostly dropped packets and a scan full of `filtered` ports.

---

## 7. Cleaning up / resetting rules

Good to know for testing, since rules persist until removed or the system reboots:

```bash
sudo iptables -D INPUT -p tcp --dport 23 -j DROP   # delete a specific rule
sudo iptables -F                                    # flush all rules in the chain
```

---

## Takeaway

The interesting part of this step wasn't writing the rules — that's two lines. It was seeing the actual before/after in Nmap: the same port, same target, same scan command, but the result flips from `open` to `filtered` the moment the DROP rule goes in. That's the whole point of a firewall from an attacker's perspective — it doesn't have to fix the vulnerable service behind the port, it just needs to stop the traffic from reaching it in the first place.
