# Problem: Reliable Data Transfer – Stop-and-Wait, Go-Back-N, and Selective Repeat

## Problem

A sender transmits packets over a reliable data transfer protocol to a receiver. The network has the following properties:

- One-way propagation delay = 50 ms  
- Transmission time per packet = 10 ms  
- Round-trip time (RTT) = 2 × 50 ms = 100 ms  
- Ignore ACK transmission time  

Answer the following:

1. For **Stop-and-Wait**, what is the utilization of the sender?
2. For **Go-Back-N (GBN)** with window size \( N = 5 \), what is the utilization?
3. For **Selective Repeat (SR)** with window size \( N = 5 \), what is the utilization?
4. If packet 3 is lost in GBN, describe what happens.
5. If packet 3 is lost in SR, describe what happens.

---

## Solution

---

## 1. Stop-and-Wait Utilization

In Stop-and-Wait:
- Sender transmits 1 packet, then waits for ACK

Total cycle time:

\[
\text{Cycle time} = \text{Transmission time} + \text{RTT}
\]

\[
= 10 + 100 = 110 \text{ ms}
\]

Utilization:

\[
U = \frac{\text{Transmission time}}{\text{Cycle time}} = \frac{10}{110} \approx 0.091
\]

\[
\boxed{U \approx 9.1\%}
\]

---

## 2. Go-Back-N Utilization (N = 5)

GBN allows sending up to 5 packets before waiting.

Total useful transmission time:

\[
5 \times 10 = 50 \text{ ms}
\]

Cycle time:

\[
50 + 100 = 150 \text{ ms}
\]

Utilization:

\[
U = \frac{50}{150} = \frac{1}{3} \approx 0.333
\]

\[
\boxed{U \approx 33.3\%}
\]

---

## 3. Selective Repeat Utilization (N = 5)

Selective Repeat behaves similarly to GBN in ideal (no loss) conditions:

\[
U = \frac{N \cdot T_{tx}}{RTT + T_{tx}}
\]

\[
U = \frac{5 \cdot 10}{100 + 10} = \frac{50}{110} \approx 0.455
\]

\[
\boxed{U \approx 45.5\%}
\]

---

## 4. Packet Loss in Go-Back-N

If packet 3 is lost:

- Sender transmits packets 1, 2, 3, 4, 5
- Receiver:
  - Accepts packets 1 and 2
  - Discards packets 4 and 5 (out of order)
- Receiver sends ACK for last in-order packet (ACK 2)

Sender:
- Times out for packet 3
- Retransmits: