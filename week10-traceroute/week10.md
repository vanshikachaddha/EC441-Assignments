# EC 441 — Tools Practice Problems & Solutions
## Tools Covered: ping, traceroute, Wireshark, Mininet, Sockets

---

## Problem 1: Interpreting `ping` Output and ICMP Mechanics

### Problem Statement

You run the following command on your laptop:

```
$ ping -c 5 8.8.8.8
PING 8.8.8.8: 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=117 time=14.3 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=14.1 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=13.9 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=117 time=51.2 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=117 time=14.0 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 packets received, 0.0% packet loss
round-trip min/avg/max = 13.9/21.5/51.2 ms
```

Answer the following:

**(a)** What ICMP message types and codes are used in a `ping` exchange? Describe the round-trip at the protocol level.

**(b)** The TTL in the reply is 117. What was most likely the **initial TTL** set by the Google server, and how many hops did the packet travel?

**(c)** `icmp_seq=3` has a round-trip time of 51.2 ms while all others are ~14 ms. Give **two distinct** possible explanations for this spike.

**(d)** Does `ping` use TCP or UDP? Explain precisely where in the IP stack it operates and what the IP `Protocol` field value will be in these packets.

**(e)** Your classmate says "ping proves the path is working end-to-end." Is this fully accurate? Describe a scenario where `ping` succeeds but TCP connections to the same host fail.

---

### Solution

**(a) ICMP message types used by ping:**

The client sends an **ICMP Echo Request (Type 8, Code 0)**. It includes an identifier (so the sender can match replies to the right `ping` process), a sequence number (so you can detect loss and match RTTs), and an optional data payload. The destination host's network layer sees `Protocol = 1` in the IP header and hands the payload to ICMP. ICMP responds with an **ICMP Echo Reply (Type 0, Code 0)** containing the same identifier, sequence number, and payload mirrored back. The sender timestamps the departure and arrival to compute RTT.

Neither TCP nor UDP is involved — `ping` operates at the **network layer** using raw ICMP.

**(b) Estimating hops from TTL:**

Common initial TTL values are:
- **64** → Linux/macOS
- **128** → Windows
- **255** → Cisco routers and some network equipment

The reply TTL is 117. The most likely initial TTL is **128** (Windows or some network appliances), because:

```
hops = initial TTL − observed TTL = 128 − 117 = 11 hops
```

If Google's servers used an initial TTL of 64, the observed TTL would need to be negative (impossible). If initial TTL were 255, that would imply 138 hops, which is unrealistically large for a path to Google's DNS server.

**Answer: Initial TTL = 128; the packet traveled 11 hops.**

**(c) Two explanations for the 51.2 ms spike:**

1. **Queuing delay at a congested router.** One or more routers along the 11-hop path had a momentarily full output queue. The ICMP packet was buffered behind a burst of other traffic. This is transient and does not indicate a routing change. Think of it like hitting a traffic jam on one specific intersection during your commute — the road is the same, just temporarily backed up.

2. **Route change / brief path oscillation.** BGP or OSPF caused the packet to briefly take a longer path (more hops or a slower link) before reverting. This is less common but observable, especially on intercontinental paths. The subsequent packets resumed the original shorter path.

**(d) Where ping operates:**

`ping` does **not** use TCP or UDP. It uses **raw ICMP**, which is encapsulated directly in an IP datagram. The **`Protocol` field in the IP header = 1** (ICMP). There is no transport-layer header — the ICMP header immediately follows the IP header. This is why `ping` requires elevated privileges on many systems: opening a raw IP socket bypasses the transport layer entirely.

**(e) ping success does not guarantee TCP works:**

A firewall may be configured to **allow ICMP but block TCP SYN packets** to specific ports. For example:

- A server hosting a web service might allow `ping` (ICMP Echo) for network diagnostic purposes but block `TCP:443` via a firewall rule.
- `ping` would return replies (no packet loss), but `curl https://server` would time out because the TCP three-way handshake can never complete.

Also, `ping` tests ICMP reachability of the **IP address**, not whether any service is listening. A host could be reachable via ICMP but have its web server process crashed.

---

## Problem 2: Reading `traceroute` Output

### Problem Statement

You run `traceroute` from a host at Boston University to a server at MIT:

```
$ traceroute mit.edu
traceroute to mit.edu (18.9.22.169), 30 hops max
 1  192.168.1.1         1.1 ms    1.2 ms    1.1 ms
 2  10.24.0.1           6.3 ms    6.4 ms    6.2 ms
 3  128.197.8.1         9.1 ms    9.0 ms    9.2 ms
 4  192.5.89.33        10.5 ms   10.4 ms   10.6 ms
 5  * * *
 6  18.168.0.1         11.2 ms   11.4 ms   11.1 ms
 7  18.9.22.169        11.8 ms   11.9 ms   11.7 ms
```

