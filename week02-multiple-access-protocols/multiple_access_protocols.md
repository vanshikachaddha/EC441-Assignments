# Multiple Access Protocols: ALOHA and CSMA

## Overview
In networking, **multiple access protocols** determine how multiple devices share a common communication medium (e.g., a wireless channel or Ethernet cable). The goal is to coordinate transmissions to minimize collisions and maximize efficiency.

This artifact explores two foundational protocols:
- **ALOHA (Pure and Slotted)**
- **CSMA (Carrier Sense Multiple Access)**

---

## The Core Problem
When multiple devices transmit over the same channel:
- Collisions can occur
- Data gets corrupted
- Retransmissions reduce efficiency

Protocols are needed to:
- Decide *when* a node can transmit
- Handle collisions when they happen

---

## ALOHA

### Pure ALOHA
**Idea:**  
Nodes transmit whenever they have data.

**How it works:**
1. Send immediately
2. If collision occurs → wait random time → retransmit

**Key Concept: Vulnerability Period**
- A frame is vulnerable to collisions for **2 × frame time**

**Efficiency:**
- Maximum throughput ≈ **18%**

**Why so low?**
- No coordination → high collision probability

---

### Slotted ALOHA
**Improvement over Pure ALOHA**

**Idea:**  
Time is divided into discrete slots. Nodes can only transmit at the start of a slot.

**Benefits:**
- Reduces collision window
- Synchronization improves efficiency

**Efficiency:**
- Maximum throughput ≈ **37%**

**Tradeoff:**
- Requires time synchronization

---

## CSMA (Carrier Sense Multiple Access)

### Core Idea
Before transmitting, a node **listens to the channel**.

> “Don’t talk if someone else is talking.”

---

### Basic CSMA Algorithm
1. Sense the channel
2. If idle → transmit
3. If busy → wait and retry

---

### Variants of CSMA

#### 1. 1-Persistent CSMA
- If idle → transmit immediately
- If busy → keep listening, transmit as soon as free

**Pros:**
- Low delay

**Cons:**
- High collision probability (many nodes jump in at once)

---

#### 2. Non-Persistent CSMA
- If busy → wait a random time before retrying

**Pros:**
- Fewer collisions

**Cons:**
- Higher delay

---

#### 3. p-Persistent CSMA (Slotted)
- If idle → transmit with probability **p**
- Otherwise → wait for next slot

**Pros:**
- Balances delay and collision probability

---

## CSMA/CD (Collision Detection)
Used in **Ethernet (wired networks)**

**Key idea:**
- Nodes detect collisions *while transmitting*

**Steps:**
1. Transmit
2. If collision detected → stop immediately
3. Send jam signal
4. Wait random backoff time (exponential backoff)
5. Retry

---

## CSMA/CA (Collision Avoidance)
Used in **Wi-Fi (802.11)**

**Why not detect collisions?**
- Hard to detect collisions in wireless

**Instead:**
- Try to avoid them

**Mechanisms:**
- Random backoff timers
- RTS/CTS (Request to Send / Clear to Send)

---

## Comparison Summary

| Protocol        | Coordination | Efficiency | Collision Handling        |
|----------------|-------------|------------|----------------------------|
| Pure ALOHA     | None        | ~18%       | Retransmit after collision |
| Slotted ALOHA  | Time slots  | ~37%       | Reduced collision window   |
| CSMA           | Listen first| Higher     | Avoid collisions           |
| CSMA/CD        | Listen + detect | Higher | Detect + stop + retry      |
| CSMA/CA        | Avoid       | Higher     | Prevent collisions         |

---

## Key Takeaways
- ALOHA is simple but inefficient
- Slotted ALOHA improves performance via synchronization
- CSMA significantly reduces collisions by sensing the channel
- Real-world systems (Ethernet, Wi-Fi) build on CSMA with detection or avoidance

---

## Reflection
ALOHA demonstrates the cost of no coordination, while CSMA shows how even a simple improvement (listening before sending) dramatically increases efficiency. These protocols form the foundation for modern network communication systems.

---

## Possible Extensions
- Simulate ALOHA vs CSMA throughput in Python
- Analyze Wireshark traces for CSMA/CD behavior
- Implement exponential backoff logic

