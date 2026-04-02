# Report: Link-State and Distance-Vector Routing

**Course:** EC 441 – Introduction to Computer Networking, Spring 2026
**Topic:** Routing — Link State (Dijkstra), Distance Vector (Bellman-Ford)
**Type:** Report

---

## Overview

Routing is the control-plane problem of computing what forwarding tables across a network
should contain so that packets reach their destinations efficiently. Every router in the internet
needs to know, for any destination, which outgoing link to use. The question is how routers
collectively figure that out without a central authority telling them.

There are two fundamental families of algorithms for solving this problem: link-state routing
and distance-vector routing. They approach the same goal from opposite directions, and
understanding both — and why each one exists — is the foundation for understanding how
the real internet works.

---

## The Network as a Graph

Before either algorithm can run, you need a model of the network. The standard abstraction
is a weighted graph G = (V, E), where V is the set of routers and E is the set of links between
them. Each edge has a cost, and the goal of routing is to find the minimum-cost path between
any pair of nodes.

The choice of edge cost determines what "shortest" means. RIP uses hop count, treating every
link as cost 1. OSPF uses inverse bandwidth by default (cost = 10^8 / link bandwidth in bps),
so a 100 Mb/s link costs 1 and a 1 Mb/s link costs 100. You can also assign administrative
costs manually to push traffic toward or away from certain links. The algorithm itself does not
care — it just minimizes total cost, whatever cost represents.

---

## Link-State Routing

### How It Works

In link-state routing, every router in the network learns the complete topology, then
independently computes shortest paths to all other routers. There are two phases:

**Phase 1 — Flooding.** Each router constructs a Link-State Advertisement (LSA) describing
its own links and their costs. It floods this LSA to every router in the network, not just its
neighbors. After flooding completes, every router has an identical copy of the full topology —
a link-state database. This is the "local information distributed globally" half of the picture.

**Phase 2 — Dijkstra.** With the full graph available locally, each router independently runs
Dijkstra's shortest-path algorithm to compute a shortest-path tree rooted at itself. The result
is a forwarding table: for each destination, the table says which outgoing link to use.

### Dijkstra's Algorithm

Dijkstra's algorithm finds the shortest path from a source node to all other nodes in a graph
with non-negative edge weights. The core idea is to maintain a set of visited nodes whose
shortest distances are finalized, and a set of unvisited nodes with current distance estimates.
At each step, the unvisited node with the smallest current estimate is moved to the visited set,
and its neighbors' estimates are updated if a shorter path is found through it.

The update step is called relaxation:

```
relax(u, v):
    if d[v] > d[u] + w(u, v):
        d[v] = d[u] + w(u, v)
        predecessor[v] = u
```

A simple Python implementation:

```python
def dijkstra(graph, source):
    dist = {v: float('inf') for v in graph}
    dist[source] = 0
    prev = {v: None for v in graph}
    unvisited = set(graph.keys())

    while unvisited:
        u = min(unvisited, key=lambda v: dist[v])
        unvisited.remove(u)
        for v, weight in graph[u].items():
            if v in unvisited and dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                prev[v] = u

    return dist, prev
```

**Complexity:** O(V^2) with a naive min-search, O((V + E) log V) with a priority queue.

One important constraint: Dijkstra requires all edge weights to be non-negative. A negative
weight would allow a visited node's distance to be improved after it has been finalized, which
breaks the algorithm's correctness guarantee.

### Worked Example

Consider a 4-node network:

```
    A ---2--- B
    |         |
    4         1
    |         |
    C ---3--- D
```

Running Dijkstra from A:

| Step | Visited | d[A] | d[B] | d[C] | d[D] |
|------|---------|------|------|------|------|
| Init | {}      | 0    | inf  | inf  | inf  |
| 1    | {A}     | 0    | 2    | 4    | inf  |
| 2    | {A,B}   | 0    | 2    | 4    | 3    |
| 3    | {A,B,D} | 0    | 2    | 4    | 3    |
| 4    | {A,B,C,D}| 0   | 2    | 4    | 3    |

Shortest paths from A: to B = 2 (direct), to D = 3 (A->B->D), to C = 4 (direct, since
A->B->D->C would cost 6).

### OSPF: Link-State in Practice

OSPF (Open Shortest Path First) is the deployed link-state protocol for intra-AS routing.
Each router floods LSAs when a link state changes. Routers use hello packets to detect
neighbor failures. When a failure is detected, an LSA is immediately flooded and every router
reruns Dijkstra. Because every router has the full topology, there are no routing loops after
convergence — a router that knows the complete graph cannot make a forwarding decision that
contradicts another router's correct view.

---

## Distance-Vector Routing

### How It Works

Distance-vector routing takes the opposite approach. No router ever learns the full topology.
Instead, each router maintains a distance vector — a table of its current best estimate of the
cost to reach every destination, plus which neighbor to route through. Routers exchange these
vectors only with direct neighbors, and iteratively improve their estimates using the
Bellman-Ford equation.

This is the "global information distributed locally" half of the picture: each router shares
what it knows about the entire network, but only with its immediate neighbors.

### The Bellman-Ford Equation

The cost of the best path from node x to destination y is:

```
D_x(y) = min over all neighbors v of { c(x,v) + D_v(y) }
```

Where c(x,v) is the cost of the direct link from x to neighbor v, and D_v(y) is v's current
reported cost to y. The min is taken over all neighbors — you try every neighbor as a possible
next hop and keep the cheapest option.

### The Algorithm

