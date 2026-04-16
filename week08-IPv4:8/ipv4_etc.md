# Problem Set: IPv4, IPv6, DHCP, and NAT

**Course:** EC 441 – Introduction to Computer Networking, Spring 2026
**Topic:** Network Layer — IPv4, IPv6, DHCP, NAT, ICMP, Fragmentation
**Type:** Problem

---

## Problem 1 — IPv4 Header Fields

### Part (a)

For each field below, state its size in bits and explain its purpose in one or two sentences.

1. TTL
2. Protocol
3. IHL
4. Fragment Offset
5. DF flag

### Part (b)

A router receives an IPv4 packet with TTL = 1. What does the router do with it, and what
message does it send back? What tool exploits this behavior, and how?

### Part (c)

The IPv4 header checksum covers only the header, not the payload. Why is it still recomputed
at every hop? Why did IPv6 remove the header checksum entirely?

### Part (d)

A host sends a packet with Protocol = 17. What does the receiving host's network layer do
with this value, and what upper-layer module receives the payload?

---

**Solutions**

**1a.**

1. TTL (8 bits) — Time to Live. Each router decrements this field by 1 before forwarding. When
   it reaches 0, the router drops the packet and sends an ICMP Time Exceeded message back
   to the source. This prevents packets from looping forever in the presence of routing loops.

2. Protocol (8 bits) — identifies the upper-layer protocol carried in the payload. The receiving
   host's network layer uses this as a demultiplexing key to decide which transport-layer module
   should receive the data. Common values: 1 = ICMP, 6 = TCP, 17 = UDP, 89 = OSPF.

3. IHL (4 bits) — Internet Header Length. Specifies the length of the header in 32-bit words.
   The minimum is 5 (20 bytes, no options); the maximum is 15 (60 bytes). Routers need this
   field to know where the header ends and the payload begins, because the header size is
   variable when options are present.

4. Fragment Offset (13 bits) — indicates the position of this fragment's payload within the
   original datagram, measured in 8-byte units. The first fragment has offset 0. The destination
   uses this field, along with the Identification field, to reassemble fragments in the correct
   order.

5. DF flag (1 bit) — "Don't Fragment." When set, the router must not fragment the datagram
   even if it is larger than the outgoing link's MTU. Instead, the router drops it and sends
   ICMP Destination Unreachable: Fragmentation Needed (Type 3, Code 4) back to the sender.
   This is the mechanism behind Path MTU Discovery.

**1b.**

The router decrements TTL from 1 to 0, drops the packet, and sends an ICMP Time Exceeded
message (Type 11, Code 0) back to the source. The ICMP message includes the router's own
source IP address.

traceroute exploits this behavior. It sends a series of packets with TTL = 1, then TTL = 2,
then TTL = 3, and so on. Each router along the path that decrements TTL to 0 sends back an
ICMP Time Exceeded, revealing its IP address. By incrementing TTL one at a time, traceroute
maps out every hop along the path to a destination.

**1c.**

The TTL field is decremented at every hop, which changes the header contents and therefore
invalidates the checksum. The checksum must be recomputed after each TTL decrement so that
the next router can verify the header was not corrupted in transit. IPv6 removed the header
checksum because this per-hop recomputation is expensive at line rate, and it is largely
redundant — link-layer CRCs already catch bit errors per link, and transport-layer checksums
(TCP, UDP) catch end-to-end corruption. Eliminating the checksum simplifies router fast-path
processing, which is one of the reasons IPv6 headers can be processed faster.

**1d.**

Protocol = 17 means UDP. The receiving host's network layer reads the Protocol field and hands
the payload up to the UDP module. This is the demultiplexing step at Layer 3: the network
layer does not interpret the payload itself — it uses Protocol as a tag to direct the payload to
the correct Layer 4 handler.

---

## Problem 2 — IP Fragmentation

### Part (a)

A router needs to forward an IPv4 datagram with total length 5,000 bytes onto a link with
MTU = 1,500 bytes. The DF flag is not set. The datagram has Identification = 422.

Compute the three fragments. For each, give:
- Total Length
- MF flag value
- Fragment Offset (in 8-byte units)
- Which bytes of the original payload are carried

Show your work.

### Part (b)

Fragment 2 of the datagram from Part (a) is lost in transit. What happens at the destination?
What happens to the other two fragments that did arrive?

### Part (c)

What is Path MTU Discovery and why is it preferred over in-network fragmentation? What
can go wrong with PMTUD, and why?

### Part (d)

