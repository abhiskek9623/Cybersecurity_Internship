# Cryptography Notes (with OpenSSL hands-on)

Notes on symmetric vs asymmetric encryption, hashing, certificates/TLS, and some actual OpenSSL commands I ran through while learning this stuff. Tested these on a regular Linux box, should work basically anywhere OpenSSL is installed (check with `openssl version`).

## Symmetric vs Asymmetric Encryption

### Symmetric

Same key encrypts and decrypts. Fast, simple, but you've got a problem: how do you get that shared key to the other person in the first place without someone intercepting it along the way? That's the whole "key distribution problem."

Common algorithms: AES (the standard these days, AES-256 especially), also older ones like DES/3DES which are basically deprecated now.

Good for: encrypting large amounts of data quickly - files, disk encryption, bulk data. Bad for: sending the key to whoever needs to decrypt it, when you don't already have a secure channel.

### Asymmetric

Two keys - a public one and a private one. Public key encrypts, only the matching private key can decrypt (or the reverse, for signing - private key signs, public key verifies). You can hand your public key to literally anyone, doesn't matter if it gets intercepted, because it can't decrypt anything on its own.

Common algorithms: RSA, ECC (elliptic curve, newer and faster for equivalent security).

Good for: solving the key exchange problem, digital signatures, initial handshakes. Bad for: speed - it's way slower than symmetric, so nobody uses it to encrypt big files directly.

### Why both get used together

This is the part that actually clicked for me once I understood it - real systems (like TLS) use asymmetric encryption just to securely exchange a symmetric key, then switch to symmetric for the actual bulk data transfer. Best of both - asymmetric solves the "how do we agree on a secret without meeting in person" problem, symmetric handles the heavy lifting fast once that's sorted.

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared key | Public + private pair |
| Speed | Fast | Slow |
| Key exchange problem | Yes, this is the issue | No, public key can be shared openly |
| Common algos | AES, 3DES | RSA, ECC |
| Typical use | Bulk data encryption | Key exchange, signatures |

---

## Hashing (MD5, SHA256)

Hashing is one-way - you turn data into a fixed-length fingerprint, and there's no going backward from the hash to the original data. Not encryption, there's no key, no decrypting a hash back into the original message.

Used for: verifying a file hasn't been tampered with (download a file, check its hash matches what the site says it should be), storing passwords (never store the plaintext, store the hash), detecting duplicate files, etc.

**MD5** - 128-bit output. Considered broken for security purposes now, collisions (two different inputs producing the same hash) have been demonstrated. Still fine for basic "did this file get corrupted" checks, not fine for anything security related like password storage or certificates.

**SHA-256** - part of the SHA-2 family, 256-bit output, the standard for actual security use these days. This is what you want for anything that matters.

A couple things that are easy to mix up at first:
- Hashing is not encryption - you can't decrypt a hash, there's no key involved
- A tiny change in input completely changes the output hash (this is called the avalanche effect) - even flipping one character changes the whole hash, which is exactly what makes it useful for tamper detection

### Hands-on hashing with OpenSSL

```
echo -n "hello world" | openssl md5
echo -n "hello world" | openssl sha256
```

The `-n` matters - without it, echo adds a trailing newline and you'll get a different hash than you expect, and it'll be different from what other tools give you for "the same" input.

Hash a file instead of a string:
```
openssl md5 myfile.txt
openssl sha256 myfile.txt
```

This is basically what you're doing when you check a downloaded ISO's checksum against what the website published.

---

## Digital Certificates & SSL/TLS

### The problem certificates solve

Asymmetric encryption gives you a public/private key pair, but there's still an issue - how do you know a public key actually belongs to who it claims to belong to? Anyone can generate a keypair and claim to be your bank. Certificates solve this by having a trusted third party (a Certificate Authority, CA) vouch for the connection between a public key and an identity.

### What's actually in a certificate

- The public key
- Who it belongs to (domain name, organization)
- Who issued it (the CA)
- Validity dates
- The CA's digital signature over all of that, so you can verify it hasn't been tampered with