# Problem: Throughput of Pure ALOHA and Slotted ALOHA

## Problem
A shared channel uses ALOHA for medium access. Let the total offered load be \( G \), where \( G \) represents the average number of transmission attempts per frame time.

1. Derive the throughput formula for **Pure ALOHA**.
2. Find the value of \( G \) that maximizes throughput.
3. Compute the maximum throughput.
4. Repeat the same for **Slotted ALOHA**.
5. Compare the two results.

---

## Solution

## 1. Pure ALOHA Throughput

In **Pure ALOHA**, a node can begin transmitting at any time.

A transmission succeeds only if no other node transmits during the frame's **vulnerable period**.

### Vulnerable period
For Pure ALOHA, the vulnerable period is:

\[
2T
\]

where \( T \) is one frame time.

If the average number of transmission attempts per frame time is \( G \), then over a vulnerable period of length \( 2T \), the average number of attempts is:

\[
2G
\]

Using the Poisson model, the probability that **zero** other transmissions occur during this vulnerable period is:

\[
P(\text{success}) = e^{-2G}
\]

So throughput is:

\[
S = G \cdot e^{-2G}
\]

---

## 2. Maximize Pure ALOHA Throughput

We differentiate:

\[
S(G) = G e^{-2G}
\]

Using the product rule:

\[
\frac{dS}{dG} = e^{-2G} + G(-2e^{-2G})
\]

\[
\frac{dS}{dG} = e^{-2G}(1 - 2G)
\]

Set derivative equal to 0:

\[
e^{-2G}(1 - 2G) = 0
\]

Since \( e^{-2G} \neq 0 \), we get:

\[
1 - 2G = 0
\]

\[
G = \frac{1}{2}
\]

---

## 3. Maximum Pure ALOHA Throughput

Substitute \( G = \frac{1}{2} \) into the throughput formula:

\[
S_{\max} = \frac{1}{2} e^{-1}
\]

\[
S_{\max} \approx \frac{1}{2}(0.3679)
\]

\[
S_{\max} \approx 0.184
\]

So the maximum throughput of Pure ALOHA is:

\[
\boxed{18.4\%}
\]

---

## 4. Slotted ALOHA Throughput

In **Slotted ALOHA**, transmissions can only begin at the start of a time slot.

This reduces the vulnerable period to just **1 frame time**.

So the probability that no other node transmits in the same slot is:

\[
P(\text{success}) = e^{-G}
\]

Thus throughput is:

\[
S = G e^{-G}
\]

### Maximize throughput
Differentiate:

\[
S(G) = G e^{-G}
\]

\[
\frac{dS}{dG} = e^{-G} + G(-e^{-G})
\]

\[
\frac{dS}{dG} = e^{-G}(1 - G)
\]

Set equal to 0:

\[
e^{-G}(1 - G) = 0
\]

So:

\[
G = 1
\]

Now substitute back:

\[
S_{\max} = 1 \cdot e^{-1} = e^{-1}
\]

\[
S_{\max} \approx 0.3679
\]

So the maximum throughput of Slotted ALOHA is:

\[
\boxed{36.8\%}
\]

---

## 5. Comparison

| Protocol        | Throughput Formula | Optimal \(G\) | Maximum Throughput |
|----------------|--------------------|---------------|--------------------|
| Pure ALOHA     | \(S = Ge^{-2G}\)   | \(1/2\)       | \(1/(2e) \approx 0.184\) |
| Slotted ALOHA  | \(S = Ge^{-G}\)    | \(1\)         | \(1/e \approx 0.368\) |

Slotted ALOHA achieves about **double** the maximum throughput of Pure ALOHA because dividing time into slots cuts the vulnerable period in half.

---

## Final Answer
- **Pure ALOHA:**  
  \[
  S = Ge^{-2G}, \quad S_{\max} = \frac{1}{2e} \approx 0.184
  \]

- **Slotted ALOHA:**  
  \[
  S = Ge^{-G}, \quad S_{\max} = \frac{1}{e} \approx 0.368
  \]

- **Conclusion:** Slotted ALOHA is significantly more efficient than Pure ALOHA.

---