IPv6 does not allow routers to fragment packets in transit. If a packet is too large, what does
the router do instead? What does this mean for the sender?

---

**Solutions**

**2a.**

The original datagram: 5,000 bytes total = 20-byte header + 4,980 bytes of payload.

MTU = 1,500 bytes. Each fragment can carry at most 1,500 - 20 = 1,480 bytes of payload.
Fragment offsets are measured in 8-byte units, and 1,480 is divisible by 8, so the math is clean.

Fragment 1:
- Carries payload bytes 0 through 1,479 (1,480 bytes)
- Total Length = 1,480 + 20 = 1,500
- MF = 1 (more fragments follow)
- Fragment Offset = 0 / 8 = 0

Fragment 2:
- Carries payload bytes 1,480 through 2,959 (1,480 bytes)
- Total Length = 1,480 + 20 = 1,500
- MF = 1 (more fragments follow)
- Fragment Offset = 1,480 / 8 = 185

Fragment 3:
- Carries payload bytes 2,960 through 4,979 (2,020 bytes)
- Total Length = 2,020 + 20 = 2,040
- MF = 0 (last fragment)
- Fragment Offset = 2,960 / 8 = 370

All three fragments carry Identification = 422.

**2b.**

Reassembly happens only at the destination. The destination receives fragments 1 and 3,
buffers them, and waits for fragment 2 (offset 185). It never arrives. After a reassembly
timeout, the destination discards all buffered fragments for Identification = 422 and sends an
ICMP Time Exceeded: Fragment Reassembly Timeout (Type 11, Code 1) back toward the
source, if it has received enough to know the source address.

Fragments 1 and 3 are simply discarded — they consumed network bandwidth and destination
buffer space for nothing. The original datagram is lost in its entirety. If this is a TCP segment,
TCP will eventually time out and retransmit the original data.

**2c.**

Path MTU Discovery avoids fragmentation entirely by having the sender probe for the smallest
MTU along the path. The sender sets the DF flag on all packets. If any router on the path
cannot forward the packet without fragmenting it, the router drops the packet and sends ICMP
Destination Unreachable: Fragmentation Needed (Type 3, Code 4) back to the sender, including
the MTU of the problematic link. The sender then reduces its packet size and retransmits. This
repeats until the sender discovers the path MTU and uses it for all subsequent packets.

PMTUD is preferred because fragmentation is expensive and fragile: losing one fragment
causes the entire datagram to be retransmitted, reassembly requires buffering at the
destination, and fragmentation has historically been exploited in security attacks.

The failure mode for PMTUD is firewalls that block all ICMP traffic. If the ICMP Fragmentation
Needed message is blocked, the sender never learns that its packets are being dropped. The
connection stalls silently — the sender keeps sending large packets that get dropped, while
the receiver never sees them. This is called a PMTUD black hole. The fix is for network
operators to allow ICMP Type 3, Code 4 through their firewalls even if other ICMP types
are blocked.

**2d.**

In IPv6, if a router encounters a packet that is too large for the outgoing link, it drops the
packet and sends an ICMPv6 Packet Too Big message (Type 2) back to the source, including
the link's MTU. The router never fragments the packet itself.

This means the source is responsible for not sending packets larger than the path MTU. IPv6
senders use Path MTU Discovery (same concept as IPv4 PMTUD) to determine the appropriate
size. Only the source host may fragment in IPv6, using a Fragment extension header. The
benefit is that routers are simpler and faster because they never need to perform fragmentation
logic in the forwarding path.

---

## Problem 3 — NAT and Port Translation

### Part (a)

A NAT router has public IP address 203.0.113.5. Three internal hosts send packets outward:

| Internal Host | Internal Port | Destination         |
|---------------|---------------|---------------------|
| 10.0.0.10     | 4442          | 8.8.8.8:53          |
| 10.0.0.11     | 4442          | 8.8.8.8:53          |
| 10.0.0.12     | 8080          | 93.184.216.34:80    |

The NAT router assigns external ports 50001, 50002, 50003 in order.

Fill in the NAT translation table and show what the source IP and source port look like on
the public internet side for each packet.

### Part (b)

A reply arrives from 8.8.8.8:53 addressed to 203.0.113.5:50002. Trace exactly what the NAT
router does with this packet, step by step.

### Part (c)

Why can a single public IP address support approximately 65,000 simultaneous connections
under NAT? What is the practical limit and what determines it?

### Part (d)

A host at 10.0.0.20 behind the NAT router wants to run a web server on port 80 that is
accessible from the public internet. Why does NAT break this, and what is the standard
workaround?

