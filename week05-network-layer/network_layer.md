# Lab: Network Layer — IP Addressing, CIDR, Subnetting & Forwarding
## Solved

**Course:** EC 441 – Introduction to Computer Networking, Spring 2026
**Topic:** Network Layer (Lectures 13 & 14)

---

## Part 1 — Subnet Arithmetic by Hand

### 1a. `10.20.30.0/26`

A /26 means 26 bits are the network portion, leaving 32 - 26 = 6 bits for hosts.

**Subnet mask:**
26 ones followed by 6 zeros: `11111111.11111111.11111111.11000000`
Last octet: 11000000 = 128 + 64 = 192
Subnet mask: `255.255.255.192`

**Total addresses:** 2^6 = 64

**Usable hosts:** 64 - 2 = 62 (subtract network address and broadcast)

**Network address:** `10.20.30.0` (host bits all zero, already given)

**Broadcast address:** Set all 6 host bits to 1. Last octet becomes `00111111` = 63.
Broadcast: `10.20.30.63`

**Valid host range:** `10.20.30.1` through `10.20.30.62`

---

### 1b. `172.16.5.200/21`

A /21 means 21 bits are network, 11 bits are host. The split falls inside the third octet:
- First two octets: all network bits (16 bits)
- Third octet: 5 network bits + 3 host bits
- Fourth octet: all 8 host bits

**Subnet mask:**
Third octet: top 5 bits = `11111000` = 248
Mask: `255.255.248.0`

**Network address** (AND the address with the mask):
```
172  .  16  .  5   .  200
255  . 255  . 248  .    0
--------------------------
172  .  16  .  0   .    0
```
Third octet: `00000101` AND `11111000` = `00000000` = 0
Network address: `172.16.0.0`

**Broadcast address:** Set all 11 host bits to 1.
- Third octet: `00000111` = 7
- Fourth octet: `11111111` = 255
Broadcast: `172.16.7.255`

**Is `172.16.12.1` in the same subnet?**

AND `172.16.12.1` with `255.255.248.0`:
- Third octet: `00001100` AND `11111000` = `00001000` = 8
- Network of `172.16.12.1` is `172.16.8.0`

`172.16.8.0` != `172.16.0.0`, so `172.16.12.1` is NOT in the same subnet.

---

### 1c. Divide `192.168.100.0/24` into 4 equal subnets

To get 4 subnets we need 2 extra bits (2^2 = 4), so the new prefix is /24 + 2 = /26.

Each /26 has 2^(32-26) = 64 addresses, 62 usable hosts.

| Subnet | Network Address      | Host Range                           | Broadcast         |
|--------|----------------------|--------------------------------------|-------------------|
| 1      | 192.168.100.0/26     | 192.168.100.1 - 192.168.100.62       | 192.168.100.63    |
| 2      | 192.168.100.64/26    | 192.168.100.65 - 192.168.100.126     | 192.168.100.127   |
| 3      | 192.168.100.128/26   | 192.168.100.129 - 192.168.100.190    | 192.168.100.191   |
| 4      | 192.168.100.192/26   | 192.168.100.193 - 192.168.100.254    | 192.168.100.255   |

Usable hosts per subnet: 62

---

## Part 2 — Python Verification

### Starter code output

```
=== 10.20.30.0/26 ===
Subnet mask:       255.255.255.192
Total addresses:   64
Usable hosts:      62
Network address:   10.20.30.0
Broadcast address: 10.20.30.63
First usable host: 10.20.30.1
Last usable host:  10.20.30.62

=== 172.16.5.200/21 ===
Network address:   172.16.0.0
Broadcast address: 172.16.7.255
172.16.12.1 in same subnet? False

=== 192.168.100.0/24 split into 4 subnets ===
Subnet 1: 192.168.100.0/26  (usable hosts: 62)
Subnet 2: 192.168.100.64/26  (usable hosts: 62)
Subnet 3: 192.168.100.128/26  (usable hosts: 62)
Subnet 4: 192.168.100.192/26  (usable hosts: 62)
```