Your browser trusts a bunch of root CAs by default (this list ships with your OS/browser). When a site presents a cert, your browser checks the chain of trust back up to one of those roots.

### How the TLS handshake actually works (simplified)

1. Client says hello, lists what encryption it supports
2. Server responds with its certificate (contains its public key) and picks the encryption method
3. Client verifies the cert is legit (signed by a trusted CA, not expired, matches the domain)
4. Client and server use asymmetric crypto to agree on a shared symmetric key
5. From here on, everything's encrypted with that symmetric key, which is way faster than doing asymmetric for every single byte of the actual conversation

This is the "why both types of encryption get used together" thing from earlier, playing out in an actual real protocol.

### Hands-on: generating and looking at certs with OpenSSL

Generate a private key:
```
openssl genrsa -out mykey.pem 2048
```

Generate a self-signed certificate from that key (fine for testing/local dev, browsers will complain because no real CA signed it):
```
openssl req -new -x509 -key mykey.pem -out mycert.pem -days 365
```

It'll prompt you for country, organization, common name (this should be the domain if you're actually testing something) etc.

View what's inside a certificate:
```
openssl x509 -in mycert.pem -text -noout
```

Check a live site's certificate (useful for actually seeing expiry dates, issuer, etc on a real cert):
```
openssl s_client -connect example.com:443 -showcerts
```

---

## Hands-on: Encrypt and Decrypt with OpenSSL

### Symmetric encryption (AES)

Encrypt a file with a password:
```
openssl enc -aes-256-cbc -salt -in secret.txt -out secret.enc -pbkdf2
```

It'll prompt for a password (or you can pass `-k yourpassword` inline, though typing it in a command that gets saved to shell history isn't great practice).

Decrypt it back:
```
openssl enc -aes-256-cbc -d -in secret.enc -out secret_decrypted.txt -pbkdf2
```

`-pbkdf2` matters on newer OpenSSL versions - it's a stronger key derivation function, older guides you find online skip this flag and use a weaker default, worth using it explicitly.

### Asymmetric encryption (RSA)

Using the keypair from the certificate section above (or generate a fresh one):
```
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

Encrypt a short message with the public key:
```
openssl rsautl -encrypt -pubin -inkey public.pem -in message.txt -out message.enc
```

(Newer OpenSSL versions want `pkeyutl` instead of `rsautl`, which is technically deprecated - swap in `openssl pkeyutl -encrypt -pubin -inkey public.pem -in message.txt -out message.enc` if you get a deprecation warning.)

Decrypt with the private key:
```
openssl rsautl -decrypt -inkey private.pem -in message.enc -out message_decrypted.txt
```

Worth noting - RSA encryption like this only works on small messages (limited by key size), which is exactly why real-world systems use it to exchange a symmetric key rather than encrypting the actual data with it directly. Same theme as the TLS handshake above, just doing it by hand this time so it actually sinks in.

---

## Quick reference

| Task | Command |
|---|---|
| MD5 hash a string | `echo -n "text" \| openssl md5` |
| SHA256 hash a string | `echo -n "text" \| openssl sha256` |
| Hash a file | `openssl sha256 file.txt` |
| Generate RSA private key | `openssl genrsa -out key.pem 2048` |
| Extract public key | `openssl rsa -in key.pem -pubout -out pub.pem` |
| Self-signed cert | `openssl req -new -x509 -key key.pem -out cert.pem -days 365` |
| View a cert | `openssl x509 -in cert.pem -text -noout` |
| Check a live site's cert | `openssl s_client -connect domain.com:443` |
| AES encrypt a file | `openssl enc -aes-256-cbc -salt -pbkdf2 -in file -out file.enc` |
| AES decrypt a file | `openssl enc -aes-256-cbc -d -pbkdf2 -in file.enc -out file` |
| RSA encrypt small message | `openssl pkeyutl -encrypt -pubin -inkey pub.pem -in msg.txt -out msg.enc` |
| RSA decrypt | `openssl pkeyutl -decrypt -inkey key.pem -in msg.enc -out msg.txt` |

Next thing I want to dig into is actually setting up a local test server with the self-signed cert and watching the handshake happen in Wireshark instead of just reading about it.