Each node initializes its distance vector with direct link costs (infinity for non-neighbors,
0 for itself). It sends this vector to all neighbors. When a node receives an updated vector
from a neighbor, it reruns the Bellman-Ford equation for each destination. If any estimate
improves, it sends its updated vector to its own neighbors. This continues until no node has
anything new to report — convergence.

Key properties: the algorithm is distributed (no node needs global information), asynchronous
(nodes do not update in lockstep), and self-terminating (stops when nothing changes).

### Worked Example: Convergence Trace

5-node network with the following direct link costs:

```
A-B: 1, A-C: 4, B-C: 2, B-D: 3, C-E: 1, D-E: 5
```

Round 0 — initialization (each node knows only direct neighbors):

| Node | d(A) | d(B) | d(C) | d(D) | d(E) |
|------|------|------|------|------|------|
| A    | 0    | 1    | 4    | inf  | inf  |
| B    | 1    | 0    | 2    | 3    | inf  |
| C    | 4    | 2    | 0    | inf  | 1    |
| D    | inf  | 3    | inf  | 0    | 5    |
| E    | inf  | inf  | 1    | 5    | 0    |

Round 1 — after first exchange. Example update at A: A receives B's vector [1,0,2,3,inf].
A updates d_A(C) = min(4, 1+2) = 3 via B. A updates d_A(D) = min(inf, 1+3) = 4 via B.

| Node | d(A) | d(B) | d(C) | d(D) | d(E) |
|------|------|------|------|------|------|
| A    | 0    | 1    | 3    | 4    | 5    |
| B    | 1    | 0    | 2    | 3    | 3    |
| C    | 3    | 2    | 0    | 5    | 1    |
| D    | 4    | 3    | 5    | 0    | 5    |
| E    | 5    | 3    | 1    | 5    | 0    |

Round 2 — only two entries change: d_A(E) improves from 5 to 4 (via B, since B now reports
d_B(E) = 3), and d_E(A) improves from 5 to 4 (via C). After round 3, nothing changes.
Converged.

---

## The Count-to-Infinity Problem

Distance-vector routing has a fundamental weakness when links fail. Consider a simple chain:

```
A ---1--- B ---1--- C
```

Initial state: d_A(C) = 2 via B, d_B(C) = 1 direct.

The B-C link fails. Here is what happens:

1. B detects the failure and checks its neighbors. A reports d_A(C) = 2. B computes:
   d_B(C) = c(B,A) + d_A(C) = 1 + 2 = 3 via A. This is wrong — A's route to C goes
   through B, but B does not know that.
2. B tells A: d_B(C) = 3. A updates: d_A(C) = 1 + 3 = 4.
3. B updates: d_B(C) = 1 + 4 = 5. And so on.

The costs count up to infinity one step at a time, with packets bouncing in a loop between A
and B. The root cause is circular dependency: B is using A's distance estimate, which itself
depends on B.

### Fixes

**Split horizon:** if A routes to C via B, A does not advertise its route to C back to B. This
breaks the feedback loop in the two-node case. When B-C fails, B never hears A claim a
working path to C, so B immediately sets d_B(C) = infinity.

**Poisoned reverse:** a stronger version. If A routes to C via B, A actively tells B that
d_A(C) = infinity (instead of simply not mentioning it). This explicitly prevents B from ever
considering A as a path to C.

**Limitations:** both fixes only work for two-node loops. In a topology with three or more
nodes forming a cycle, circular dependencies can still form. Larger loops require hold-down
timers or are simply accepted as a convergence delay. This asymmetry — fast convergence for
good news (cost decreases), slow convergence for bad news (failures) — is the defining weakness
of distance-vector routing.

---

## Side-by-Side Comparison

| Property              | Link-State (OSPF)              | Distance-Vector (RIP)         |
|-----------------------|--------------------------------|-------------------------------|
| Information needed    | Full topology at every router  | Only neighbor distance vectors|
| Algorithm             | Dijkstra                       | Bellman-Ford                  |
| Communication         | Flood LSAs to all routers      | Exchange DVs with neighbors only |
| Convergence on failure| Fast (immediate flood + recompute) | Slow (count-to-infinity)  |
| Loop-free?            | Yes, after convergence         | Not guaranteed during convergence |
| Memory/CPU            | Higher (stores full graph)     | Lower (stores only DV table)  |
| Metric flexibility    | Rich (bandwidth, delay, admin) | Typically hop count only      |
| Max network size      | Scales to large networks       | RIP limited to 15 hops        |
| Deployed as           | OSPF, IS-IS                    | RIP                           |

---

## Why Both Exist

Link-state is strictly better in almost every technical dimension: faster convergence, no
count-to-infinity, richer metrics, larger scale. OSPF has largely replaced RIP in real networks.
So why does distance-vector still matter?

First, distance-vector is simpler to implement and requires less memory. In very small
networks (home routers, small offices) this used to be a real advantage, and RIP still appears
in some embedded devices.

Second, and more importantly, the distance-vector idea scales up. BGP — the protocol that
routes traffic between autonomous systems across the entire internet — is a path-vector
protocol, which is a generalization of distance-vector routing. BGP exchanges not just costs
but full AS paths, which allows loop detection and policy enforcement. The internet cannot
use link-state globally: flooding topology information across 100,000+ autonomous systems is
impossible, and organizations have no obligation to share their internal topology with anyone.
BGP's distance-vector heritage is exactly what makes it feasible at internet scale.

Understanding both algorithms — and their tradeoffs — is what makes the full routing picture
legible: OSPF (link-state) inside each autonomous system, BGP (path-vector, descended from
distance-vector) between them.

---

