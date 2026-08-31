# Security Tools Notes: Wireshark, Nmap, Burp Suite, Netcat

Notes on the four tools that come up constantly in networking/security work. These are all things I've actually run, not just read about, so the commands here are ones I've tested myself. Standard disclaimer that applies to all of this: only scan/capture/test on networks and systems you own or have explicit permission to test. Running these against stuff you don't own is illegal in most places, full stop.

## Wireshark (packet capture)

Wireshark captures and lets you inspect traffic going across a network interface, packet by packet. GUI tool mostly (there's a CLI version called tshark too), great for actually seeing what a protocol is doing instead of just reading about it.

### Getting started

Pick an interface to capture on (Wi-Fi, Ethernet, loopback, whatever you want to watch), hit start, and it just starts filling up with every packet crossing that interface. Can get overwhelming fast on a busy network, which is where filters come in.

### Filters that actually get used

```
http                        # only HTTP traffic
tcp.port == 443             # traffic on a specific port
ip.addr == 192.168.1.10     # traffic to/from a specific IP
dns                          # only DNS queries/responses
tcp.flags.syn == 1 && tcp.flags.ack == 0    # just the SYN packets (start of a TCP handshake)
http.request.method == "POST"                # only POST requests
```

You can also right-click any packet and "Follow > TCP Stream" to see the whole conversation reassembled instead of jumping packet to packet, this is way easier for actually reading what's going on in something like an HTTP request/response.

### What I actually use it for

- Watching a TCP handshake happen in real time (SYN, SYN-ACK, ACK) instead of just reading about it
- Confirming whether traffic is actually encrypted or not (if you can read the payload in plaintext, it's not TLS)
- Troubleshooting "why is this connection slow/failing" by watching retransmissions or resets
- Just generally building intuition for what protocols look like on the wire

### tshark (command line version)

If you don't want the GUI, or you're on a server with no display:

```
tshark -i eth0                          # capture on interface eth0
tshark -i eth0 -f "port 80"             # capture filter, only port 80
tshark -r capture.pcap                  # read a saved capture file instead of live capturing
tshark -i eth0 -w output.pcap           # write the capture to a file for later analysis in the GUI
```

---

## Nmap (network scanning)

Maps out what's alive on a network and what's running on it - open ports, services, sometimes OS guesses. This is the tool everyone learns first in networking/security courses because it makes the abstract "ports and services" concept concrete really fast.

### Basic scans

```
nmap 192.168.1.1                    # basic scan, default port list, single host
nmap 192.168.1.0/24                 # scan a whole subnet
nmap -p 80,443 192.168.1.1          # only scan specific ports
nmap -p- 192.168.1.1                # scan ALL 65535 ports, slow but thorough
```

### Scan types worth knowing

```
nmap -sS 192.168.1.1        # SYN scan ("stealth" scan) - doesn't complete the handshake, faster, needs root/admin
nmap -sT 192.168.1.1        # full TCP connect scan, completes the handshake, doesn't need elevated privileges
nmap -sU 192.168.1.1        # UDP scan, slower than TCP scans but needed to check UDP services like DNS
nmap -sV 192.168.1.1        # version detection, tries to figure out what software/version is running on open ports
nmap -O 192.168.1.1          # OS detection, guesses the target's operating system
```

### The one I use most

```
nmap -sV -sC -p- 192.168.1.1
```

`-sC` runs a set of default safe scripts alongside the scan (banner grabbing, basic vuln checks, that kind of thing), `-sV` grabs versions, `-p-` covers every port so nothing sneaky on a weird port gets missed. Slower than a quick scan but this is what I run when I actually want a full picture instead of a fast check.

### Output options

```
nmap -oN results.txt 192.168.1.1     # normal text output to a file
nmap -oX results.xml 192.168.1.1     # XML output, useful if feeding into another tool
```

---

## Burp Suite (web proxy)

Sits between your browser and the web app you're testing, so every request/response passes through it and you can inspect or modify it before it goes anywhere. This is the standard tool for actually testing web applications rather than just browsing them.

### Basic setup

1. Start Burp, it runs a proxy listener on `127.0.0.1:8080` by default
2. Point your browser at that proxy (either manually in browser settings, or easier, use FoxyProxy extension to toggle it on/off quickly)
3. Install Burp's CA certificate in your browser if you want to inspect HTTPS traffic too, otherwise you'll just get cert warnings on every site

### The tabs that actually matter day to day

- **Proxy > Intercept** - pauses each request before it's sent, lets you edit it live before forwarding. Good for tweaking a parameter on the fly and seeing what happens.
- **Proxy > HTTP History** - a log of every request/response that's passed through, without needing to intercept each one manually. This is where I spend most of my time, just scrolling back through what actually happened.
- **Repeater** - grab any request from history, send it to Repeater, and you can resend it over and over with small tweaks each time without the whole page reloading. This is how you test "what if I change this one parameter" type questions.
- **Intruder** - automates sending a request repeatedly with different payloads swapped into a chosen spot (like brute forcing a login field, or fuzzing a parameter with a wordlist). Rate limited in the free/Community edition, full speed in Pro.
- **Decoder** - quick encode/decode for base64, URL encoding, hex, etc, without needing to open a separate tool.

### Typical workflow

Browse the target app normally with intercept off so HTTP History fills up naturally → find an interesting request (login, a parameter that looks like it controls something) → send it to Repeater → start tweaking values and watching how responses change → if it's something like testing many values quickly, move it to Intruder instead.

---

## Netcat (network debugging)

The classic "Swiss army knife" of networking - reads and writes data across network connections, no frills. Good for quick tests when you don't want to spin up a whole other tool just to check if something's listening.

### Basic connection

```
nc example.com 80          # connect to a host on a port, like a stripped down telnet
```

Once connected, you can type raw protocol commands by hand, like typing an HTTP request manually:
```
GET / HTTP/1.1
Host: example.com

```
(hit enter twice after Host to send the blank line HTTP expects, then you'll see the raw response come back)

### Port scanning (basic, nmap does this better but nc can do quick checks)

```
nc -zv example.com 20-100
```

`-z` means don't actually send data, just check if the port's open, `-v` for verbose output so you actually see the results as it goes.

### Listening on a port

```
nc -lvp 4444
```

Starts netcat listening on port 4444, waiting for an incoming connection. Combine this with a connect from another machine and you've basically got a simple chat between two terminals - useful for confirming a firewall rule actually works, or testing if traffic between two hosts is getting through at all.

### File transfer

On the receiving machine:
```
nc -lvp 4444 > received_file.txt
```

On the sending machine:
```
nc target_ip 4444 < file_to_send.txt
```

No encryption, no authentication, this is purely for quick tests on a network you control, not something you'd use for anything sensitive.

### Banner grabbing

```
nc -v example.com 22
```

Connect to a port and see what the service announces about itself right away (SSH servers usually blurt out their version immediately on connect, for example). Quick way to check what's actually running on a port without a full nmap scan.

---