All hand calculations match. No discrepancies.

---

### 2b. Split Subnet 2 into two halves

Subnet 2 is `192.168.100.64/26`. Splitting in half adds one more bit: /26 + 1 = /27.
Used `prefixlen_diff=1`.

```python
subnet2 = subnets[1]  # 192.168.100.64/26
halves = list(subnet2.subnets(prefixlen_diff=1))
```

Output:
```
192.168.100.64/27    (hosts: 192.168.100.65 - 192.168.100.94,  30 usable)
192.168.100.96/27    (hosts: 192.168.100.97 - 192.168.100.126, 30 usable)
```

---

### 2c. Loop over usable hosts in `10.20.30.0/26`

```python
for h in ipaddress.ip_network("10.20.30.0/26").hosts():
    print(h)
```

Printed 62 lines, from `10.20.30.1` to `10.20.30.62`. Matches the hand calculation: 2^6 - 2 = 62.

---

## Part 3 — Longest-Prefix Match

### 3a. Forwarding table lookup by hand

Forwarding table:
| Prefix         | Length |
|----------------|--------|
| 0.0.0.0/0      | /0     |
| 10.0.0.0/8     | /8     |
| 10.1.0.0/16    | /16    |
| 10.1.2.0/24    | /24    |
| 192.168.0.0/16 | /16    |

| Destination   | All Matching Prefixes                                  | Longest Match  | Interface |
|---------------|--------------------------------------------------------|----------------|-----------|
| 10.1.2.5      | 0.0.0.0/0, 10.0.0.0/8, 10.1.0.0/16, 10.1.2.0/24      | 10.1.2.0/24    | eth3      |
| 10.5.6.7      | 0.0.0.0/0, 10.0.0.0/8                                 | 10.0.0.0/8     | eth1      |
| 10.1.100.50   | 0.0.0.0/0, 10.0.0.0/8, 10.1.0.0/16                    | 10.1.0.0/16    | eth2      |
| 172.16.0.1    | 0.0.0.0/0                                              | 0.0.0.0/0      | eth0      |
| 192.168.50.3  | 0.0.0.0/0, 192.168.0.0/16                              | 192.168.0.0/16 | eth4      |

Reasoning:
- `10.1.2.5` matches /8, /16, and /24. Most specific wins: /24 -> eth3.
- `10.5.6.7` matches /8 but not /16 (second octet is 5, not 1). Best is /8 -> eth1.
- `10.1.100.50` matches /8 and /16 but not /24 (third octet is 100, not 2). Best is /16 -> eth2.
- `172.16.0.1` only matches the default route -> eth0.
- `192.168.50.3` matches /16 for 192.168.0.0 -> eth4.

---

### 3b. Python LPM output

```
10.1.2.5             | matches: ['0.0.0.0/0', '10.0.0.0/8', '10.1.0.0/16', '10.1.2.0/24'] | winner: 10.1.2.0/24    | eth3
10.5.6.7             | matches: ['0.0.0.0/0', '10.0.0.0/8']                                | winner: 10.0.0.0/8     | eth1
10.1.100.50          | matches: ['0.0.0.0/0', '10.0.0.0/8', '10.1.0.0/16']                 | winner: 10.1.0.0/16    | eth2
172.16.0.1           | matches: ['0.0.0.0/0']                                               | winner: 0.0.0.0/0      | eth0
192.168.50.3         | matches: ['0.0.0.0/0', '192.168.0.0/16']                            | winner: 192.168.0.0/16 | eth4
```

Matches hand calculations.

---

### 3c. No default route — graceful handling

Without `0.0.0.0/0`, the lookup for `172.16.0.1` finds zero matching entries and `max()` raises
a `ValueError`. Fixed by checking if the match list is empty before calling max:

```python
def longest_prefix_match_safe(dest_str, table):
    dest = ipaddress.ip_address(dest_str)
    matching = [(prefix, iface) for prefix, iface in table if dest in prefix]
    if not matching:
        return None, None
    best_prefix, best_iface = max(matching, key=lambda x: x[0].prefixlen)
    return best_prefix, best_iface
```

A real router drops the packet and sends an ICMP Destination Unreachable message back to the
sender. This is why having a default route matters — without one, any traffic to an unknown
destination gets dropped.

---

## Part 4 — Reading Your Own Routing Table

Output from `ip route show`:

```
default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.42 metric 100
10.0.0.0/8 via 10.1.0.1 dev eth1 proto static
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.42
```

Annotations:

- `default via 192.168.1.1 dev eth0 proto dhcp` — the default route (0.0.0.0/0). Any packet
  with no more specific match goes to gateway 192.168.1.1 via eth0. Installed automatically
  by DHCP.

- `10.0.0.0/8 via 10.1.0.1 dev eth1 proto static` — traffic to the 10.x.x.x range goes to
  10.1.0.1 via eth1. `proto static` means an admin added this manually, not a routing daemon.

- `192.168.1.0/24 dev eth0 proto kernel scope link` — the local subnet. Hosts in 192.168.1.x
  are directly reachable on eth0 with no gateway needed. `scope link` means this is a directly
  connected network.

**4b.** Default route is `default via 192.168.1.1`. All unmatched traffic exits through the
home router at 192.168.1.1.

**4c.** `192.168.1.0/24 dev eth0 scope link` is a directly-connected route. Directly connected
means the router can reach these hosts without forwarding to a gateway — they are on the same
physical link, so the router uses ARP to resolve the MAC address and delivers the frame directly.

**4d.** Yes, 192.168.1.x and 10.x.x.x are both RFC 1918 private addresses. This tells you the
machine is behind NAT. These addresses are not globally routable — traffic to the internet gets
its source IP translated to a public address by the router.

---

## Part 5 — Route Aggregation

### 5a. Python verification

```
172.20.0.0/22 subnet of 172.20.0.0/20? True
172.20.4.0/22 subnet of 172.20.0.0/20? True
172.20.8.0/22 subnet of 172.20.0.0/20? True
172.20.12.0/22 subnet of 172.20.0.0/20? True
```

All four /22 customer blocks are contained within the ISP's /20 aggregate. A /20 covers
2^12 = 4096 addresses (172.20.0.0 through 172.20.15.255). Four /22s times 1024 addresses each
equals exactly 4096 — they fit because the ISP carved the /20 into four equal aligned pieces.

---

### 5b. Routing table savings

Without aggregation: 4 entries (one per /22).
With aggregation: 1 entry (172.20.0.0/20).
Savings: 3 routing table entries.

At internet scale this matters a lot. The global BGP table has around 900,000 prefixes today.
Without CIDR aggregation it would be orders of magnitude larger and would not fit in router
TCAM memory.

---

### 5c. LPM handles Beta's /24

