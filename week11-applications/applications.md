# EC 441 — Client/Server & P2P Application Architecture
## Detailed Homework Problems + Full Solutions

---

## Problem 1: The Core Distinction — What Makes Something Client/Server vs. P2P?

### Problem Statement

**(a)** Define the **client/server model** precisely. What are the three defining characteristics of a server? What are the three defining characteristics of a client?

**(b)** Define the **peer-to-peer (P2P) model**. How does it differ fundamentally from client/server? What is a "peer," and what roles can a peer play simultaneously?

**(c)** Classify each of the following as client/server, P2P, or **hybrid** (and justify your answer):
1. A student loads `http://bu.edu` in their browser
2. Two students video call each other on Skype
3. A user downloads a torrent file using BitTorrent
4. A user asks Siri a voice question (processed in Apple's data center)
5. A multiplayer video game where players see each other's positions in real time
6. DNS — a client asks a resolver, which then asks authoritative servers

**(d)** The client/server model has **one fundamental scaling problem**. Describe it precisely in terms of server upload capacity, number of clients, and file size. Write the formula for the minimum time to distribute a file of size F to N clients if the server has upload bandwidth u_s and each client has download bandwidth d_i.

---

### Solution

**(a) Client/Server definitions:**

A **server** has three defining characteristics:
1. **Always-on** — it runs continuously, waiting for requests. It is not started on demand.
2. **Permanent, known address** — it has a fixed, publicly reachable IP address (or hostname) so clients can find it.
3. **Passive role** — it never initiates contact; it only responds to clients.

A **client** has three defining characteristics:
1. **Initiates contact** — the client always sends the first message to begin an interaction.
2. **Intermittent connectivity** — clients connect and disconnect; they do not need to be always-on.
3. **Dynamic/private addresses acceptable** — clients can use ephemeral or NAT-translated addresses because the server never needs to initiate a connection to them.

Think of it like a restaurant (server) vs. a customer (client): the restaurant is always open at a known address, and the customer walks in and places an order — never the reverse.

**(b) Peer-to-Peer definition:**

In a **P2P model**, there is no dedicated always-on server. Instead, arbitrary end hosts called **peers** communicate directly with each other. A peer is simultaneously a **client** (it requests data from others) and a **server** (it supplies data to others). Peers are intermittently connected and may have dynamic IP addresses.

The fundamental difference from client/server: **the capacity of the system grows as more peers join**, because each new peer brings both demand *and* supply (its upload bandwidth). In client/server, adding more clients adds only demand — the server's upload bandwidth is fixed and becomes the bottleneck.

**(c) Classifications:**

1. **`http://bu.edu` in a browser** → **Client/Server.** The browser is the client (initiates TCP SYN to port 80/443); BU's web server is the always-on server with a fixed address. Classic client/server.

2. **Skype video call** → **Hybrid.** Skype uses a central server for *signaling* (finding the other user, exchanging addresses) but then attempts *direct peer-to-peer media streaming* between the two users for the actual video/audio. If a direct P2P connection fails (both behind NAT), Skype routes media through Microsoft's relay servers — back to client/server for the data path.

3. **BitTorrent** → **Hybrid.** A central **tracker** server (or distributed hash table in trackerless torrents) helps peers find each other — that part is client/server. But the actual file data flows directly **peer-to-peer**: peers upload pieces to each other simultaneously. Pure P2P for data transfer.

4. **Siri voice question** → **Client/Server.** The phone is the client; Apple's data centers process the audio and return the answer. You can't process a Siri query without Apple's servers. No P2P component.

5. **Multiplayer game** → **Hybrid** (often). Many games use a **dedicated server** (client/server) for authoritative game state, while some use P2P for low-latency position updates between nearby players. Modern games like Fortnite run on cloud servers (pure client/server); older games like early Call of Duty used P2P lobbies.

6. **DNS** → **Client/Server at every step.** Your computer is the client to a recursive resolver. The resolver is a client to root nameservers, then to TLD nameservers, then to authoritative nameservers. Each is a distinct client/server transaction. DNS is hierarchical client/server, not P2P.

**(d) Client/Server scaling bottleneck:**

The server must upload the file to every client individually. With N clients each needing file of size F bits, the server must transmit N×F bits total. If server upload bandwidth is u_s, this takes at minimum:

**D_cs ≥ NF / u_s** (server is the bottleneck)

But clients also have download limits. Each client i downloads at rate d_i; the slowest client limits how fast that copy completes:

**D_cs ≥ F / min(d_i)** (slowest client is the bottleneck)

The minimum distribution time for client/server is:

> **D_cs = max( NF/u_s , F/min(d_i) )**

The first term **grows linearly with N** — double the clients, double the time. The server upload bandwidth u_s is fixed and becomes the hard ceiling on throughput. This is the core scaling problem: the server is a single point of congestion regardless of how many peers join.

---

## Problem 2: P2P Distribution Time Analysis

### Problem Statement

Consider distributing a file of **F = 8 Gb** (gigabits) to a swarm of peers. The server (or initial seeder) has upload bandwidth **u_s = 30 Mbps**. Each peer has upload bandwidth **u_i = 5 Mbps** and download bandwidth **d_i = 10 Mbps** for all peers.

**(a)** Write the formula for minimum P2P file distribution time **D_P2P** for N peers. (Note: in P2P, every peer who receives a piece can immediately re-upload it to others.)

**(b)** Compute D_cs (client/server) and D_P2P for N = 10, 100, and 1000 peers. Show your work. Put results in a table.

**(c)** At what value of N does D_P2P first *stop* increasing with N? (i.e., when is the total upload capacity no longer the bottleneck?) Explain intuitively why D_P2P is bounded even as N → ∞.

**(d)** In practice, BitTorrent **never achieves** the theoretical minimum D_P2P. Give two real-world reasons why.

**(e)** The formula assumes every peer's upload bandwidth is fully available for the swarm. Why might this be unrealistic? What protocol mechanism does BitTorrent use to incentivize peers to actually upload?

---

### Solution

**(a) P2P minimum distribution time formula:**

In P2P, the total upload capacity available to the swarm is the server's bandwidth plus all peers' bandwidth: **u_total = u_s + N·u_i**.

Three constraints must all be satisfied:
1. The server must get the file "into" the swarm at least once: **F / u_s**
2. Every peer must download the complete file: **F / d_min**
3. Total bits uploaded must cover N copies of F, and total upload rate is (u_s + N·u_i): **NF / (u_s + N·u_i)**

> **D_P2P = max( F/u_s , F/d_min , NF/(u_s + N·u_i) )**

**(b) Numerical comparison:**

Given: F = 8,000 Mb, u_s = 30 Mbps, u_i = 5 Mbps, d_min = 10 Mbps

**Fixed terms (independent of N):**
- F/u_s = 8000/30 = **266.7 min**
- F/d_min = 8000/10 = **800 min**

Wait — F/d_min = 800 min > F/u_s = 266.7 min, so the download constraint dominates the first two terms for all N.

**Third term (varies with N):**
- NF/(u_s + N·u_i) = N×8000 / (30 + 5N)

| N | D_cs = max(NF/u_s, F/d_min) | Third term NF/(u_s+N·u_i) | D_P2P = max(800, F/d_min, third) |
|---|---|---|---|
| 10 | max(2667, 800) = **2,667 min** | 10×8000/(30+50) = 80,000/80 = **1,000 min** | max(266.7, 800, 1000) = **1,000 min** |
| 100 | max(26,667, 800) = **26,667 min** | 100×8000/(30+500) = 800,000/530 = **1,509 min** | max(266.7, 800, 1509) = **1,509 min** |
| 1000 | max(266,667, 800) = **266,667 min** | 1000×8000/(30+5000) = 8,000,000/5030 = **1,590 min** | max(266.7, 800, 1590) = **1,590 min** |

**Summary table:**

| N peers | D_cs (min) | D_P2P (min) | Speedup |
|---------|------------|-------------|---------|
| 10 | 2,667 | 1,000 | 2.7× |
| 100 | 26,667 | 1,509 | 17.7× |
| 1,000 | 266,667 | 1,590 | 167.7× |

The client/server time grows linearly with N (catastrophic); P2P time grows slowly and nearly flattens.

**(c) When does D_P2P stop increasing?**

The third term NF/(u_s + N·u_i) approaches F/u_i = 8000/5 = **1,600 min** as N → ∞:

```
lim (N→∞) of NF/(u_s + N·u_i) = NF/(N·u_i) = F/u_i = 1,600 min
```

Since F/d_min = 800 min < 1,600 min, the D_P2P converges to **1,600 min** as the asymptote.

**Why it's bounded:** As N grows, each new peer brings u_i = 5 Mbps of *additional* upload capacity to the swarm. The total upload capacity grows proportionally with N (u_total ≈ N·u_i for large N), and the total demand also grows proportionally with N (need to deliver N copies). These cancel out — the ratio NF/(N·u_i) = F/u_i is constant. Every new peer demands exactly as much as it contributes. The system is self-scaling.

Think of it like a potluck dinner: each new guest brings exactly one dish and eats exactly one dish. No matter how many guests come, there's always enough food.

**(d) Why BitTorrent doesn't achieve theoretical minimum:**

1. **Rarest-first is imperfect at small swarm sizes.** The theoretical minimum assumes every peer can immediately get any piece it needs. In practice, early in a download, only a few peers have each piece. Pieces propagate in a tree-like fashion — the root (seeder) is still a bottleneck for rare pieces even if the total bandwidth math works out.

2. **Overhead and coordination cost.** Peers spend bandwidth on tracker communications, piece announcements, handshakes, and metadata — not just payload data. The protocol headers and coordination messages consume bandwidth that the formula treats as fully available for file data.

**(e) Free-riding and BitTorrent's incentive mechanism:**

Peers are not obligated to upload — they could download pieces and refuse to share, "free-riding" on the swarm. This is the classic tragedy of the commons applied to P2P.

BitTorrent uses **tit-for-tat (choking/unchoking)**:
- Each peer simultaneously downloads from multiple others. It **prioritizes uploading to peers that are also uploading to it** ("unchoking" them).
- Peers that don't upload get **choked** — their download rate is throttled or cut off.
- Periodically, a peer "optimistically unchokes" a random peer to discover potentially better uploaders.

The result: peers that upload fast *receive* fast, creating a direct incentive to contribute upload bandwidth. Free-riders get slow download speeds from peers who notice they contribute nothing in return.

---

## Problem 3: NAT and P2P — Why Direct Connections Are Hard

### Problem Statement

Alice (behind NAT router A, public IP 203.0.113.10) and Bob (behind NAT router B, public IP 198.51.100.20) both want to video call each other using a P2P application. Their private addresses are Alice: 10.0.0.5 and Bob: 192.168.1.7.

**(a)** Explain precisely why neither Alice nor Bob can simply initiate a TCP connection directly to the other. What does each NAT router do to block this?

**(b)** Describe the **STUN** (Session Traversal Utilities for NAT) technique. What is a STUN server's role? After STUN, what address does each side learn?

**(c)** What is **UDP hole punching**? Walk through the exact sequence of steps Alice and Bob's clients would take to establish a direct UDP session. Why does it work even though each router initially blocks incoming traffic?

**(d)** When hole punching fails (e.g., symmetric NATs), a **TURN relay** server is used. Describe how TURN works. Does this maintain the P2P property of the connection? Why or why not?

**(e)** How does **IPv6** eliminate this entire problem? Explain in one sentence why P2P is fundamentally simpler in an IPv6-native world.

---

### Solution

**(a) Why direct TCP fails:**

NAT routers only maintain translation table entries for **outbound-initiated connections**. When Alice's device at 10.0.0.5 sends a packet to the internet, Router A creates an entry:

```
(203.0.113.10 : port 40001) ↔ (10.0.0.5 : port 5000)
```

But if Bob tries to initiate an inbound TCP SYN to `203.0.113.10`, Router A has **no entry for that destination port** in its translation table — it doesn't know which internal host (Alice, her phone, her laptop, her TV) should receive it. Router A **drops the packet silently**.

The same problem applies in reverse: if Alice tries to connect to `198.51.100.20`, Router B has no entry for an inbound connection to Bob. Both are stuck. Neither can play the "server" role because neither has a publicly reachable address. This is the core P2P-under-NAT problem that Lecture 17 identifies: "Two hosts both behind NAT cannot directly communicate."

**(b) STUN — discovering your public address:**

STUN works like looking in a mirror you don't own. A **STUN server** is a well-known always-on server with a public IP. Here's what happens:

1. Alice's device sends a UDP packet from `10.0.0.5:5000` to the STUN server.
2. Router A translates this: the STUN server sees the packet arriving from `203.0.113.10:40001` (the NAT-translated address).
3. The STUN server **replies** to Alice with: "Your public address is `203.0.113.10:40001`."
4. Alice now knows her external IP:port as seen by the internet.
5. Bob does the same: he learns his external address is `198.51.100.20:55321`.

Both sides share their STUN-discovered addresses via a **signaling server** (a separate, always-on server both can reach). Now each side knows the other's public IP:port. But knowing the address isn't enough — they still can't initiate incoming connections due to NAT filtering.

**(c) UDP Hole Punching — step by step:**

The key insight: when a NAT router sees an *outbound* UDP packet, it creates a translation entry and will accept *inbound* UDP replies from that destination. If both sides "shoot first," both routers open holes simultaneously.

**Step-by-step:**

1. Alice's client and Bob's client both connect to a **signaling server** (regular client/server connection each can initiate outbound).
2. Via signaling, Alice learns Bob's public address `198.51.100.20:55321`, and Bob learns Alice's `203.0.113.10:40001`.
3. **Simultaneously** (coordinated by the signaling server):
   - Alice sends a UDP packet to `198.51.100.20:55321`. Router A creates an outbound entry: traffic from Alice going to Bob is allowed. Router B likely drops this first packet (no entry yet for inbound from Alice), but the important thing is **Router A now has an entry**.
   - Bob sends a UDP packet to `203.0.113.10:40001`. Router B creates an outbound entry. Router A may have *already created* the entry from Alice's outbound packet — so this packet from Bob **passes through Router A** to Alice!
4. Once the first packet gets through in either direction, both NATs have entries and bidirectional UDP flows.

**Why it works:** Each router only blocks traffic *that has no corresponding outbound entry*. By sending outbound packets simultaneously, each peer "punches a hole" in its own NAT — creating the table entry that allows the peer's incoming response to pass through.

**(d) TURN relay — fallback when hole punching fails:**

**Symmetric NAT** is more aggressive: it creates a *different* external port for each different destination. So Alice's port to the STUN server (40001) differs from her port to Bob (40002). Bob's hole-punch attempt to 40001 fails because Router A maps Alice→Bob traffic differently. Hole punching fails.

**TURN (Traversal Using Relays around NAT):**
1. Both Alice and Bob connect to a TURN relay server (outbound connections both can initiate).
2. The TURN server allocates a relay address for each.
3. All media/data is sent to the TURN server, which forwards it to the other party.

**Does this maintain the P2P property?** **No — not really.** The data path is now:

```
Alice → TURN Server → Bob
```

This is functionally client/server for the data plane. The TURN server must have sufficient bandwidth to handle both sides. The only "P2P" aspect remaining is that Alice and Bob pay for the call rather than a dedicated media server — but architecturally, the TURN relay is the server. WebRTC (used in browser video calls) uses ICE (Interactive Connectivity Establishment), which tries hole punching first and falls back to TURN if needed.

**(e) IPv6 eliminates the problem:**

In IPv6, every device gets a **globally unique, publicly routable address** — there is no NAT, no address translation, and therefore no NAT traversal problem. Alice's device at `2001:db8::1` is directly reachable from Bob's device at `2001:db8::2` with no intermediaries. P2P becomes straightforward: both peers simply connect to each other's global addresses directly.

---

## Problem 4: HTTP — Client/Server at the Application Layer

### Problem Statement

A web client (browser) at `192.168.1.10` connects to `http://www.example.com/index.html`. The DNS lookup has already resolved `www.example.com` to `93.184.216.34`. The client and server are using **HTTP/1.1** (non-persistent vs. persistent connections).

**(a)** Describe the **complete sequence of events** from the moment the user presses Enter to the moment the page starts rendering. Include: TCP handshake, HTTP request format, HTTP response, and TCP teardown. How many RTTs does this take total?

**(b)** HTTP/1.0 uses **non-persistent connections**: one TCP connection per object. HTTP/1.1 uses **persistent connections** (keep-alive). If the page has 1 HTML file + 10 embedded images, how many TCP connections does each version open? How many total RTTs to load all 11 objects (ignoring transmission time)?

**(c)** HTTP is a **request-response** protocol. Who always initiates? Can a server push data to a client without a request in HTTP/1.1? How does HTTP/2 change this?

**(d)** HTTP is **stateless** — the server keeps no memory of past requests. This is a deliberate design choice. Give one advantage and one disadvantage of statelessness for web applications. How do web apps maintain state (e.g., login sessions) despite a stateless protocol?

**(e)** HTTP uses TCP (port 80) or TLS over TCP (port 443 for HTTPS). Why not UDP? What specific TCP properties does HTTP depend on?

---

### Solution

**(a) Complete sequence of events:**

1. **DNS is already resolved** → client knows destination IP `93.184.216.34`

2. **TCP Three-Way Handshake** (1 RTT):
   - Client sends SYN to `93.184.216.34:80`
   - Server replies SYN-ACK
   - Client sends ACK
   - *RTT count: 1*

3. **HTTP GET Request** (client sends immediately after ACK in HTTP/1.1):
   ```
   GET /index.html HTTP/1.1
   Host: www.example.com
   Connection: keep-alive
   User-Agent: Mozilla/5.0 ...
   Accept: text/html
   [blank line]
   ```

4. **Server processes and responds** (1 RTT for the response to travel back):
   ```
   HTTP/1.1 200 OK
   Content-Type: text/html
   Content-Length: 2048
   [blank line]
   [HTML body...]
   ```
   - *RTT count: 2 total (1 handshake + 1 request/response)*

5. **Browser begins rendering** as HTML arrives (streaming, not waiting for full body).

6. **TCP stays open** (HTTP/1.1 persistent) for subsequent requests.

7. **TCP teardown** (FIN/ACK exchange) happens after the server or client closes the connection, typically after a timeout.

**Total: 2 RTTs before first byte of content arrives** (1 for handshake, 1 for HTTP request/response).

*Note: If DNS is not cached, add 1+ RTTs for DNS resolution before step 2.*

**(b) Non-persistent vs. persistent connections:**

**HTTP/1.0 (non-persistent):**
- Each object requires its own TCP connection: 3-way handshake + request + response + teardown
- 11 objects → **11 TCP connections**
- Each connection costs 2 RTTs (1 handshake + 1 request)
- With **sequential** loading: 11 × 2 = **22 RTTs**
- With **parallel** connections (browsers open ~6 at once): approximately 2×ceil(11/6) ≈ **6 RTTs**

**HTTP/1.1 (persistent, pipelined):**
- 1 TCP connection for all objects (1 handshake = 1 RTT)
- With pipelining: all requests sent back-to-back without waiting for responses
- 1 RTT for handshake + 1 RTT for first HTML + 1 RTT for all 10 images pipelined = approximately **3 RTTs**
- Without pipelining (sequential requests on same connection): 1 + 11 = **12 RTTs**

**Summary:**

| Mode | TCP Connections | Approximate RTTs |
|------|----------------|-----------------|
| HTTP/1.0 non-persistent, sequential | 11 | 22 |
| HTTP/1.0 non-persistent, parallel (6) | 11 | ~6 |
| HTTP/1.1 persistent, sequential | 1 | 12 |
| HTTP/1.1 persistent, pipelined | 1 | ~3 |

**(c) Request-response and server push:**

In HTTP/1.1, the **client always initiates**. The server cannot push data proactively — it can only respond to requests. If new data becomes available on the server (e.g., a new tweet, stock price), the client must poll repeatedly: "anything new? anything new?" This is wasteful.

**HTTP/2** introduced **server push**: the server can speculatively send resources it *predicts* the client will need, before the client requests them. For example, when serving `index.html`, the server can push `style.css` and `logo.png` immediately, saving RTTs for those requests.

HTTP/3 (QUIC) keeps server push but the mechanism is built into QUIC streams.

**(d) Statelessness — tradeoffs:**

**Advantage:** Each request is independent — servers can be **horizontally scaled** with no coordination. Any server in a data center can handle any request from any client because no session state lives on the server. This is why websites like Google and Facebook can run on hundreds of thousands of servers without "sticky sessions."

**Disadvantage:** Applications that need to track user context (login, shopping cart, preferences) must rebuild that context on every request. This adds overhead.

**How state is maintained despite statelessness:**

1. **Cookies** — the server sends a `Set-Cookie` header with a session identifier. The browser stores it and sends it back with every subsequent request via the `Cookie` header. The server looks up the session ID in a database. The *cookie* travels with each request; the *session data* lives on the server (or in a distributed cache like Redis).

2. **Tokens (JWT)** — the session data itself is encoded, signed, and sent to the client. The client sends it back with each request. The server verifies the signature without needing a session database.

State lives at the **endpoints** (client stores the cookie/token; server stores/verifies session data), not *in the protocol* — consistent with the end-to-end principle.

**(e) Why HTTP uses TCP, not UDP:**

HTTP **requires** the properties that TCP provides:

1. **Reliable, ordered delivery** — an HTML page missing bytes or received out of order would be unparseable. CSS or JavaScript files with missing segments would break page rendering.
2. **Flow control (rwnd)** — the server should not overwhelm a slow client's buffer.
3. **Congestion control (cwnd)** — HTTP transfers should back off when the network is congested, sharing bandwidth fairly with other flows.
4. **Byte-stream semantics** — HTTP can send arbitrarily large responses (a video file, a large HTML page) and TCP handles segmentation transparently.

UDP would require HTTP to re-implement all of these — which is essentially what QUIC does (HTTP/3 uses QUIC over UDP, reimplementing reliability and congestion control). For standard HTTP/1.1 and HTTP/2, TCP is the correct and natural choice.

---

## Problem 5: DNS — A Critical Client/Server Infrastructure

### Problem Statement

DNS (Domain Name System) is the phone book of the internet — it translates human-readable names to IP addresses. It is a **hierarchical, distributed client/server system**.

**(a)** Describe the 4-level DNS hierarchy: root servers, TLD servers, authoritative servers, and local resolvers. What does each level know? Draw the query chain for resolving `www.cs.bu.edu`.

**(b)** DNS uses **iterative** and **recursive** resolution. Describe both. In iterative resolution, who does the work? In recursive, who does the work?

**(c)** DNS primarily uses **UDP port 53**, not TCP. Explain why UDP is appropriate here. Under what conditions does DNS fall back to TCP, and why?

**(d)** DNS has a **caching** mechanism at each resolver. Explain how **TTL (Time to Live)** in DNS records controls caching. If `bu.edu`'s IT team wants to change their server's IP address, what TTL strategy should they use *before* and *after* the change, and why?

**(e)** DNS is fundamental infrastructure — it is a dependency for almost all internet traffic. Describe the **Pakistan Telecom / YouTube incident (2008)** from Lecture 16 in terms of the DNS/BGP interaction. How does a more-specific BGP prefix route interact with DNS to cause an internet-wide outage? (Hint: connect what you know about longest-prefix match.)

---

### Solution

**(a) The DNS Hierarchy for `www.cs.bu.edu`:**

```
Client (your laptop)
     ↓ (1) "Who is www.cs.bu.edu?"
Local Resolver (e.g., 8.8.8.8 — Google's DNS, or your ISP's resolver)
     ↓ (2) ask Root Server
Root Server (.): "I don't know, but .edu TLD servers are at these IPs"
     ↓ (3) ask .edu TLD Server
.edu TLD Server: "I don't know www.cs.bu.edu, but bu.edu's authoritative server is at 128.197.x.x"
     ↓ (4) ask bu.edu Authoritative Server
bu.edu Authoritative: "www.cs.bu.edu → 128.197.10.20" ← final answer
     ↓ (5) resolver returns answer to client
Client uses 128.197.10.20
```

**What each level knows:**
- **Root servers (13 logical clusters, ~1000 physical):** Know the addresses of all TLD servers (.com, .edu, .org, .uk, etc.). They know *nothing* about individual domains.
- **TLD servers (.edu, .com, etc.):** Know which nameservers are *authoritative* for each second-level domain (bu.edu, mit.edu, etc.). Don't know individual hostnames.
- **Authoritative servers (e.g., bu.edu's nameserver):** Know the actual IP address records (A records, AAAA records) for every hostname within their domain.
- **Local resolver (recursive resolver):** Caches answers to avoid re-querying the hierarchy. Does the leg work on behalf of clients.

**(b) Iterative vs. Recursive resolution:**

**Iterative resolution** — the resolver does all the work:
- Client asks resolver: "What is www.cs.bu.edu?"
- Resolver asks root: "I don't know, ask .edu TLD at X"
- Resolver asks .edu TLD: "I don't know, ask bu.edu's nameserver at Y"
- Resolver asks bu.edu: "It's 128.197.10.20"
- Resolver returns the answer to the client
- The client just asks once and waits — all the iteration happens inside the resolver

**Recursive resolution** — each server in the chain does the next query itself:
- Client asks resolver → resolver asks root → root asks .edu TLD → .edu asks bu.edu → answer propagates back up the chain to the client
- Each server in the chain is responsible for querying the next level
- Less common in practice; typically only the **local resolver** is recursive (it queries on behalf of clients); interactions between resolvers and authoritative servers are iterative

In practice: **client → resolver is recursive** (resolver does all the work); **resolver → authoritative servers is iterative** (resolver contacts each level itself).

**(c) Why DNS uses UDP:**

DNS is a **query-response** protocol: one small request, one small response. TCP's three-way handshake adds at minimum 1 RTT of overhead *before* any data flows. For a DNS lookup that completes in 1 UDP round-trip (request + response), TCP would at least double the latency. Since DNS lookups happen before almost every web request, this overhead would be felt everywhere.

UDP is appropriate because:
- DNS messages are typically small (< 512 bytes historically, < 4096 bytes with EDNS0)
- If a UDP response is lost, the resolver simply retransmits — a simpler retry mechanism than TCP teardown + reconnect
- No state needs to be maintained between queries

**DNS falls back to TCP when:**
1. Response exceeds the UDP payload limit (the server sets the TC "truncation" bit, signaling retry over TCP)
2. **Zone transfers (AXFR)** — copying an entire DNS zone between servers requires TCP for reliability and ordering
3. **DNSSEC responses** — cryptographic signatures make responses much larger, often exceeding UDP limits

**(d) TTL strategy for IP address changes:**

DNS records carry a **TTL in seconds** — the duration for which resolvers may cache the answer without re-querying. Once cached, a resolver uses the cached value even if the authoritative server has updated it, until the TTL expires.

**Before the change (days/weeks ahead):** Lower the TTL dramatically — from a typical value like 86400 seconds (24 hours) down to **300 seconds (5 minutes)**. Wait at least one full original-TTL period so all caches expire the old high-TTL value. Now all resolvers worldwide will re-query frequently.

**During the change:** Update the DNS record to the new IP. The low TTL means all resolvers will pick up the new address within 5 minutes.

**After the change (once stable):** Raise the TTL back to 86400 seconds (24 hours) to reduce DNS query load and improve performance.

**Why this matters:** If you change the IP with TTL=86400 still in effect, some clients will still be directed to the old server for up to 24 hours after the change — their resolver cached the old answer and won't re-query until TTL expires. Lowering TTL first "drains" the cache.

**(e) Pakistan Telecom / YouTube BGP-DNS interaction:**

In 2008, Pakistan Telecom announced to the global BGP routing table a more-specific prefix for YouTube's IP space: a **/24** covering part of YouTube's address space, while YouTube's legitimate prefix was a **/22**.

**How longest-prefix match causes the problem:** When a router has both entries:
- YouTube's legitimate: `208.65.153.0/22` (covers 1024 addresses)
- Pakistan Telecom's hijack: `208.65.153.0/24` (covers only 256 addresses)

Longest-prefix match means routers **always prefer the more specific /24** over the /22. So any traffic destined for those 256 IP addresses gets routed toward Pakistan Telecom instead of YouTube.

**The DNS connection:** DNS still returned YouTube's real IP addresses correctly — DNS itself was not compromised. But when your browser used that IP address to open a TCP connection, the *routing layer* sent your packets to Pakistan Telecom (a black hole — they weren't serving YouTube content). Your connection timed out. DNS said "go to 208.65.153.128" — a true answer — but the route to that IP was hijacked.

This illustrates a critical point: **DNS and IP routing are separate systems**. DNS correctness does not guarantee connectivity. The Facebook 2021 outage was the inverse: BGP routes for Facebook's network were correctly advertised, but Facebook's DNS servers were *also behind those BGP routes* — so when the BGP routes were withdrawn, DNS lookups for `facebook.com` failed too (the DNS servers were unreachable), even though DNS itself wasn't "broken."

---

## Problem 6: Comparing Architectures — Scalability Analysis

### Problem Statement

A startup is building a file-sharing service. They are deciding between three architectures:

- **Architecture A:** Pure Client/Server — a central server stores all files; clients request files from the server.
- **Architecture B:** Pure P2P — no central server; peers find each other via a distributed hash table (DHT); all content is stored and served by peers.
- **Architecture C:** Hybrid — a central **index server** tracks who has what file; actual file transfer happens peer-to-peer.

**(a)** For each architecture, identify: (1) the **single point of failure**, if any; (2) how **distribution time** scales with number of users N; (3) whether a **NAT traversal** problem exists.

**(b)** A new copyright law requires the startup to be able to **remove a specific file** from the system within 24 hours of a takedown notice. Which architecture makes this easiest? Which makes it nearly impossible? Explain.

**(c)** The startup expects 10 million users globally with peak concurrent connections of 500,000. Describe the **server cost** structure for each architecture as N grows. Which is cheapest to operate at scale? Which is most expensive?

**(d)** Architecture C (hybrid) is essentially how **original Napster** worked (1999–2001) and how **BitTorrent with trackers** works today. Napster was shut down via a court order targeting its central index server. Could BitTorrent be shut down the same way? Why or why not?

**(e)** Modern BitTorrent uses a **DHT (Distributed Hash Table)** for trackerless operation. What problem does DHT solve? Why does a pure DHT eliminate the legal vulnerability of a central index server?

---

### Solution

**(a) Comparison table:**

| Criterion | A: Pure Client/Server | B: Pure P2P (DHT) | C: Hybrid (Index + P2P) |
|-----------|----------------------|-------------------|------------------------|
| Single point of failure | **Yes** — the server. If it goes down, the entire service stops. | **No** — the network continues as long as some peers are online. Highly resilient. | **Yes** — the index server. If it goes down, peers can't find each other (even if they have files). |
| Distribution time scale | **O(N)** — grows linearly with users. Server upload bandwidth is fixed ceiling. | **O(1)** (approximately) — approaches constant as N grows because capacity scales with demand. | **O(1)** for data distribution (P2P); index server is small per query. |
| NAT traversal problem | **No** — clients always initiate outbound connections to the server. Clients behind NAT work fine. | **Yes** — peers need to connect to each other. Both may be behind NAT. Requires STUN/ICE/hole punching. | **Partial** — index lookup is client/server (no NAT issue); data transfer is P2P (NAT problem). |

**(b) File takedown:**

**Easiest: Architecture A (Pure Client/Server).** The startup simply deletes the file from its server. Done. Since all content lives on company-controlled infrastructure, a takedown is a database delete. This can happen in seconds.

**Nearly impossible: Architecture B (Pure P2P with DHT).** The file is replicated across thousands or millions of peer devices worldwide. The startup has no control over peers' devices. To remove a file, they would need to contact every peer that has it — impossible. Even if 99% of peers comply, the remaining 1% continue seeding. The content is effectively indestructible. This is by design for legitimate use cases (censorship resistance); it's a liability for compliance.

**Architecture C is intermediate:** The index server can be modified to stop returning results for that file's hash — clients can no longer discover peers who have it. However, peers who already know each other's addresses (from a previous lookup) can still transfer directly. New users can't find the file; existing seeders can still share if they coordinate out-of-band.

**(c) Server cost as N grows:**

**Architecture A:** Server cost grows **linearly with N**. Double the users → double the bandwidth costs → double the server costs (or double the number of CDN nodes). At 500,000 concurrent downloads of a 1 GB file, the server must sustain massive outbound bandwidth. This is the dominant operating cost. Netflix, for example, spends hundreds of millions annually on CDN bandwidth.

**Architecture B:** Server cost is **near zero** (or fixed/minimal). The startup only needs minimal infrastructure (perhaps bootstrap nodes to help new peers discover the network). All bandwidth and storage is contributed by peers. Operating cost is essentially constant regardless of N. The "server" cost is O(1).

**Architecture C:** Server cost is **small but fixed** for the index server (index queries are tiny — just metadata, not file data). The index server handles lookups but no file content. Actual bandwidth comes from peers. The index server needs to scale with query rate (roughly O(N)) but each query is tiny. This is far cheaper than serving files directly.

**Cheapest at scale: Architecture B** (nearly free infrastructure), then **C** (cheap index only), then **A** (expensive — proportional to total data transferred).

**(d) Napster shutdown vs. BitTorrent:**

**Napster** had a **central index server** — this was Architecture C. When courts ordered Napster to remove infringing content and ultimately shut down the service (2001), they simply targeted the company that operated the central index servers. One court order, one company, and the service died immediately: without the index, peers couldn't find each other.

**BitTorrent with trackers** has the same legal vulnerability. Individual trackers (like The Pirate Bay's tracker) have been seized by law enforcement. When a tracker is taken down, torrents pointing to it stop working (no peer discovery). This is why The Pirate Bay was repeatedly targeted.

**Could BitTorrent itself be shut down?** BitTorrent (the protocol) cannot be shut down because it is an open specification — anyone can implement it. However, specific **tracker services** can be targeted. This is why the community moved to trackerless operation.

**(e) DHT and legal resilience:**

A **Distributed Hash Table (DHT)** replaces the central tracker by distributing the "index" function across all peers. Each peer is responsible for storing the location of a small subset of torrents (determined by a hash function). To find peers for a file, you query the DHT — the network of peers collaboratively answers, with no single peer holding the full index.

**Problem DHT solves:** Eliminates the central tracker as a single point of failure *and* single point of legal vulnerability. The "index server" is now distributed across millions of devices worldwide.

**Why it eliminates the legal vulnerability:** There is no company, no server, no legal entity to serve a court order to. The index *is* the network of peers. You would need to take down every participating peer simultaneously — in every country, across all jurisdictions. This is practically impossible. The protocol itself becomes the "server," and it lives everywhere.

This is the same reason that decentralized protocols (Bitcoin, IPFS) are legally difficult to shut down: they have no center to attack.

---

## Problem 7: Putting It All Together — Architecture Design

### Problem Statement

You are designing the networking architecture for a **live video streaming platform** (think Twitch or YouTube Live). Streamers broadcast live video; viewers watch in real time.

**(a)** A pure P2P architecture seems appealing for scalability. Explain two fundamental problems with using P2P for **live** streaming specifically (as opposed to file sharing). Why is live streaming architecturally different from file distribution?

**(b)** In the real architecture (which is client/server), the streamer's client uploads to a server, and viewers download from the server. What bottleneck does the **streamer's home upload bandwidth** create? How does a CDN (Content Delivery Network) solve the viewer-side scaling problem?

**(c)** The streamer's upload capacity is `u_streamer`. There are N viewers each with download capacity `d_viewer`. The video stream requires `r` Mbps. Write the condition for when the **server-based CDN architecture** can support all N viewers without being bandwidth-constrained.

**(d)** Live streaming requires very low **end-to-end latency** — viewers should see the stream within 1–5 seconds of it happening. Map out all the sources of delay from the streamer's camera to the viewer's screen. Which is the dominant latency component, and what can be done to minimize it?

**(e)** A viewer watching a Twitch stream chats with the streamer. The chat message travels from the viewer's browser to Twitch's chat server, and the streamer sees it in their streaming software. Is this chat system client/server or P2P? What protocol would you use for the chat messages, and why?

---

### Solution

**(a) Why P2P fails for live streaming:**

**Problem 1: There is no complete file to share yet.** P2P file distribution (BitTorrent) works because the file exists completely and can be split into pieces. Any peer with a piece can share it. In live streaming, the video is being *created in real time* — the next 5 seconds of content don't exist yet. You cannot "seed" content that hasn't happened. The fundamental P2P speedup (pieces spread exponentially through the swarm) requires pre-existing content.

**Problem 2: Latency requirements are incompatible with P2P propagation delay.** For a viewer to receive live video via P2P, the video must travel: streamer → some peer → another peer → ... → viewer. Each P2P hop adds latency (multiple RTTs). For live streaming requiring 1–5 seconds end-to-end delay, multi-hop P2P propagation is too slow and too variable. CDN servers in well-connected data centers with direct, low-latency routes are far better suited.

Additionally, P2P requires viewers to upload to other viewers — many viewers are behind NAT or have asymmetric connections (high download, low upload), which limits their ability to redistribute the stream.

**(b) Streamer upload bottleneck and CDN solution:**

The streamer's home connection has limited **upload bandwidth** (often 5–20 Mbps for residential broadband). The video stream might require 6 Mbps for 1080p60. The streamer can only upload to **one destination** simultaneously without exceeding their upload cap — they cannot directly serve thousands of viewers.

**CDN solution:**
1. The streamer uploads **one copy** of the stream to the nearest CDN **ingest node** (a nearby data center with high bandwidth connectivity).
2. The CDN **replicates** the stream across dozens of **edge servers** worldwide (close to viewers geographically).
3. Each viewer connects to their nearest edge server (100–200 ms away instead of potentially 500 ms to the origin).
4. Each edge server serves thousands of local viewers from its single received copy.

The streamer still uploads just one stream; the CDN does the fan-out. This is hierarchical client/server — the streamer is a client to the ingest, edge servers are clients to origin, viewers are clients to edge servers.

**(c) Bandwidth condition for N viewers:**

Each of N viewers needs `r` Mbps of throughput. The CDN must supply `N × r` Mbps of total outbound bandwidth. If the CDN has total capacity C_cdn (sum of all edge server bandwidths), the condition for supporting all viewers is:

> **N × r ≤ C_cdn**

Or equivalently, the CDN must have:

> **C_cdn ≥ N × r**

The streamer's constraint is separate: `u_streamer ≥ r` (must be able to upload the full stream once to the CDN). As long as the CDN has enough capacity, adding viewers doesn't affect the streamer or the ingest — it only affects CDN edge server capacity, which can be scaled horizontally.

**(d) Sources of end-to-end latency:**

Working through the pipeline from camera to viewer's screen:

| Stage | Latency Source | Typical Value |
|-------|---------------|---------------|
| **Capture & encoding** | Camera captures frames; encoder (H.264/H.265) compresses video into a buffer before uploading | 200–500 ms |
| **Streaming protocol overhead** | RTMP/SRT chunks video into segments; larger segments = more buffering before upload | 500 ms – 2 s |
| **Upload to CDN ingest** | Network propagation from streamer's home to nearest CDN data center | 10–50 ms |
| **CDN processing** | Transcoding to multiple quality levels, packaging for HLS/DASH | 100–500 ms |
| **CDN edge propagation** | Replication from origin to edge server near the viewer | 50–200 ms |
| **Player buffering** | Viewer's video player buffers 2–10 seconds to absorb network jitter | 2–10 s |

**Dominant component: Player buffering** (2–10 seconds). Viewers' players buffer heavily to survive packet loss and variable bandwidth — if the buffer runs dry, playback freezes. This is the dominant latency component, not propagation delay.

**How to minimize:** Use lower-latency streaming formats (WebRTC instead of HLS achieves < 1 second; HLS Low-Latency targets 2–3 seconds vs. traditional HLS at 15–30 seconds). Reduce segment sizes, reduce player buffer depth. Accept more rebuffering events in exchange for lower latency — the tradeoff every streaming platform must make.

**(e) Chat system architecture:**

The Twitch chat system is **client/server**. Specifically:

- Each viewer's browser opens a **WebSocket connection** to Twitch's chat server
- The streamer's software also has a WebSocket connection to the same chat server
- When a viewer sends a message: viewer → Twitch chat server → streamer's client (and optionally broadcast back to all viewers)

**Protocol choice: WebSocket over TCP (port 443)**

**Why WebSocket, not regular HTTP:**
- HTTP is request/response — the *server* cannot push a message to the viewer without a request. Chat requires the server to push incoming messages to all viewers in real time.
- WebSocket upgrades an HTTP connection to a **full-duplex persistent channel** — both sides can send at any time without waiting for a request.
- WebSocket runs over TCP (reliable, ordered delivery) — important for chat, where you don't want messages arriving out of order or dropped silently.

**Why not P2P for chat:** Chat messages must be **moderated** (filtered for banned words, spam, abuse) and must reach **all viewers simultaneously** — a fan-out problem that a central server handles efficiently. P2P chat would require complex gossip protocols to reach everyone, with no central point for moderation. The client/server model is correct here.

---

*End of Practice Problems*

**Topics covered in this set:**
- Client/Server vs. P2P definitions and classification
- P2P distribution time formula and scaling analysis (D_P2P vs. D_cs)
- NAT traversal: STUN, UDP hole punching, TURN, ICE
- HTTP as client/server: RTTs, persistent connections, statelessness, cookies
- DNS hierarchy: iterative/recursive resolution, TTL, BGP-DNS interactions
- Architecture comparison: scalability, SPOF, legal resilience, DHT
- Real-world design: live streaming, CDN, latency budgets, WebSocket