Answer the following:

**(a)** Explain the TTL trick that `traceroute` exploits. What ICMP message does each intermediate router send back, and what does that reveal?

**(b)** What does `* * *` on hop 5 mean? Does it mean the packet was dropped at that hop for all traffic? How would you determine whether the end-to-end path is actually broken?

**(c)** Classify each visible hop by its likely role:
- Hop 1: `192.168.1.1`
- Hop 2: `10.24.0.1`
- Hop 3: `128.197.8.1`
- Hop 6: `18.168.0.1`

**(d)** The three RTT measurements per hop (three columns) for hop 6 are 11.2, 11.4, and 11.1 ms, while hop 7 is 11.8, 11.9, and 11.7 ms. These are suspicious — hop 7 (the destination) is only ~0.6 ms more than hop 6. What does this suggest about where MIT's server is physically located relative to hop 6?

**(e)** A student says: "traceroute uses a special protocol that makes routers report their addresses." Correct this misconception precisely.

---

### Solution

**(a) The TTL trick:**

IP requires every router to **decrement the TTL field by 1** when forwarding a packet. If a router decrements TTL to 0, it must:
1. Drop the packet
2. Send an **ICMP Time Exceeded (Type 11, Code 0)** message back to the original source — and that reply's **source IP reveals the router's address**.

`traceroute` exploits this by sending probes with TTL = 1, 2, 3, 4, … in sequence:
- TTL=1 → first router expires it, sends back ICMP Time Exceeded → first hop revealed
- TTT=2 → second router expires it → second hop revealed
- And so on until the destination is reached

Think of it like sending a series of messages each stamped "self-destruct after N stops" — each one detonates at a different point in the journey, and the debris (ICMP reply) tells you who handled it.

**(b) `* * *` on hop 5:**

`* * *` means the router at hop 5 **did not send back an ICMP Time Exceeded reply** when it discarded the TTL-expired probe. This is a **common firewall policy** — many routers are configured to silently drop TTL-expired packets rather than generate ICMP replies (rate limiting, security policy, or administrative choice).

**It does NOT mean the packet was dropped for all traffic.** The probe for hop 5 was silently dropped, but subsequent probes (with TTL = 6, 7) reached and got replies from hops 6 and 7. This proves that traffic IS passing through whatever is at hop 5 — it just doesn't respond to ICMP diagnostics.

To check if the end-to-end path is actually broken: look at whether the **final destination** (hop 7) responds. Here it does (18.9.22.169 replies), so the path is intact.

**(c) Classifying hops:**

| Hop | Address | Role |
|-----|---------|------|
| 1 | `192.168.1.1` | **Home/office router** — RFC 1918 private address; this is the default gateway on your local LAN |
| 2 | `10.24.0.1` | **ISP edge router or BU upstream** — another RFC 1918 address (10.x.x.x); suggests NAT/carrier infrastructure or BU internal routing |
| 3 | `128.197.8.1` | **BU campus edge router** — 128.197.0.0/16 is BU's publicly assigned address block (Class B from the classful era); this is the BU network boundary |
| 6 | `18.168.0.1` | **MIT internal router** — 18.0.0.0/8 is MIT's Class A block (one of the few Class A blocks allocated in the classful era); this is MIT's campus infrastructure |

**(d) Physical proximity of server to hop 6:**