When traffic destined for 172.20.5.10 leaves the internet:
1. The internet router only knows 172.20.0.0/20, so it forwards toward the ISP.
2. The ISP router knows 172.20.4.0/22 (Beta's allocation) and forwards toward Beta.
3. Beta's own router has both 172.20.4.0/22 and 172.20.5.0/24 in its table.
4. LPM picks /24 (longer, more specific) over /22, so the packet reaches the server farm.

Each layer only needs to know what is below it. The internet does not need to know about /24s
inside Beta's network — Beta's router handles it locally. That is the point of hierarchical
addressing.

---

### 5d. Aggregating four /24s

Third octets in binary:
- 4 = 00000100
- 5 = 00000101
- 6 = 00000110
- 7 = 00000111

All four share the prefix `000001` in the third octet (6 bits). Combined with 16 bits for the
first two octets, the aggregate prefix is 22 bits: `192.168.4.0/22`.

Python output:
```
Collapsed: [IPv4Network('192.168.4.0/22')]
```

Yes, the four /24s aggregate cleanly into `192.168.4.0/22`, saving 3 routing table entries.

---

## Reflection Questions

### R1. Forwarding vs. routing

Think of a large hospital. Routing is the work administrators do when they create the hospital
directory — mapping every department and room into a logical floor plan. Forwarding is what
the orderly does when they receive a patient: they check the wristband, glance at the floor
directory, and wheel the patient to the right room. The orderly does not redesign the layout —
they just follow the current map as fast as possible. Routing builds the map; forwarding
follows it, one packet at a time.

### R2. Why CIDR requires LPM

CIDR's power comes from hierarchical aggregation: an ISP advertises one short prefix (/20)
that covers many longer customer prefixes (/22, /24). A destination address can therefore
match multiple entries at different levels of the hierarchy at the same time.

If routers used shortest-prefix match instead, they would always pick the most general entry —
0.0.0.0/0 for nearly everything. Traffic that should go to a specific customer subnet would be
misrouted to the default gateway. The specificity of longer prefixes would be ignored entirely,
breaking the hierarchy. LPM is what makes CIDR actually work in practice.

### R3. Why IPv6 still uses hierarchical prefix-based addressing

Hierarchical addressing is not about scarcity — it is about routing scalability. Even with
2^128 addresses, if every device had a flat unstructured address, the global routing table
would need one entry per device. That is billions of entries, which no router could store or
search fast enough.

By allocating addresses in hierarchical blocks (IANA to RIR to ISP to organization), the
entire address space of a large ISP can be summarized in a single routing table entry at the
global level. LPM resolves more specific paths as packets travel closer to their destination.
The problem CIDR and LPM solve is not running out of addresses — it is that routers cannot
store and search a billion-entry table in nanoseconds. IPv6 inherits the same solution because
the routing scalability problem has not changed.

---

*EC 441 – Boston University, Spring 2026*

# Lab: Network Layer — IP Addressing, CIDR, Subnetting & Forwarding
## Solved

**Course:** EC 441 – Introduction to Computer Networking, Spring 2026
**Topic:** Network Layer (Lectures 13 & 14)

---

## Part 1 — Subnet Arithmetic by Hand

### 1a. `10.20.30.0/26`

A /26 means 26 bits are the network portion, leaving 32 - 26 = 6 bits for hosts.

**Subnet mask:**
26 ones followed by 6 zeros: `11111111.11111111.11111111.11000000`
Last octet: 11000000 = 128 + 64 = 192
Subnet mask: `255.255.255.192`

**Total addresses:** 2^6 = 64

**Usable hosts:** 64 - 2 = 62 (subtract network address and broadcast)

**Network address:** `10.20.30.0` (host bits all zero, already given)

**Broadcast address:** Set all 6 host bits to 1. Last octet becomes `00111111` = 63.
Broadcast: `10.20.30.63`

**Valid host range:** `10.20.30.1` through `10.20.30.62`

---

### 1b. `172.16.5.200/21`

A /21 means 21 bits are network, 11 bits are host. The split falls inside the third octet:
- First two octets: all network bits (16 bits)
- Third octet: 5 network bits + 3 host bits
- Fourth octet: all 8 host bits

**Subnet mask:**
Third octet: top 5 bits = `11111000` = 248
Mask: `255.255.248.0`

**Network address** (AND the address with the mask):
```
172  .  16  .  5   .  200
255  . 255  . 248  .    0
--------------------------
172  .  16  .  0   .    0
```
Third octet: `00000101` AND `11111000` = `00000000` = 0
Network address: `172.16.0.0`

**Broadcast address:** Set all 11 host bits to 1.
- Third octet: `00000111` = 7
- Fourth octet: `11111111` = 255
Broadcast: `172.16.7.255`

**Is `172.16.12.1` in the same subnet?**

AND `172.16.12.1` with `255.255.248.0`:
- Third octet: `00001100` AND `11111000` = `00001000` = 8
- Network of `172.16.12.1` is `172.16.8.0`

`172.16.8.0` != `172.16.0.0`, so `172.16.12.1` is NOT in the same subnet.

---

### 1c. Divide `192.168.100.0/24` into 4 equal subnets

To get 4 subnets we need 2 extra bits (2^2 = 4), so the new prefix is /24 + 2 = /26.

Each /26 has 2^(32-26) = 64 addresses, 62 usable hosts.

| Subnet | Network Address      | Host Range                           | Broadcast         |
|--------|----------------------|--------------------------------------|-------------------|
| 1      | 192.168.100.0/26     | 192.168.100.1 - 192.168.100.62       | 192.168.100.63    |
| 2      | 192.168.100.64/26    | 192.168.100.65 - 192.168.100.126     | 192.168.100.127   |
| 3      | 192.168.100.128/26   | 192.168.100.129 - 192.168.100.190    | 192.168.100.191   |
| 4      | 192.168.100.192/26   | 192.168.100.193 - 192.168.100.254    | 192.168.100.255   |

Usable hosts per subnet: 62

---

## Part 2 — Python Verification

### Starter code output

```
=== 10.20.30.0/26 ===
Subnet mask:       255.255.255.192
Total addresses:   64
Usable hosts:      62
Network address:   10.20.30.0
Broadcast address: 10.20.30.63
First usable host: 10.20.30.1
Last usable host:  10.20.30.62

=== 172.16.5.200/21 ===
Network address:   172.16.0.0
Broadcast address: 172.16.7.255
172.16.12.1 in same subnet? False

=== 192.168.100.0/24 split into 4 subnets ===
Subnet 1: 192.168.100.0/26  (usable hosts: 62)
Subnet 2: 192.168.100.64/26  (usable hosts: 62)
Subnet 3: 192.168.100.128/26  (usable hosts: 62)
Subnet 4: 192.168.100.192/26  (usable hosts: 62)
```

All hand calculations match. No discrepancies.

---

### 2b. Split Subnet 2 into two halves

Subnet 2 is `192.168.100.64/26`. Splitting in half adds one more bit: /26 + 1 = /27.
Used `prefixlen_diff=1`.

```python
subnet2 = subnets[1]  # 192.168.100.64/26
halves = list(subnet2.subnets(prefixlen_diff=1))
```

Output:
```
192.168.100.64/27    (hosts: 192.168.100.65 - 192.168.100.94,  30 usable)
192.168.100.96/27    (hosts: 192.168.100.97 - 192.168.100.126, 30 usable)
```

---

### 2c. Loop over usable hosts in `10.20.30.0/26`

```python
for h in ipaddress.ip_network("10.20.30.0/26").hosts():
    print(h)
```

Printed 62 lines, from `10.20.30.1` to `10.20.30.62`. Matches the hand calculation: 2^6 - 2 = 62.

---

## Part 3 — Longest-Prefix Match

### 3a. Forwarding table lookup by hand

Forwarding table:
| Prefix         | Length |
|----------------|--------|
| 0.0.0.0/0      | /0     |
| 10.0.0.0/8     | /8     |
| 10.1.0.0/16    | /16    |
| 10.1.2.0/24    | /24    |
| 192.168.0.0/16 | /16    |

| Destination   | All Matching Prefixes                                  | Longest Match  | Interface |
|---------------|--------------------------------------------------------|----------------|-----------|
| 10.1.2.5      | 0.0.0.0/0, 10.0.0.0/8, 10.1.0.0/16, 10.1.2.0/24      | 10.1.2.0/24    | eth3      |
| 10.5.6.7      | 0.0.0.0/0, 10.0.0.0/8                                 | 10.0.0.0/8     | eth1      |
| 10.1.100.50   | 0.0.0.0/0, 10.0.0.0/8, 10.1.0.0/16                    | 10.1.0.0/16    | eth2      |
| 172.16.0.1    | 0.0.0.0/0                                              | 0.0.0.0/0      | eth0      |
| 192.168.50.3  | 0.0.0.0/0, 192.168.0.0/16                              | 192.168.0.0/16 | eth4      |

Reasoning:
- `10.1.2.5` matches /8, /16, and /24. Most specific wins: /24 -> eth3.
- `10.5.6.7` matches /8 but not /16 (second octet is 5, not 1). Best is /8 -> eth1.
- `10.1.100.50` matches /8 and /16 but not /24 (third octet is 100, not 2). Best is /16 -> eth2.
- `172.16.0.1` only matches the default route -> eth0.
- `192.168.50.3` matches /16 for 192.168.0.0 -> eth4.

---

### 3b. Python LPM output

```
10.1.2.5             | matches: ['0.0.0.0/0', '10.0.0.0/8', '10.1.0.0/16', '10.1.2.0/24'] | winner: 10.1.2.0/24    | eth3
10.5.6.7             | matches: ['0.0.0.0/0', '10.0.0.0/8']                                | winner: 10.0.0.0/8     | eth1
10.1.100.50          | matches: ['0.0.0.0/0', '10.0.0.0/8', '10.1.0.0/16']                 | winner: 10.1.0.0/16    | eth2
172.16.0.1           | matches: ['0.0.0.0/0']                                               | winner: 0.0.0.0/0      | eth0
192.168.50.3         | matches: ['0.0.0.0/0', '192.168.0.0/16']                            | winner: 192.168.0.0/16 | eth4
```

Matches hand calculations.

---

### 3c. No default route — graceful handling

Without `0.0.0.0/0`, the lookup for `172.16.0.1` finds zero matching entries and `max()` raises
a `ValueError`. Fixed by checking if the match list is empty before calling max:

```python
def longest_prefix_match_safe(dest_str, table):
    dest = ipaddress.ip_address(dest_str)
    matching = [(prefix, iface) for prefix, iface in table if dest in prefix]
    if not matching:
        return None, None
    best_prefix, best_iface = max(matching, key=lambda x: x[0].prefixlen)
    return best_prefix, best_iface
```

A real router drops the packet and sends an ICMP Destination Unreachable message back to the
sender. This is why having a default route matters — without one, any traffic to an unknown
destination gets dropped.

---

## Part 4 — Reading Your Own Routing Table

Output from `ip route show`:

```
default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.42 metric 100
10.0.0.0/8 via 10.1.0.1 dev eth1 proto static
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.42
```

Annotations:

- `default via 192.168.1.1 dev eth0 proto dhcp` — the default route (0.0.0.0/0). Any packet
  with no more specific match goes to gateway 192.168.1.1 via eth0. Installed automatically
  by DHCP.

- `10.0.0.0/8 via 10.1.0.1 dev eth1 proto static` — traffic to the 10.x.x.x range goes to
  10.1.0.1 via eth1. `proto static` means an admin added this manually, not a routing daemon.

- `192.168.1.0/24 dev eth0 proto kernel scope link` — the local subnet. Hosts in 192.168.1.x
  are directly reachable on eth0 with no gateway needed. `scope link` means this is a directly
  connected network.

**4b.** Default route is `default via 192.168.1.1`. All unmatched traffic exits through the
home router at 192.168.1.1.

**4c.** `192.168.1.0/24 dev eth0 scope link` is a directly-connected route. Directly connected
means the router can reach these hosts without forwarding to a gateway — they are on the same
physical link, so the router uses ARP to resolve the MAC address and delivers the frame directly.

**4d.** Yes, 192.168.1.x and 10.x.x.x are both RFC 1918 private addresses. This tells you the
machine is behind NAT. These addresses are not globally routable — traffic to the internet gets
its source IP translated to a public address by the router.

---

## Part 5 — Route Aggregation

### 5a. Python verification

```
172.20.0.0/22 subnet of 172.20.0.0/20? True
172.20.4.0/22 subnet of 172.20.0.0/20? True
172.20.8.0/22 subnet of 172.20.0.0/20? True
172.20.12.0/22 subnet of 172.20.0.0/20? True
```

All four /22 customer blocks are contained within the ISP's /20 aggregate. A /20 covers
2^12 = 4096 addresses (172.20.0.0 through 172.20.15.255). Four /22s times 1024 addresses each
equals exactly 4096 — they fit because the ISP carved the /20 into four equal aligned pieces.

---

### 5b. Routing table savings

Without aggregation: 4 entries (one per /22).
With aggregation: 1 entry (172.20.0.0/20).
Savings: 3 routing table entries.

At internet scale this matters a lot. The global BGP table has around 900,000 prefixes today.
Without CIDR aggregation it would be orders of magnitude larger and would not fit in router
TCAM memory.

---

### 5c. LPM handles Beta's /24

When traffic destined for 172.20.5.10 leaves the internet:
1. The internet router only knows 172.20.0.0/20, so it forwards toward the ISP.
2. The ISP router knows 172.20.4.0/22 (Beta's allocation) and forwards toward Beta.
3. Beta's own router has both 172.20.4.0/22 and 172.20.5.0/24 in its table.
4. LPM picks /24 (longer, more specific) over /22, so the packet reaches the server farm.

Each layer only needs to know what is below it. The internet does not need to know about /24s
inside Beta's network — Beta's router handles it locally. That is the point of hierarchical
addressing.

---

### 5d. Aggregating four /24s

Third octets in binary:
- 4 = 00000100
- 5 = 00000101
- 6 = 00000110
- 7 = 00000111

All four share the prefix `000001` in the third octet (6 bits). Combined with 16 bits for the
first two octets, the aggregate prefix is 22 bits: `192.168.4.0/22`.

Python output:
```
Collapsed: [IPv4Network('192.168.4.0/22')]
```

Yes, the four /24s aggregate cleanly into `192.168.4.0/22`, saving 3 routing table entries.

---

## Reflection Questions

### R1. Forwarding vs. routing

Think of a large hospital. Routing is the work administrators do when they create the hospital
directory — mapping every department and room into a logical floor plan. Forwarding is what
the orderly does when they receive a patient: they check the wristband, glance at the floor
directory, and wheel the patient to the right room. The orderly does not redesign the layout —
they just follow the current map as fast as possible. Routing builds the map; forwarding
follows it, one packet at a time.

### R2. Why CIDR requires LPM

CIDR's power comes from hierarchical aggregation: an ISP advertises one short prefix (/20)
that covers many longer customer prefixes (/22, /24). A destination address can therefore
match multiple entries at different levels of the hierarchy at the same time.

If routers used shortest-prefix match instead, they would always pick the most general entry —
0.0.0.0/0 for nearly everything. Traffic that should go to a specific customer subnet would be
misrouted to the default gateway. The specificity of longer prefixes would be ignored entirely,
breaking the hierarchy. LPM is what makes CIDR actually work in practice.

### R3. Why IPv6 still uses hierarchical prefix-based addressing

Hierarchical addressing is not about scarcity — it is about routing scalability. Even with
2^128 addresses, if every device had a flat unstructured address, the global routing table
would need one entry per device. That is billions of entries, which no router could store or
search fast enough.

By allocating addresses in hierarchical blocks (IANA to RIR to ISP to organization), the
entire address space of a large ISP can be summarized in a single routing table entry at the
global level. LPM resolves more specific paths as packets travel closer to their destination.
The problem CIDR and LPM solve is not running out of addresses — it is that routers cannot
store and search a billion-entry table in nanoseconds. IPv6 inherits the same solution because
the routing scalability problem has not changed.

---

*EC 441 – Boston University, Spring 2026*