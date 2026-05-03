# Problem Set: Autonomous Systems and BGP

**Course:** EC 441 – Introduction to Computer Networking, Spring 2026
**Topic:** Autonomous Systems, BGP, Inter-AS vs. Intra-AS Routing
**Type:** Problem

---

## Background

The internet is not one network — it is tens of thousands of independently administered
networks (autonomous systems) stitched together by BGP. Within each AS, a protocol like
OSPF finds shortest paths. Between ASes, BGP enforces routing policy based on business
relationships. These problems cover AS classification, BGP path-vector mechanics, the
valley-free rule, loop prevention, and real-world BGP incidents.

---

## Problem 1 — AS Classification

### Part (a)

Classify each of the following as a stub AS, multi-homed AS, or transit AS. Justify each answer.

1. A small startup with one upstream ISP connection
2. Boston University, which connects to both Internet2 and a commercial ISP
3. Comcast, which carries traffic between its customers and the rest of the internet
4. A cloud provider like AWS, which peers with hundreds of other networks and carries
   traffic for thousands of customers

### Part (b)

A stub AS and a transit AS are both connected to the same upstream provider. Explain why
the stub AS would never appear in the AS-PATH of a route advertisement that the provider
sends to other ASes, but the transit AS would.

### Part (c)

AS numbers were originally 16-bit. Why was this eventually a problem, and what was the
solution?

---

**Solutions**

**1a.**

1. Stub AS. A single upstream connection means there is no path through this AS to any
   other network. Traffic either originates here or is destined here — no transit.

2. Multi-homed AS. BU has multiple upstream connections (Internet2 and a commercial ISP)
   for redundancy, but BU does not carry transit traffic between those two providers on behalf
   of the rest of the internet.

3. Transit AS. Comcast carries traffic between its customers and other networks. Packets
   that neither originate nor terminate at Comcast still pass through it. Comcast appears as
   an intermediate hop in many AS-PATHs.

4. Transit AS. AWS peers broadly and carries traffic for customers. It also connects to many
   other large networks. Even though AWS has its own customer traffic, it is large enough to
   serve as a transit path for some traffic flows.

**1b.**

A stub AS has no downstream customers for the rest of the internet to reach through it. The
only traffic it handles is its own. A provider has no reason to advertise routes through a stub
because there is nothing useful on the other side. A transit AS, by definition, has downstream
customers whose prefixes need to be reachable from the broader internet. The provider
advertises those customer routes onward, and the transit AS appears in the resulting AS-PATH
as the path from any external network goes: external -> provider -> transit AS -> end customer.

**1c.**

16-bit ASNs allow for 65,536 unique values. As the internet grew to tens of thousands of
independently administered networks — ISPs, universities, cloud providers, enterprises — the
16-bit space began to run out, similar to the IPv4 address exhaustion problem. The solution
was to extend ASNs to 32 bits (RFC 4893), allowing approximately 4 billion unique ASNs.
The 32-bit space is divided so that the original 16-bit values remain valid, preserving
backwards compatibility.

---

## Problem 2 — Why the Internet Cannot Use OSPF Globally

### Part (a)

Name and briefly explain the three reasons why running a single link-state protocol (like OSPF)
across the entire internet is not feasible.

### Part (b)

OSPF requires every router to store the complete link-state database and flood LSAs when
topology changes. The internet currently has over 75,000 autonomous systems and hundreds
of millions of IP prefixes. Estimate concretely why the flooding overhead alone would be
unworkable. You do not need exact numbers — reason through the scaling behavior.

### Part (c)

Even within a single large AS, OSPF is sometimes divided into areas. Why? What does this
tell you about the scaling limits of link-state routing even in a single organization?

---

**Solutions**

**2a.**

1. Scale. OSPF requires every router to maintain a complete topology map and rerun Dijkstra
   whenever anything changes. With millions of routers and links across the internet, the
   link-state database would be enormous, flooding would consume enormous bandwidth, and
   the compute cost of running Dijkstra on a graph that large would be prohibitive.

2. Administrative autonomy. Different organizations have different goals and no obligation to
   share their internal topology with anyone else. A company's internal router layout is
   proprietary. OSPF requires full topology sharing — that is a non-starter across
   organizational boundaries.