---

**Solutions**

**3a.**

NAT translation table:

| WAN Side (public)        | LAN Side (private)    |
|--------------------------|-----------------------|
| 203.0.113.5:50001        | 10.0.0.10:4442        |
| 203.0.113.5:50002        | 10.0.0.11:4442        |
| 203.0.113.5:50003        | 10.0.0.12:8080        |

On the public internet side, all three packets appear to come from 203.0.113.5, distinguished
only by source port:
- 10.0.0.10 appears as 203.0.113.5:50001
- 10.0.0.11 appears as 203.0.113.5:50002
- 10.0.0.12 appears as 203.0.113.5:50003

Note that two internal hosts both used port 4442, but the NAT router assigned them different
external ports (50001 and 50002). This is why the port mapping exists — to demultiplex
replies back to the correct internal host even when internal port numbers collide.

**3b.**

The reply arrives: source = 8.8.8.8:53, destination = 203.0.113.5:50002.

1. The NAT router looks up destination port 50002 in the WAN-side column of its translation
   table.
2. It finds the entry: 203.0.113.5:50002 <-> 10.0.0.11:4442.
3. The router rewrites the destination IP from 203.0.113.5 to 10.0.0.11 and the destination
   port from 50002 to 4442.
4. It recomputes the IP header checksum (because the destination IP changed) and the UDP
   checksum (because the destination port changed).
5. It forwards the modified packet to 10.0.0.11 on the internal network.

**3c.**

Port numbers in TCP and UDP are 16-bit fields, giving a range of 0 to 65,535. The NAT router
assigns one external port number per active connection. With one public IP, there are at most
65,535 available port numbers, so approximately 65,000 simultaneous connections can be
tracked. In practice the limit is slightly lower because some port numbers are reserved and
the NAT table has a finite size in memory. The actual bottleneck in most deployments is the
NAT table memory and the router's ability to do table lookups at line rate, not the 16-bit port
space.

**3d.**

NAT only creates translation table entries for outgoing connections — connections that
originate from inside the private network. When an external host tries to reach 10.0.0.20:80,
it addresses the packet to the NAT router's public IP (203.0.113.5) on port 80. The NAT
router receives this packet but has no translation table entry telling it to forward port 80
traffic to 10.0.0.20. The packet is dropped.

The standard workaround is port forwarding: a static entry is manually added to the NAT
table mapping an external port on the public IP to a specific internal host and port (e.g.,
203.0.113.5:80 -> 10.0.0.20:80). The router then always forwards incoming traffic on that
external port to the designated internal host, even without an outgoing connection to trigger
the mapping.

---

## Problem 4 — IPv4 vs. IPv6 Header Comparison

### Part (a)

For each field that exists in the IPv4 header but was removed in IPv6, state the field name and
explain why it was removed.

### Part (b)

IPv6 addresses are 128 bits. Write the following address in compressed notation, showing each
compression step:

```
2001:0db8:0000:0000:0000:0000:0000:0001
```

Then expand this compressed address back to full form:

```
fe80::1
```

### Part (c)

An IPv6 packet arrives at a router. The packet is 2,000 bytes and the outgoing link has MTU
1,280 bytes (the IPv6 minimum). What does the router do?

### Part (d)

IPv6 has 2^128 addresses — enough that address exhaustion is essentially impossible. Yet
IPv6 still uses hierarchical prefix-based allocation and longest-prefix match, just like IPv4.
Explain why scarcity is not the reason for hierarchical addressing.

---

**Solutions**

**4a.**

Three fields were removed from IPv4 to create the IPv6 header:

1. Header Checksum — removed because it must be recomputed at every hop whenever TTL
   (renamed Hop Limit in IPv6) is decremented. This is expensive at high line rates. Link-layer
   CRCs catch per-link bit errors and end-to-end transport checksums catch corruption across
   the path, making the IP header checksum redundant.

2. Fragmentation fields (Identification, Flags, Fragment Offset) — removed from the base
   header because IPv6 does not allow routers to fragment packets in transit. If fragmentation
   is needed, it is handled by the source using an optional Fragment extension header. Removing
   these fields from the base header simplifies router processing for the common case where
   fragmentation is not needed.

3. IHL (Internet Header Length) and Options — the variable-length options field is gone.
   IPv6 uses a fixed 40-byte header with a chained extension header mechanism instead.
   Options are moved to extension headers that appear between the fixed header and the
   payload. The fixed header length eliminates the need for IHL.