The RTT difference between hop 6 (MIT's internal router) and hop 7 (the final destination) is only ~0.6–0.7 ms. At the speed of light in fiber (~200,000 km/s), 0.6 ms round-trip represents a one-way distance of only ~60 km. This strongly suggests the MIT server is **physically on or very close to MIT's campus network**, possibly in the same building as the border router. The near-zero additional latency is consistent with an in-campus data center.

**(e) Correcting the misconception:**

Routers do **not** do anything special for `traceroute`. They are simply executing their **normal, standard IP forwarding logic** — specifically, the rule that says "if TTL reaches 0 after decrement, drop the packet and send ICMP Time Exceeded." This behavior exists for all IP traffic (it's how routing loops are prevented), not just traceroute probes.

`traceroute` is an emergent behavior from the TTL mechanism — it's a clever exploitation of an existing protocol feature, not a special protocol itself. The routers have no idea they're being "traced."

---

## Problem 3: Wireshark Analysis — TCP Three-Way Handshake

### Problem Statement

You capture traffic with Wireshark while loading `http://bu.edu`. You see the following packets (simplified):

```
No.  Time    Src             Dst             Protocol  Info
1    0.000   192.168.1.42    128.197.9.202   TCP       45231 → 80 [SYN] Seq=0 Win=65535 Len=0 MSS=1460 SACK_PERM
2    0.009   128.197.9.202   192.168.1.42    TCP       80 → 45231 [SYN, ACK] Seq=0 Ack=1 Win=28960 Len=0 MSS=1452 SACK_PERM
3    0.009   192.168.1.42    128.197.9.202   TCP       45231 → 80 [ACK] Seq=1 Ack=1 Win=65535 Len=0
4    0.009   192.168.1.42    128.197.9.202   HTTP      GET / HTTP/1.1
5    0.018   128.197.9.202   192.168.1.42    TCP       80 → 45231 [ACK] Seq=1 Ack=78 Win=28960 Len=0
6    0.020   128.197.9.202   192.168.1.42    HTTP      HTTP/1.1 200 OK (text/html)
```

Answer the following:

**(a)** Identify each phase of the TCP three-way handshake. Which packets form it, and what is exchanged in each step?

**(b)** In packet 2, the `Ack=1` field acknowledges the SYN. But no data was sent in packet 1 — it was zero bytes (Len=0). Why does the ACK say `1` instead of `0`?

**(c)** In packet 2, the server's `MSS=1452` while the client advertised `MSS=1460`. Why might these differ? What link technology might explain the server's smaller MSS?

**(d)** Both packets 1 and 2 carry `SACK_PERM`. What does this mean and what TCP behavior does it enable?

**(e)** The time between packet 1 (SYN) and packet 3 (ACK) is 9 ms. What is the approximate RTT to the BU web server, and what does this tell you about the physical distance?

**(f)** Write the Wireshark display filter that would show only this TCP connection (both directions).

---

### Solution

**(a) Three-way handshake phases:**

- **Packet 1 — SYN**: The client (192.168.1.42) initiates the connection. `SYN=1`, client ISN implicitly chosen (displayed as relative Seq=0 by Wireshark). This packet also negotiates options: MSS=1460 and SACK_PERM. The SYN logically "consumes" one sequence number.

- **Packet 2 — SYN-ACK**: The server (128.197.9.202) accepts. `SYN=1, ACK=1`. The server chooses its own ISN (relative Seq=0). The `Ack=1` confirms receipt of the client's SYN (ISN+1). The server also negotiates its own MSS=1452 and SACK support.

- **Packet 3 — ACK**: The client acknowledges the server's SYN. `ACK=1`, `Ack=1` (server ISN+1). After this, both sides are `ESTABLISHED` and data can flow.

**(b) Why does ACK=1 when no data was sent?**

TCP uses **byte-sequence semantics**, and the SYN flag itself **logically consumes one sequence number** even though it carries no data payload. This is by design — it makes the SYN retransmittable and acknowledg-able. If the SYN were at position 0 and consumed nothing, retransmitted SYNs would be indistinguishable from new connection requests. By consuming sequence number 0, the ACK=1 unambiguously says "I received your SYN and I'm ready for byte 1 (the first real data byte)."

The same applies to FIN — it also consumes one sequence number.

**(c) Why MSS=1452 vs MSS=1460:**

The standard Ethernet MTU is 1500 bytes. Subtracting a 20-byte IP header and 20-byte TCP header gives **MSS = 1460 bytes** — the client's advertised value, consistent with a standard Ethernet connection.

The server's MSS=1452 is **8 bytes smaller**. A common explanation: the server (or an upstream link) uses **PPPoE (Point-to-Point Protocol over Ethernet)**, which adds an 8-byte PPPoE header inside the Ethernet frame. This reduces the available payload from 1500 to 1492 bytes, and after subtracting IP (20) and TCP (20) headers: 1492 − 40 = **1452 bytes**. DSL connections commonly use PPPoE.

**(d) SACK_PERM:**

`SACK_PERM` (Selective Acknowledgment Permitted) is a TCP option included in both the SYN and SYN-ACK to **negotiate SACK support**. If both sides include it, the connection uses Selective Acknowledgment.

Without SACK: the receiver can only send cumulative ACKs (Go-Back-N style) — "I have everything up to byte N." If there's a gap, the sender doesn't know what arrived after the gap.

With SACK: the receiver can report specific byte ranges it has received out-of-order. The sender then retransmits **only the missing segments** rather than re-sending everything from the gap onward. This is Selective Repeat behavior from Lecture 18 applied to TCP.

SACK is enabled by default on Linux, macOS, and Windows, so you'll see `SACK_PERM` on virtually every modern TCP connection you capture.

**(e) RTT and physical distance:**

From the capture: packet 1 sent at `t=0.000`, packet 2 received at `t=0.009`. The time from SYN to SYN-ACK = **9 ms = RTT**.

Light travels through fiber at approximately 200,000 km/s. One-way time = 4.5 ms. Maximum one-way physical distance = 4.5 ms × 200,000 km/s = **900 km**.

Since BU is in Boston and the server is at BU, 9 ms is consistent with a server in the same city or building — likely just routing through the campus network and back. The physical fiber distance could be far less than 900 km (probably < 1 km).

**(f) Wireshark filter for this specific connection:**

```
tcp and (
  (ip.src == 192.168.1.42 and ip.dst == 128.197.9.202 and tcp.srcport == 45231 and tcp.dstport == 80)
  or
  (ip.src == 128.197.9.202 and ip.dst == 192.168.1.42 and tcp.srcport == 80 and tcp.dstport == 45231)
)
```

Shorter equivalent using Wireshark's conversation tracking:

```
tcp.stream eq 0
```

(assuming this is the first TCP stream captured — Wireshark assigns stream numbers automatically).

Or filter by the 4-tuple:
```
ip.addr == 192.168.1.42 and ip.addr == 128.197.9.202 and tcp.port == 80
```

---

## Problem 4: Wireshark — IP Header Fields

### Problem Statement

You capture a single packet in Wireshark. Expanding the IP layer shows:

```
Internet Protocol Version 4
  Version: 4
  Header Length: 20 bytes (5)
  Differentiated Services Field: 0x00
  Total Length: 576
  Identification: 0x1a2b (6699)
  Flags: 0x40, Don't fragment
    0... .... = Reserved bit: Not set
    .1.. .... = Don't fragment: Set
    ..0. .... = More fragments: Not set
  Fragment Offset: 0
  Time to Live: 128
  Protocol: TCP (6)
  Header Checksum: 0x3f2a [correct]
  Source Address: 172.31.4.5
  Destination Address: 104.18.25.34
```

Answer the following:

**(a)** What does the `Don't Fragment` flag tell you about how this packet was generated? What happens if this packet reaches a link with MTU < 576 bytes?

**(b)** The `TTL = 128` is an initial TTL value. What OS likely sent this packet? If you observe this packet arriving at a router 5 hops away, what TTL would you see?

**(c)** The source IP `172.31.4.5` is an RFC 1918 private address. Yet the destination `104.18.25.34` is a public internet address. Explain the full journey this packet takes from source to destination. What happens at the edge of the private network?

**(d)** The `Protocol = 6` field means TCP. If this were a DNS query instead, what Protocol value and transport would you expect, and what destination port?

**(e)** The `Header Checksum` is listed as "correct." When will a router **recompute** this checksum, and why?

---

### Solution

**(a) Don't Fragment flag:**

`DF=1` means the sender has explicitly instructed every router along the path: **do not fragment this packet, even if you encounter a smaller MTU.** This is typically set by applications using **Path MTU Discovery (PMTUD)** — they want to discover the minimum MTU along the path rather than have the network silently break their data into fragments.

If this packet arrives at a router that needs to forward it onto a link with MTU < 576:
- The router **cannot fragment** (DF is set)
- The router **drops the packet**
- The router sends an **ICMP Type 3, Code 4 (Destination Unreachable: Fragmentation Needed)** message back to `172.31.4.5`, including the MTU of the problematic link
- The sender reduces its packet size to fit that MTU and retransmits

If firewalls block ICMP Type 3 Code 4, this creates a **PMTUD black hole** — packets are silently dropped and the connection stalls.

**(b) OS fingerprinting from TTL:**

**Initial TTL = 128** → most likely **Windows** (Windows uses 128 as its default initial TTL; Linux/macOS use 64; Cisco uses 255).

If this packet has traveled 5 hops (each router decrements TTL by 1), the observed TTL = 128 − 5 = **TTL 123**.

**(c) Journey from private to public address — NAT:**

1. The host at `172.31.4.5` sends the packet with its private source address
2. The packet reaches the **NAT router** at the edge of the private network (e.g., a home router or office gateway)
3. The NAT router **rewrites the source IP** from `172.31.4.5` to its public IP (e.g., `203.0.113.1`) and replaces the source port with a unique port from its translation pool (e.g., `52341`)
4. The NAT router **records this mapping** in its translation table: `(203.0.113.1:52341) ↔ (172.31.4.5:<original_port>)`
5. The packet traverses the public internet with source `203.0.113.1` and reaches `104.18.25.34`
6. Return traffic arrives addressed to `203.0.113.1:52341` — the NAT router looks up port 52341, finds the mapping, and **rewrites the destination** back to `172.31.4.5` before forwarding inward

The address `172.16.0.0/12` is an RFC 1918 private block (172.16.0.0 – 172.31.255.255). This is commonly used in cloud VPCs (e.g., AWS default VPC uses 172.31.0.0/16).

**(d) DNS query protocol fields:**

A DNS query uses:
- **Protocol field = 17** (UDP)
- **Destination port = 53** (the well-known port for DNS)
- Source port = an ephemeral port (49152–65535) chosen by the OS

DNS uses UDP (not TCP) by default for queries because:
1. Single request / single response — no need for TCP's connection overhead
2. Typical response fits in one UDP datagram
3. The application can retry faster than TCP's backoff if no response arrives

If the response exceeds 512 bytes (or the EDNS0 advertised size), the server sets the TC (truncation) bit, signaling the resolver to retry over TCP.

**(e) When routers recompute the header checksum:**

Routers **recompute the header checksum at every hop** because the **TTL field changes** (decremented by 1) at each router. Since the TTL is covered by the checksum, any change to TTL invalidates the old checksum. The router decrements TTL, then recomputes the checksum over the updated header before forwarding.

This per-hop checksum recomputation is one reason IPv6 **eliminated the header checksum entirely** — it was constant CPU overhead at every router, when link-layer CRCs and transport-layer checksums already provide error detection at their respective levels.

---

## Problem 5: Mininet — Network Topology and Testing

### Problem Statement

You create the following Mininet topology:

```python
from mininet.topo import Topo
from mininet.net import Mininet

class MyTopo(Topo):
    def build(self):
        h1 = self.addHost('h1', ip='10.0.0.1/24')
        h2 = self.addHost('h2', ip='10.0.0.2/24')
        h3 = self.addHost('h3', ip='10.0.0.3/24')
        s1 = self.addSwitch('s1')
        s2 = self.addSwitch('s2')
        
        self.addLink(h1, s1, bw=100, delay='5ms')
        self.addLink(h2, s1, bw=100, delay='5ms')
        self.addLink(s1, s2, bw=10, delay='20ms')
        self.addLink(h3, s2, bw=100, delay='5ms')

net = Mininet(topo=MyTopo())
net.start()
```

Answer the following:

**(a)** Draw the topology. What is the path between h1 and h3? What is the bottleneck link?

**(b)** You run `h1 ping h3` inside the Mininet CLI. Predict the approximate round-trip time. Show your calculation.

**(c)** You run `h1 iperf h3` (TCP throughput test). Predict approximately what throughput you'll observe, and explain which link constrains it.

**(d)** You run `h1 ping h2`. What RTT do you predict, and why is it different from the h1→h3 case?

**(e)** What is the difference between a Mininet **switch** and a **router**? Are `s1` and `s2` routing IP packets or switching Ethernet frames? What protocol does `ping` use, and at which layer does the switch operate?

**(f)** You want to simulate packet loss. Write the Mininet/Linux command to add 2% random packet loss to the link between h1 and s1.

---

### Solution

**(a) Topology diagram:**

```
h1 (10.0.0.1) --[100Mbps, 5ms]-- s1 --[10Mbps, 20ms]-- s2 --[100Mbps, 5ms]-- h3 (10.0.0.3)
                                   |
h2 (10.0.0.2) --[100Mbps, 5ms]---/
```

The **bottleneck link** is `s1 ↔ s2`: only 10 Mbps, compared to 100 Mbps on all other links. Any traffic between h1/h2 and h3 must pass through this link.

**(b) Predicted RTT for h1 → h3:**

RTT = sum of **one-way delays × 2** (ping is round-trip):

One-way path: h1 → s1 → s2 → h3
- h1 to s1: 5 ms
- s1 to s2: 20 ms
- s2 to h3: 5 ms
- One-way total: **30 ms**

Round-trip (ping): **30 × 2 = 60 ms**

Plus small processing overhead at each switch (typically < 1 ms in Mininet), so you'd observe approximately **60–62 ms**.

**(c) Predicted iperf throughput:**

The bottleneck is the s1↔s2 link at **10 Mbps**. TCP cannot send faster than this link allows — all traffic between h1 and h3 must traverse it. The 100 Mbps links on either side are not the constraint.

Expected throughput: approximately **9–10 Mbps** (TCP has some overhead from headers and ACKs, so you rarely achieve exactly the link rate, but you'll be close).

If the TCP window is too small relative to the bandwidth-delay product, throughput may be further limited. BDP = 10 Mbps × 60 ms RTT = **75 KB**. Linux's default socket buffer is ~87 KB, which is just sufficient — so throughput should be close to 10 Mbps.

**(d) Predicted RTT for h1 → h2:**

h1 and h2 are both connected to the **same switch** (s1). The path is h1 → s1 → h2.

One-way: 5 ms (h1→s1) + 5 ms (s1→h2) = 10 ms
Round-trip: **20 ms**

This is much lower because the traffic never traverses the slow s1↔s2 link. Both hosts are on the same local segment.

**(e) Switch vs. Router:**

In Mininet, `s1` and `s2` are **Layer 2 switches** (they run Open vSwitch by default). They:
- Forward **Ethernet frames** based on **MAC addresses**
- Do **NOT** examine IP addresses
- Do **NOT** perform IP routing or decrement TTL
- Operate at the **Data Link Layer (Layer 2)**

`ping` uses **ICMP**, which is a **Layer 3 (Network layer)** protocol. However, since all three hosts (h1, h2, h3) are configured on the same /24 subnet (`10.0.0.0/24`), no routing is needed — they are all directly reachable at Layer 2. The switch just learns MAC addresses and forwards frames accordingly.

If hosts were on different subnets, you'd need a Mininet **router** (using `addHost` with IP forwarding enabled, or using `FRRouting`), which would examine IP headers and perform routing.

**(f) Adding packet loss with tc (traffic control):**

Inside the Mininet CLI, open a shell on h1 and use the `tc` (traffic control) command:

```bash
mininet> h1 tc qdisc add dev h1-eth0 root netem loss 2%
```

Or equivalently from outside using the Python API:

```python
h1 = net.get('h1')
h1.cmd('tc qdisc add dev h1-eth0 root netem loss 2%')
```

To verify it's applied:
```bash
mininet> h1 tc qdisc show dev h1-eth0
```

To remove:
```bash
mininet> h1 tc qdisc del dev h1-eth0 root
```

The `netem` (Network Emulator) qdisc is built into the Linux kernel and can simulate loss, delay, jitter, duplication, and reordering — making it the standard tool for controlled network experiments in Mininet.

---

## Problem 6: Sockets — Client-Server Communication

### Problem Statement

Consider the following Python server and client code:

**Server:**
```python
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 8080))
server.listen(5)
print("Server listening on port 8080")

while True:
    conn, addr = server.accept()
    print(f"Connection from {addr}")
    data = conn.recv(1024)
    conn.sendall(b"Echo: " + data)
    conn.close()
```

**Client:**
```python
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('127.0.0.1', 8080))
client.sendall(b"Hello, server!")
response = client.recv(1024)
print(f"Received: {response}")
client.close()
```

Answer the following:

**(a)** `socket.AF_INET` and `socket.SOCK_STREAM` are used. What do these constants specify? What would you use for a UDP socket instead?

**(b)** The server binds to `'0.0.0.0'`. What does this mean? How is this different from binding to `'127.0.0.1'` or `'192.168.1.5'`?

**(c)** Walk through what happens at the TCP level when `client.connect(('127.0.0.1', 8080))` is called. What packets are exchanged?

**(d)** The server calls `server.listen(5)`. What does the argument `5` mean? What happens if more than 5 clients attempt to connect simultaneously?

**(e)** The server uses `conn.recv(1024)`. Is it guaranteed that this call receives exactly the entire message "Hello, server!" in one call? Why or why not? How should production code handle this?

**(f)** The 5-tuple uniquely identifies a TCP connection. Write out the 5-tuple for the connection between client and server in this example, filling in what you know and noting what is unknown/dynamic.

**(g)** `SO_REUSEADDR` is set on the server socket. Why is this option important? What error would you see without it when restarting the server quickly?

---

### Solution

**(a) Socket constants:**

- `AF_INET` — **Address Family: Internet**. Specifies IPv4 addressing (use `AF_INET6` for IPv6, or `AF_UNIX` for Unix domain sockets on the same machine).
- `SOCK_STREAM` — **Streaming socket**. This selects **TCP** — a reliable, ordered, connection-oriented byte-stream protocol. The OS handles segmentation, retransmission, flow control, and congestion control.

For a **UDP socket**, you would use:
```python
socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
```
`SOCK_DGRAM` = datagram socket = UDP. No connection, no reliability guarantees, messages are discrete packets.

**(b) Binding to `0.0.0.0`:**

`0.0.0.0` means **"listen on all available network interfaces"** — the server will accept connections arriving on any IP address configured on the machine (localhost `127.0.0.1`, the LAN address `192.168.1.5`, a public IP, etc.).

- Binding to `127.0.0.1`: only accepts connections **from the same machine** (loopback). External clients cannot connect.
- Binding to `192.168.1.5`: only accepts connections arriving on that **specific interface** (e.g., only from the LAN, not from a VPN interface).
- Binding to `0.0.0.0`: accepts on **all interfaces** — the most permissive option, typical for servers that should be reachable from anywhere.

**(c) TCP-level events during `connect()`:**

`client.connect(('127.0.0.1', 8080))` triggers the TCP **three-way handshake**:

1. **SYN**: The client's OS sends a TCP segment with `SYN=1` and a randomly chosen ISN to `127.0.0.1:8080`. The OS also assigns an ephemeral source port (e.g., 54321).

2. **SYN-ACK**: The server's OS (which is listening because `server.listen()` was called) responds with `SYN=1, ACK=1` — its own ISN and an acknowledgment of the client's ISN+1.

3. **ACK**: The client's OS sends the final ACK. The connection is now `ESTABLISHED` on both sides.

`connect()` **blocks** until step 3 completes (or times out with a connection refused/timeout error). After `connect()` returns, the socket is ready for `send()`/`recv()`.

**(d) `server.listen(5)` — the backlog:**

The argument `5` is the **backlog** — the maximum number of connections waiting in the **accept queue** (connections that have completed the three-way handshake but have not yet been `accept()`-ed by the server application).

If more than 5 clients complete their handshakes before `server.accept()` is called, the OS **drops incoming SYN packets** for the 6th and beyond (or returns RST on some systems). The client will retry or fail with "Connection refused" or timeout.

The backlog is not the maximum number of simultaneous active connections — it's just the pending queue. Once `accept()` is called, connections are moved out of the queue and a new slot opens.

**(e) `recv(1024)` — is the full message guaranteed?**

**No, it is not guaranteed.** TCP is a **byte-stream protocol**, not a message protocol. There are no message boundaries at the TCP level. `recv(1024)` may return:
- Exactly 14 bytes ("Hello, server!") — common for short messages on localhost
- Fewer bytes if the OS only has part of the data buffered
- More bytes if two sends were coalesced (Nagle's algorithm)

This is called the **framing problem**. Production code should:

1. **Use a fixed message length** and loop until all expected bytes arrive:
```python
def recv_exactly(sock, n):
    data = b''
    while len(data) < n:
        chunk = sock.recv(n - len(data))
        if not chunk:
            raise ConnectionError("Socket closed")
        data += chunk
    return data
```

2. **Use a delimiter** (e.g., newline `\n`) and buffer until the delimiter is found.

3. **Prefix messages with a length field** (e.g., 4-byte big-endian integer indicating payload size) and read that many bytes.

HTTP solves this with the `Content-Length` header and blank-line delimiters for headers.

**(f) The 5-tuple:**

A TCP connection is uniquely identified by:

```
(Protocol, Source IP, Source Port, Destination IP, Destination Port)
```

For this connection:

| Field | Value |
|-------|-------|
| Protocol | TCP (6) |
| Source IP | 127.0.0.1 (client) |
| Source Port | Dynamic ephemeral (e.g., 54321) — assigned by OS |
| Destination IP | 127.0.0.1 (server on same machine) |
| Destination Port | 8080 |

The source port is the only unknown — it is dynamically assigned from the ephemeral range (49152–65535 on Linux) by the OS when `connect()` is called. You can observe it with `ss -tnp` on Linux.

**(g) `SO_REUSEADDR`:**

After a TCP connection closes, the port enters **TIME_WAIT** state for 2×MSL (~60–120 seconds) on the active closer (usually the server if it closes connections). Without `SO_REUSEADDR`, trying to re-`bind()` to the same port while it's in TIME_WAIT gives:

```
OSError: [Errno 98] Address already in use
```

`SO_REUSEADDR` tells the OS: "allow binding to a port that is in TIME_WAIT, as long as the full 5-tuple of any new connection is different from the old one." Since a new connection would come from a different client port, the 5-tuple will differ and there's no ambiguity — so it's safe.

This is essential for server development: without it, you'd have to wait 60–120 seconds after every restart before the server could bind to its port again.

---

## Problem 7: Integrated Tools Problem — Diagnosing a Network Problem

### Problem Statement

A user reports: "I can ping google.com but I cannot load any web pages in my browser." You are the network engineer. Walk through a systematic diagnosis using the tools covered in this course.

You have access to: `ping`, `traceroute`, `Wireshark`, `netstat`/`ss`, and a browser.

The user's machine has:
- IP address: `10.0.1.45/24`
- Default gateway: `10.0.1.1`
- DNS server: `8.8.8.8`

**Step through the diagnosis:**

**(a)** `ping 10.0.1.1` succeeds. `ping 8.8.8.8` succeeds. `ping google.com` succeeds. What have you confirmed about the network layer?

**(b)** You run `traceroute google.com` — it completes successfully showing 12 hops. What additional information does this give you beyond what `ping` already confirmed?

**(c)** You run `curl http://www.google.com` from the command line — it hangs indefinitely. What does this tell you that changes the diagnosis?

**(d)** You open Wireshark and capture while running `curl http://www.google.com`. You see:
```
TCP [SYN] from 10.0.1.45:52301 to 142.250.80.46:80
TCP [RST, ACK] from 142.250.80.46:80 to 10.0.1.45:52301
```
What does the RST response mean? What entity likely sent it, and why?

**(e)** You modify the curl to use HTTPS: `curl https://www.google.com`. Wireshark shows:
```
TCP [SYN] to 142.250.80.46:443
TCP [SYN, ACK] from 142.250.80.46:443
TCP [ACK]
TLSv1.3 Client Hello
(no response — connection hangs)
```
Now what is your diagnosis?

**(f)** What is the most likely root cause? What command would confirm your theory, and what is the fix?

---

### Solution

**(a) What successful ping of `google.com` confirms:**

- **Layer 1/2**: Physical and link layer connectivity is working (you can reach the gateway)
- **Layer 3 — local routing**: The default gateway `10.0.1.1` is reachable
- **Layer 3 — internet routing**: IP packets can traverse the internet to and from Google's infrastructure (8.8.8.8 and google.com's IPs)
- **DNS**: `ping google.com` resolves the hostname — DNS lookups over UDP port 53 are working
- **ICMP**: Specifically ICMP Echo Request/Reply (Type 8/Type 0) packets are passing in both directions

**Not confirmed**: TCP connectivity, HTTP/HTTPS services, firewall rules specific to TCP.

**(b) What traceroute adds:**

`traceroute` confirms the **path** — the sequence of routers between the user and Google. Beyond what ping confirms, it tells you:
- How many hops the path traverses (12 in this case)
- The RTT to each intermediate router, helping identify where latency lives
- Whether any hops are unusually slow (suggesting congestion at a specific router)
- The ASNs traversed (e.g., your ISP vs. Google's AS)

It also confirms that **TTL-expiry ICMP replies** work at each hop — somewhat useful but still only confirming ICMP.

**(c) `curl http://google.com` hangs:**

This is a critical clue. `curl` attempts a **TCP connection** to port 80. The fact that it hangs (rather than getting an error quickly) tells you:
- The TCP SYN packet is either being **silently dropped** somewhere
- Or the **connection is blocked** by a firewall that doesn't send RST

This isolates the problem to **TCP connectivity**, not IP connectivity. The IP layer works fine (ping succeeds); TCP port 80 does not.

**(d) RST response in Wireshark:**

A **TCP RST (Reset)** is an abrupt connection termination — no data, no graceful teardown. When you send a SYN to port 80 and receive RST-ACK in response, it means:

> "No one is listening on this port, or access is explicitly denied."

The most likely sender is **not** Google's web server (Google's servers definitely have HTTP running on port 80). More likely candidates:
1. **A firewall** (corporate/school firewall, or a firewall on the user's router) that intercepts outbound HTTP (port 80) and sends a synthetic RST to reject it — some organizations block HTTP to force HTTPS usage
2. **The user's own firewall** (Windows Defender, iptables rule) blocking outbound TCP:80

The RST is "polite" compared to a silent drop — at least you get an immediate error signal.

**(e) HTTPS hangs after TLS Client Hello:**

The TCP three-way handshake to port 443 **succeeds** (SYN → SYN-ACK → ACK). The TLS Client Hello is sent. But no response arrives — the connection stalls after the TLS handshake begins.

This pattern is consistent with a **firewall doing Deep Packet Inspection (DPI)** that:
1. Allows the TCP handshake (so SYN/SYN-ACK works)
2. Inspects the TLS ClientHello to check the SNI (Server Name Indication) — which reveals the target hostname even in HTTPS traffic
3. **Silently drops** the connection once it identifies the destination as unapproved

Alternatively, it could be a **transparent HTTPS proxy** that is misconfigured, or an **MTU/PMTUD black hole** (but this would typically affect large packets, not small TLS handshakes).

**(f) Most likely root cause and fix:**

**Most likely cause: A corporate or ISP firewall is blocking or inspecting HTTP/HTTPS traffic.**

Given:
- ping works ✓ (ICMP allowed)
- DNS works ✓ (UDP:53 allowed)
- HTTP (TCP:80) → RST immediately (blocked, port reset)
- HTTPS (TCP:443) → hangs after TLS hello (deep packet inspection dropping connections)

**Confirming command:**
```bash
# Test if other TCP ports work (e.g., SSH to a known server)
nc -zv google.com 443
# Or try a DNS-over-HTTPS or alternate port
curl --verbose https://1.1.1.1/
# Check if the issue is specific to this machine's firewall
sudo iptables -L -n  # Linux
# Check for any proxy settings
env | grep -i proxy
echo $https_proxy $http_proxy
```

**Fix options:**
1. If it's a corporate firewall policy: contact IT — the machine may not be properly authenticated to the network, or HTTP may be intentionally blocked for compliance reasons
2. If it's a local iptables rule: `sudo iptables -F` (flush all rules) or identify and remove the specific blocking rule
3. If it's a proxy requirement: configure the system or browser proxy settings to route through the corporate proxy server

---

*End of Practice Problems*

**Topics covered:**
- `ping`: ICMP Echo Request/Reply, TTL, RTT interpretation
- `traceroute`: TTL exploitation, ICMP Time Exceeded, `* * *`, hop classification
- `Wireshark`: TCP handshake analysis, IP header fields, filter syntax, SACK
- `Mininet`: topology construction, RTT prediction, bottleneck analysis, traffic shaping with `tc`
- `sockets`: TCP vs UDP, bind/listen/accept, byte-stream framing, 5-tuple, TIME_WAIT