3. Business relationships. Internet routing is driven by contracts and economics, not shortest
   paths. An ISP may refuse to carry a competitor's traffic even if doing so would produce a
   shorter path. OSPF has no mechanism for expressing "do not route through this link for
   policy reasons." BGP is built around policy; OSPF is not.

**2b.**

In OSPF, every link-state change triggers a flood. Each LSA must reach every router in the
routing domain. If there are millions of routers, one link failure means millions of LSA
messages. With 75,000 ASes each potentially containing hundreds or thousands of routers,
the number of routers in a global OSPF domain would be in the tens of millions. A single
busy period with frequent link flaps could generate billions of LSA messages per second. No
router hardware could process that flood while simultaneously forwarding traffic at line rate.
The bandwidth consumed by control-plane flooding alone would crowd out actual data traffic.

**2c.**

Even within one AS, OSPF is divided into areas (typically a backbone area 0 and stub areas)
specifically to limit the size of the link-state database each router must store and the scope
of flooding. Routers only need the full topology for their own area; inter-area routing is
summarized. This is direct evidence that full link-state does not scale past a few hundred
routers without hierarchical decomposition. The internet has orders of magnitude more routers,
which is why a fundamentally different approach (BGP) is needed at the inter-AS level.

---

## Problem 3 — BGP Path Vector and Loop Prevention

### Part (a)

Consider the following AS topology:

```
AS 100 (university) -- AS 200 (ISP A) -- AS 300 (ISP B) -- AS 400 (content provider)
                                |
                             AS 500 (ISP C)
                                |
                             AS 400 (content provider)
```

AS 400 originates the prefix 1.2.3.0/24. Trace the BGP route advertisements as they propagate
from AS 400 to AS 100. What AS-PATH does AS 100 ultimately see for this prefix? (Assume
AS 200 prefers the path through AS 300 over the path through AS 500.)

### Part (b)

Suppose AS 300 receives the prefix advertisement from AS 400 with AS-PATH [400]. AS 300
then sends this advertisement to AS 200. AS 200 receives it with AS-PATH [300, 400] and
forwards it to AS 100. AS 100 receives it with AS-PATH [200, 300, 400].

Now AS 300 receives a route advertisement for the same prefix from AS 200 with AS-PATH
[100, 200, 300, 400]. What does AS 300 do with this advertisement, and why?

### Part (c)

In distance-vector routing, loop prevention relies on split horizon or poisoned reverse, which
only reliably prevents two-node loops. Why is AS-PATH loop prevention in BGP fundamentally
more robust than split horizon?

---

**Solutions**

**3a.**

Starting from AS 400:
- AS 400 originates the prefix and advertises to its neighbors with AS-PATH [400]
- AS 300 receives it, prepends its own ASN, advertises to AS 200 with AS-PATH [300, 400]
- AS 500 also receives it from AS 400, prepends its own ASN, advertises to AS 200 with
  AS-PATH [500, 400]
- AS 200 receives both paths. It prefers the path through AS 300 (given in the problem).
  AS 200 prepends its own ASN and advertises to AS 100 with AS-PATH [200, 300, 400]
- AS 100 receives the advertisement with AS-PATH [200, 300, 400]

**3b.**

AS 300 receives the advertisement with AS-PATH [100, 200, 300, 400] and sees its own ASN
(300) in the path. BGP loop prevention requires a router to reject any advertisement whose
AS-PATH contains the receiving AS's own ASN. AS 300 therefore drops this advertisement
entirely and does not propagate it further. This prevents a routing loop from forming.

**3c.**

Split horizon is a local rule applied to one link: "do not advertise a route back to the neighbor
you learned it from." This prevents two-node loops (A -> B -> A) but fails when a loop spans
three or more nodes, because no single node can detect the circular dependency from its local
view alone.

BGP's AS-PATH carries the complete sequence of every AS the advertisement has traversed.
Any router anywhere along the path can detect a loop by checking whether its own ASN already
appears in the list — a O(path length) check with no false negatives. A loop of any length,
spanning any number of ASes, is caught the moment the advertisement reaches a node whose
ASN is already in the path. This works because the full history is carried in the advertisement
itself, rather than relying on any single node's local knowledge.