**4b.**

Starting address: `2001:0db8:0000:0000:0000:0000:0000:0001`

Step 1 — drop leading zeros in each group:
`2001:db8:0:0:0:0:0:1`

Step 2 — replace the longest run of consecutive all-zero groups with `::`:
`2001:db8::1`

Expanding `fe80::1`:
`fe80` is the first group. `::` represents enough zero groups to make 8 total. There is one
explicit group before `::` (fe80) and one after (1), so `::` replaces 6 groups of zeros:
`fe80:0000:0000:0000:0000:0000:0000:0001`

**4c.**

The router cannot fragment the packet — IPv6 forbids in-transit fragmentation. The router
drops the packet and sends an ICMPv6 Packet Too Big message (Type 2) back to the source,
containing the MTU of the outgoing link (1,280 bytes). The source is then responsible for
reducing its packet size to at most 1,280 bytes and retransmitting using Path MTU Discovery.

**4d.**

Hierarchical addressing is about routing scalability, not address scarcity. Even with 2^128
addresses, a flat address space where every device had an unstructured globally unique address
would require every router to have one forwarding table entry per device. With billions of
connected devices, no router could store or search a table that large at line rate.

By allocating addresses hierarchically (IANA to RIR to ISP to organization), the entire address
block of a large ISP can be summarized in a single forwarding table entry at the global level.
Longest-prefix match then handles more specific routes closer to the destination. This keeps
the global routing table manageable regardless of how many devices exist. IPv6 inherits the
same approach because the routing scalability problem is independent of the address space size.

---

## Problem 5 — DHCP

### Part (a)

Trace the four-step DHCP process (DORA) for a new host joining a network. For each step,
state: the message name, who sends it, who receives it, the source IP, the destination IP, and
the transport protocol and ports used.

### Part (b)

Why does DHCP use UDP instead of TCP? Why does the initial Discover use 0.0.0.0 as the
source IP and 255.255.255.255 as the destination?

### Part (c)

A DHCP server is on a different subnet than the client. How does the DHCP Discover (a
broadcast) reach the server? What network device handles this, and what does it do?

### Part (d)

What is a DHCP lease, and why does it exist? What happens if a client's lease expires before
it renews?

### Part (e)

IPv6 introduces SLAAC as an alternative to DHCP. Briefly explain how SLAAC works and
what the key difference is from DHCPv6.

---

**Solutions**

**5a.**

DHCP DORA exchange:

1. Discover
   - Sent by: the client
   - Received by: all hosts on the local subnet (broadcast)
   - Source IP: 0.0.0.0 (client has no address yet)
   - Destination IP: 255.255.255.255 (limited broadcast)
   - Transport: UDP, source port 68, destination port 67

