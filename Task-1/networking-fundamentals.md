# Networking Fundamentals Notes

Notes on the OSI model, TCP/IP, DNS/HTTP, and IP addressing/subnetting/NAT. These are the concepts that never quite stick until you've explained them to someone else a few times, so writing it out here too.

## The OSI Model

Seven layers, describes how data moves from one machine to another conceptually. Nobody actually implements networking exactly along these lines anymore (that's more what TCP/IP does in practice) but it's still the mental model everyone uses to talk about "which layer" a problem is happening at.

Bottom to top:

**Layer 1 - Physical**
The actual wires, radio waves, light pulses. Cables, network cards, hubs. This layer doesn't know what a "packet" is, it just moves raw bits.

**Layer 2 - Data Link**
Handles communication between devices on the same local network. This is where MAC addresses live. Switches operate here. Ethernet and Wi-Fi framing happen at this layer.

**Layer 3 - Network**
This is where IP addresses and routing come in. Figures out how to get a packet from one network to another. Routers live here.

**Layer 4 - Transport**
TCP and UDP. This is where you get reliable delivery (TCP, with retransmission and ordering) vs fast-but-unreliable delivery (UDP, no guarantees). Ports also belong to this layer.

**Layer 5 - Session**
Manages the connection/session between two applications - opening, maintaining, closing it. Honestly this layer gets glossed over a lot in real-world discussions, it kind of blends into layers 4 and 6.

**Layer 6 - Presentation**
Formats/translates data so the application layer can use it - encryption, compression, character encoding. SSL/TLS is sometimes described as living here, though in practice people usually just say "layer 4-7ish" for TLS.

**Layer 7 - Application**
The layer actual applications talk to. HTTP, FTP, SMTP, DNS - all layer 7 protocols. This is the layer most people interact with without ever thinking about the six below it.

Mnemonic people use: "All People Seem To Need Data Processing" (physical to application) or the reverse "Please Do Not Throw Sausage Pizza Away" (application down to physical). Pick whichever sticks.

Why this matters practically - when someone says "is this a layer 2 or layer 3 issue," they're basically asking "is this a local network problem or a routing problem." Framing it that way helped it click for me more than memorizing the layer names ever did.

---

## TCP/IP Protocol Suite

This is what actually runs the internet, as opposed to OSI which is more of a teaching model. TCP/IP is usually described in 4 layers instead of 7, and they roughly map onto OSI like this:

- Network Access (maps to OSI layers 1-2)
- Internet (maps to OSI layer 3) - this is where IP lives
- Transport (maps to OSI layer 4) - TCP and UDP
- Application (maps to OSI layers 5-7) - HTTP, DNS, SMTP, etc, all lumped together

**TCP (Transmission Control Protocol)** - connection-oriented, reliable. Does a three-way handshake before sending data (SYN, SYN-ACK, ACK), numbers every segment so it can detect missing/out-of-order data, retransmits what got lost. Used for anything where you need the data intact - web pages, file transfers, email.

**UDP (User Datagram Protocol)** - connectionless, no handshake, no guarantee of delivery or order, just fire and forget. Faster because of all that skipped overhead. Used where speed matters more than perfection - video calls, DNS lookups, online gaming, live streaming. If a packet gets dropped in a video call you just get a glitch for a frame, nobody wants the call to pause and wait for a retransmit.

Quick comparison:

| | TCP | UDP |
|---|---|---|
| Connection | Yes, handshake first | No handshake |
| Reliability | Guaranteed delivery, retransmits | No guarantee |
| Order | Preserved | Not preserved |
| Speed | Slower (overhead) | Faster |
| Use case | Web, email, file transfer | Video/voice, DNS, gaming |

---

## DNS & HTTP/HTTPS

### DNS

Translates domain names into IP addresses, because humans remember "google.com" way better than an IP address.

Rough flow when you type a URL:
1. Browser checks its own cache first
2. If not cached, asks the OS, which asks a configured DNS resolver (often your ISP's, or something like 8.8.8.8)
3. Resolver checks root servers, then TLD servers (.com, .org etc), then the authoritative nameserver for that specific domain
4. Authoritative server returns the actual IP
5. Answer gets cached along the way so next time is faster

Common record types worth knowing:
- **A record** - maps a domain to an IPv4 address
- **AAAA record** - same idea but IPv6
- **CNAME** - alias, points one domain to another domain name instead of directly to an IP
- **MX** - mail server for the domain
- **TXT** - arbitrary text, often used for verification or SPF/DKIM email auth stuff
- **NS** - which nameservers are authoritative for the domain

Handy commands for poking at DNS:
```
nslookup example.com
dig example.com
dig example.com MX
```

### HTTP / HTTPS

HTTP is the protocol browsers and servers use to talk to each other - request/response based. You ask for a page, server responds with the content (or an error).

Common methods:
- GET - retrieve data
- POST - submit data (like a form)
- PUT - update/replace a resource
- DELETE - remove a resource
- PATCH - partial update

Status codes worth actually remembering instead of looking up every time:
- 200 - OK, all good
- 301/302 - redirect (permanent vs temporary)
- 400 - bad request, client sent something malformed
- 401 - unauthorized, you need to log in
- 403 - forbidden, you're logged in but not allowed
- 404 - not found
- 500 - server error, something broke on their end
- 503 - service unavailable, server's overloaded or down for maintenance

**HTTPS** is just HTTP wrapped in TLS encryption. The handshake at the start negotiates encryption keys so everything after that is encrypted in transit - protects against anyone snooping on the connection or tampering with it in the middle. Look for the padlock icon, though obviously HTTPS just means the connection is encrypted, it says nothing about whether the site itself is trustworthy.

Port-wise: HTTP defaults to port 80, HTTPS defaults to port 443.

---

## IP Addressing, Subnetting, and NAT

### IP Addressing

IPv4 addresses are 32 bits, written as four numbers 0-255 separated by dots, like `192.168.1.10`. IPv6 exists because IPv4 basically ran out of addresses, way longer format like `2001:0db8:85a3::8a2e:0370:7334`, still not fully adopted everywhere.

Addresses split into public (routable on the actual internet) and private (only meaningful inside a local network). The private ranges you'll see constantly:
```
10.0.0.0 - 10.255.255.255
172.16.0.0 - 172.31.255.255
192.168.0.0 - 192.168.255.255
```

If you see an address in one of those ranges, it's a local/internal address, not something reachable directly from the internet.

### Subnetting

Splitting a network into smaller sub-networks. The subnet mask (like `255.255.255.0`, often written as `/24`) tells you which part of the address is the network portion vs the host portion.

`/24` means the first 24 bits are the network, leaving 8 bits for hosts - that's 256 addresses (254 usable, since the first is the network address and the last is the broadcast address).

Quick reference for common ones:
```
/24 = 255.255.255.0     = 254 usable hosts
/25 = 255.255.255.128   = 126 usable hosts
/26 = 255.255.255.192   = 62 usable hosts
/27 = 255.255.255.224   = 30 usable hosts
```

Why bother subnetting instead of just using one big flat network - it isolates traffic (broadcast traffic doesn't flood the whole company), it's a security boundary (you can firewall between subnets), and it just organizes things logically (finance on one subnet, engineering on another, etc).

### NAT (Network Address Translation)

This is the thing that lets your whole home network share one public IP address. Your router does NAT - internally your laptop, phone, etc all have private addresses like `192.168.1.x`, but to the outside world, all traffic looks like it's coming from your router's single public IP.

Router keeps a translation table so when a response comes back, it knows which internal device to actually send it to.

Main reason NAT exists in the first place is IPv4 address scarcity - way fewer public IPs exist than devices that need internet access, so NAT lets many devices share one public address. Side effect is it also acts as a rough security layer since nothing from outside can initiate a connection to an internal device unless you specifically set up port forwarding for it.

---

## Quick reference

| Concept | Key thing to remember |
|---|---|
| OSI Layer 3 | IP addressing, routing |
| OSI Layer 4 | TCP/UDP, ports |
| TCP | Reliable, handshake, ordered |
| UDP | Fast, no guarantees |
| DNS | Translates names to IPs, A/CNAME/MX records |
| HTTP | Port 80, unencrypted |
| HTTPS | Port 443, TLS encrypted |
| Private IP ranges | 10.x, 172.16-31.x, 192.168.x |
| /24 subnet | 254 usable host addresses |
| NAT | Many private IPs share one public IP |

More to add once I get further into routing protocols and VLANs.