---

## Problem 4 — The Valley-Free Rule

### Part (a)

Consider the following AS topology with customer-provider relationships marked:

```
         AS 1 (Tier-1)  ---peer---  AS 2 (Tier-1)
           /     \                      /     \
    (provider)  (provider)       (provider)  (provider)
        /             \              /              \
    AS 3 (ISP)     AS 4 (ISP)   AS 5 (ISP)     AS 6 (ISP)
        |               |
    (provider)      (provider)
        |               |
    AS 7 (stub)     AS 8 (stub)
```

For each of the following traffic flows, state whether it is valley-free or not. Explain.

1. AS 7 -> AS 3 -> AS 1 -> AS 4 -> AS 8
2. AS 7 -> AS 3 -> AS 4 -> AS 8
3. AS 3 -> AS 1 -> AS 2 -> AS 5

**Part (b)**

Explain why a well-configured ISP (AS 3 in the topology above) would refuse to carry traffic
from one of its providers (AS 1) to another provider (AS 4), even if AS 3 has a direct link to
both.

---

**Solutions**

**4a.**

1. Valley-free. Traffic goes up from AS 7 (customer) to AS 3 (provider), up again to AS 1
   (Tier-1), then down to AS 4 (provider-to-customer), then down to AS 8 (customer). The
   path goes up then down, never reversing direction. This follows the valley-free rule.

2. Not valley-free. AS 7 goes up to AS 3, then the path would go laterally to AS 4 without
   going through a shared Tier-1. AS 3 and AS 4 are peers or unrelated — AS 3 would be
   providing free transit between a customer (AS 7) and another provider or peer (AS 4),
   which violates the business model. A proper path would go up to AS 1 first.

3. Valley-free. Traffic goes up from AS 3 to AS 1 (Tier-1), then across a peering link to AS 2
   (another Tier-1), then down to AS 5 (customer of AS 2). Up, peer, down — this is the
   canonical valley-free pattern.

**4b.**

The valley-free rule reflects business economics. AS 3 pays AS 1 for upstream transit. If AS 3
were to carry traffic from AS 1 to AS 4 (another provider), AS 3 would be providing free transit
between two networks it pays for, getting nothing in return. This is economically irrational.
ISPs configure BGP policy specifically to prevent this: routes learned from a provider are not
re-advertised to other providers or peers. Only customer routes are advertised upward, because
those are the routes the provider is being paid to carry.

---

## Problem 5 — BGP Incidents

### Part (a)

In 2008, Pakistan Telecom accidentally advertised the prefix 208.65.153.0/24 to its upstream
provider, which propagated globally. YouTube's actual prefix was 208.65.152.0/22. Using your
knowledge of longest-prefix match, explain precisely why this caused YouTube traffic to be
redirected to Pakistan Telecom rather than to YouTube, even though YouTube's correct route
was still in the global routing table.

### Part (b)

In 2021, Facebook withdrew all of its BGP routes due to a configuration error. This caused
not just facebook.com to go down, but made the outage very difficult to fix remotely. Explain
why withdrawing BGP routes would also take down Facebook's DNS, and why that made the
outage self-reinforcing.

### Part (c)

Both incidents involved BGP operating correctly — the protocol did exactly what it was
designed to do. What property of BGP makes it vulnerable to both of these failure modes,
and what mechanisms exist to mitigate them?

---

**Solutions**

**5a.**

BGP routers use longest-prefix match when selecting a route for a given destination address.
Pakistan Telecom advertised 208.65.153.0/24, which is a /24 — a longer (more specific) prefix
than YouTube's legitimate /22. A /24 covers 256 addresses; a /22 covers 1024. Any address
in the 208.65.153.x range matches both the /22 and the /24 simultaneously, and longest-prefix
match always selects the more specific entry.

Routers around the world that received both advertisements therefore preferred the /24 from
Pakistan Telecom for any destination in 208.65.153.x. Traffic for those addresses was
forwarded toward Pakistan rather than YouTube, even though YouTube's /22 was still
reachable and correctly advertised. The more-specific hijack only needs to cover a subset of
the victim's prefix to cause significant disruption.

**5b.**