2. Offer
   - Sent by: the DHCP server
   - Received by: the client (may be unicast to client's MAC or broadcast)
   - Source IP: DHCP server's IP
   - Destination IP: 255.255.255.255 or the offered IP
   - Transport: UDP, source port 67, destination port 68
   - Contains: offered IP address, subnet mask, default gateway, DNS servers, lease duration

3. Request
   - Sent by: the client
   - Received by: all hosts (broadcast, so other DHCP servers know the offer was accepted)
   - Source IP: 0.0.0.0 (client still has not officially accepted the address)
   - Destination IP: 255.255.255.255
   - Transport: UDP, source port 68, destination port 67
   - Contains: the IP address the client is accepting, the server identifier

4. Ack
   - Sent by: the DHCP server
   - Received by: the client
   - Source IP: DHCP server's IP
   - Destination IP: 255.255.255.255 or the client's new IP
   - Transport: UDP, source port 67, destination port 68
   - Confirms the assignment and lease duration

**5b.**

DHCP cannot use TCP because TCP requires a three-way handshake, which requires both
endpoints to have IP addresses. The client has no IP address when it starts DHCP — that is
the entire point of the protocol. UDP allows the client to send datagrams without an
established connection.

The source IP is 0.0.0.0 because the client has no assigned IP yet and cannot put a valid
source address in the packet. The destination is 255.255.255.255 (limited broadcast) because
the client does not know the IP address of the DHCP server — it broadcasts the request to
all nodes on the local network, hoping a DHCP server is listening.

**5c.**

Broadcast messages do not cross router boundaries by default. A router receiving a broadcast
packet does not forward it to other subnets. To reach a DHCP server on a different subnet,
the local router (or a dedicated device) acts as a DHCP relay agent.

The relay agent receives the broadcast Discover on the local subnet, rewrites it as a unicast
packet addressed to the DHCP server's IP, and forwards it. When the server replies, the relay
agent receives the unicast reply and forwards it back to the client's subnet (either as a
broadcast or directly to the client's MAC address). The relay agent also fills in a "giaddr"
field so the server knows which subnet the client is on and can offer an address from the
right pool.

**5d.**

A DHCP lease is a time-limited assignment of an IP address to a specific client. The server
grants the address for a fixed duration (typically hours to days). The client must renew the
lease before it expires — it typically attempts renewal at the 50% mark of the lease duration.

Leases exist because IP addresses are a finite pool. If a device disconnects without releasing
its address (laptop closes, phone leaves the network), the address would be permanently
consumed without leases. With leases, addresses automatically return to the pool when the
lease expires without a renewal, allowing them to be reassigned to new devices.

If a client's lease expires before it renews, it must stop using the address immediately.
Continuing to use an expired address could conflict with a new device that has been assigned
the same address. The client must restart the DORA process to obtain a new address.

**5e.**

SLAAC (Stateless Address Auto-Configuration) allows an IPv6 host to configure its own
global address without a DHCP server. The router periodically broadcasts Router
Advertisement (RA) messages containing the network prefix (e.g., 2001:db8::/64). The host
takes that prefix and appends its own 64-bit host identifier, derived either from its MAC
address (EUI-64) or generated randomly for privacy. The result is a globally routable IPv6
address that required no server interaction.

The key difference from DHCPv6 is that SLAAC is stateless: no server tracks which address
was assigned to which host. DHCPv6 in stateful mode maintains a lease database like IPv4
DHCP, giving administrators visibility and control over address assignments. In practice, many
IPv6 networks use SLAAC for address assignment and DHCPv6 only for additional configuration
like DNS server addresses, since SLAAC alone cannot distribute DNS information.

---

## Problem 6 — Putting It Together

A user on a laptop connects to a coffee shop WiFi network and opens a browser to load
a page from a server at 93.184.216.34. The coffee shop uses a NAT router with public IP
198.51.100.7. The laptop gets address 192.168.0.55 via DHCP.

Trace the full sequence of events from the moment the laptop connects to the network until
the first IP packet from the laptop reaches the server. Include: DHCP, any relevant IPv4
header fields as the packet traverses the NAT router, and what the server sees as the source
of the connection.

---

**Solution**

1. The laptop connects to the WiFi access point and has no IP address. It sends a DHCP
   Discover via broadcast (src 0.0.0.0, dst 255.255.255.255, UDP port 68 -> 67).

2. The coffee shop's DHCP server (likely running on the NAT router itself) replies with a
   DHCP Offer containing an available address from the private pool — say 192.168.0.55 —
   along with the subnet mask (255.255.255.0), default gateway (192.168.0.1), and DNS
   server address.

3. The laptop sends a DHCP Request accepting the offer, and the server replies with a DHCP
   Ack. The laptop configures itself: IP = 192.168.0.55, gateway = 192.168.0.1.

4. The user opens a browser. A DNS lookup resolves the hostname to 93.184.216.34 (this
   uses a separate UDP exchange to the DNS server). The browser initiates a TCP connection
   to 93.184.216.34:80.

5. The laptop sends the first TCP SYN packet:
   - Source IP: 192.168.0.55, Source Port: (some ephemeral port, say 51234)
   - Destination IP: 93.184.216.34, Destination Port: 80
   - TTL: 64 (Linux default), Protocol: 6 (TCP)

6. The packet arrives at the NAT router (192.168.0.1). The router:
   - Replaces Source IP 192.168.0.55 with its public IP 198.51.100.7
   - Replaces Source Port 51234 with a new external port (say 60001)
   - Records the mapping: 198.51.100.7:60001 <-> 192.168.0.55:51234 in its NAT table
   - Recomputes the IP header checksum and TCP checksum
   - Forwards the packet toward 93.184.216.34

7. The server at 93.184.216.34 receives a TCP SYN with:
   - Source IP: 198.51.100.7 (the NAT router's public IP)
   - Source Port: 60001
   - Destination IP: 93.184.216.34
   - Destination Port: 80

   The server has no knowledge of 192.168.0.55. From its perspective, the connection is
   coming from 198.51.100.7. All replies go back to 198.51.100.7:60001, and the NAT router
   translates them back to 192.168.0.55:51234 before delivering them to the laptop.

---

*EC 441 – Boston University, Spring 2026*