Facebook's DNS servers (the servers that resolve facebook.com, instagram.com, etc. to IP
addresses) were hosted inside Facebook's own network and reachable via Facebook's BGP
prefixes. When those BGP routes were withdrawn, the DNS servers became unreachable from
the rest of the internet — not because the servers were down, but because the internet had no
routing path to them.

This made the outage self-reinforcing: engineers trying to diagnose the problem remotely
could not reach Facebook's internal infrastructure because the infrastructure itself was
unreachable. Even end users who had Facebook's IP addresses cached could not connect
because the routes to those IPs were gone. The usual remediation path (log in remotely,
push a config fix) was blocked, and Facebook engineers reportedly had to physically access
data centers to restore the configuration.

**5c.**

BGP is trust-based. By default, a BGP router accepts route advertisements from its neighbors
and propagates them without verifying that the advertising AS actually has the right to announce
those prefixes. There is no built-in authentication of prefix ownership. Pakistan Telecom's
routers were not compromised — they simply announced a prefix, and every other router on the
internet believed them. Facebook's configuration change was legitimate from BGP's perspective
— it was a valid withdrawal.

The main mitigation for prefix hijacking is RPKI (Resource Public Key Infrastructure), which
allows organizations to cryptographically sign their prefix ownership so that other networks
can verify that an advertisement is legitimate before accepting it. BGPsec extends this to sign
the entire AS-PATH. Adoption of both is ongoing but incomplete, which is why prefix hijacking
incidents still occur.

---

## Problem 6 — Intra-AS vs. Inter-AS Routing

A packet originates at a host in AS 100 and is destined for a host in AS 400. The path
traverses AS 100 -> AS 200 -> AS 300 -> AS 400.

### Part (a)

At each AS boundary crossing, which protocol is responsible for determining which AS to
enter next? Once inside an AS, which protocol determines which router within that AS to
forward the packet to?

### Part (b)

Within AS 200, there are multiple routers. The packet enters AS 200 at router R1 and must
exit to AS 300 via router R4. Explain how OSPF and BGP work together inside AS 200 to get
the packet from R1 to R4. What is this pattern called?

### Part (c)

OSPF convergence after a link failure takes seconds. BGP convergence can take minutes. Why
the difference? What is the consequence of this gap for a packet in transit during a failure?

---

**Solutions**

**6a.**

At each AS boundary, BGP is responsible. BGP advertisements have told AS 200's border
routers which prefixes are reachable via AS 300 (and the AS-PATH to get there), so when a
packet arrives destined for something in AS 400, the border router uses BGP-learned routes
to decide to forward toward AS 300.

Once inside an AS, OSPF (or whatever intra-AS IGP the network runs) determines the path.
OSPF has computed shortest paths to every router within the AS, so each router knows which
internal link to use to reach the AS exit point toward the next hop.

**6b.**

This pattern is called hot-potato routing (or more generally, the interaction between iBGP and
OSPF). BGP runs between border routers across AS boundaries (eBGP) but also between
routers within the same AS (iBGP) so that all routers learn about external prefixes. OSPF
provides the internal path costs. When R1 needs to reach R4 (the exit point toward AS 300),
it uses OSPF-computed shortest paths within AS 200 to forward the packet hop by hop through
the internal topology until it reaches R4. BGP told R1 that R4 is the right exit point; OSPF
tells R1 how to get to R4 internally.

**6c.**

OSPF is a link-state protocol: a failure triggers an immediate LSA flood, every router reruns
Dijkstra, and new forwarding tables are installed within seconds. The convergence time is
bounded by the diameter of the network and the speed of flooding.

BGP is path-vector and operates between administrative domains over TCP sessions. After a
failure, BGP must withdraw routes, wait for TCP retransmission timers, and allow the
withdrawal to propagate through multiple ASes before alternative paths are selected and
re-advertised. BGP also has deliberate hold timers to prevent route flapping. The combination
means convergence takes minutes.

A packet in transit during a BGP failure may be dropped, looped, or delivered via a suboptimal
path until convergence completes. The gap matters most for applications that cannot tolerate
even brief interruptions — a TCP connection may time out and reset during a BGP reconvergence
event even if an alternative path exists, because that path is not yet installed in forwarding tables.

---

*EC 441 – Boston University, Spring 2026*