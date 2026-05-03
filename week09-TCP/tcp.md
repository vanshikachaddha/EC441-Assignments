# Problem Set: TCP — Sequencing, Connection Management, Congestion and Flow Control

**Course:** EC 441 – Introduction to Computer Networking, Spring 2026
**Topic:** TCP (Lectures 19 & 20)
**Type:** Problem

---

## Problem 1 — TCP Header and Byte-Stream Model

### Part (a)

For each TCP header field, state its size and purpose:

1. Sequence Number
2. Acknowledgment Number
3. Window Size
4. Data Offset (header length)
5. RST flag

### Part (b)

A TCP connection has ISN = 1000 on the client side. The client sends three segments:
- Segment 1: 500 bytes
- Segment 2: 300 bytes
- Segment 3: 700 bytes

What is the sequence number of each segment? What ACK number does the server send after
receiving each one in order?

### Part (c)

TCP sequence numbers count bytes, not segments. Why does this matter for the ACK
interpretation? What does an ACK of 2500 mean in plain English?

### Part (d)

The TCP header has no explicit payload length field. How does the receiver determine how
many bytes of data are in a given segment?

---

**Solutions**

**1a.**

1. Sequence Number (32 bits) — the byte offset of the first byte of data in this segment within
   the sender's byte stream. For SYN segments, it carries the Initial Sequence Number and
   logically consumes one sequence number even though no data is sent.

2. Acknowledgment Number (32 bits) — the sequence number of the next byte the sender of
   this segment expects to receive. An ACK of N means "I have received all bytes up through
   N-1; send me byte N next." This field is only valid when the ACK flag is set.

3. Window Size (16 bits) — the receive window (rwnd): the number of bytes the sender of
   this segment is willing to accept beyond its last acknowledged byte. This is the flow control
   mechanism. Without the Window Scale option, the maximum is 65,535 bytes.

4. Data Offset (4 bits) — the length of the TCP header in 32-bit words, telling the receiver
   where the header ends and the data begins. Minimum value is 5 (20 bytes, no options);
   maximum is 15 (60 bytes). Analogous to IHL in the IPv4 header.

5. RST flag (1 bit) — reset. Immediately aborts the connection. Sent when a segment arrives
   for a port with no listening socket, or when one side wants to terminate abnormally without
   the graceful FIN/ACK exchange. On receiving RST, the other side discards all state for that
   connection immediately.

**1b.**

ISN = 1000, so the first data byte is byte 1001 (the SYN itself consumed sequence number 1000).

Segment 1: sequence number = 1001, carries bytes 1001-1500
Server ACK after segment 1: ack = 1501

Segment 2: sequence number = 1501, carries bytes 1501-1800
Server ACK after segment 2: ack = 1801

Segment 3: sequence number = 1801, carries bytes 1801-2500
Server ACK after segment 3: ack = 2501

**1c.**

Because sequence numbers count bytes, an ACK unambiguously identifies a position in the
byte stream regardless of how the data was split across segments. It does not matter whether
the sender sent the first 1,500 bytes as one segment or three — an ACK of 2500 means exactly
the same thing in both cases: all bytes up through byte 2499 have been received, and the
receiver is waiting for byte 2500 next. This is what makes TCP a byte stream rather than a
message protocol — the segmentation is invisible to the application.

**1d.**

The receiver computes the payload length from fields already present in the IP and TCP headers:
payload length = IP Total Length - IP Header Length - TCP Data Offset.
The IP Total Length gives the total size of the IP datagram; subtracting the two header lengths
leaves exactly the number of data bytes in the TCP segment. No separate payload length field
in the TCP header is needed.

---

## Problem 2 — Three-Way Handshake and Connection Teardown

### Part (a)

Trace a complete TCP three-way handshake between a client (port 54321) and a server (port
80). For each message, state: flags set, sequence number, acknowledgment number, and which
side sends it.

### Part (b)

Why must the handshake be three messages rather than two? What specific failure case does
the third message (the ACK) prevent?

### Part (c)

Why are Initial Sequence Numbers randomized rather than starting at 0? Give two distinct
reasons.

### Part (d)

Trace a graceful TCP connection teardown initiated by the client. Explain what TIME_WAIT
is, how long it lasts, and give both reasons why it exists.

### Part (e)

A client sends a SYN to a server port that has no listening socket. What does the server send
back, and what flag does it use?

---

**Solutions**

**2a.**

Step 1 — SYN (client -> server):
- Flags: SYN
- Seq = x (client's ISN, say x = 5000)
- Ack = 0 (ACK flag not set, field ignored)

Step 2 — SYN-ACK (server -> client):
- Flags: SYN, ACK
- Seq = y (server's ISN, say y = 9000)
- Ack = x + 1 = 5001 (acknowledges the client's SYN)

Step 3 — ACK (client -> server):
- Flags: ACK
- Seq = x + 1 = 5001
- Ack = y + 1 = 9001 (acknowledges the server's SYN)

Both sides are now ESTABLISHED. Data can flow in either direction starting with this segment.

**2b.**

Two messages (SYN + SYN-ACK) only guarantee that the server knows the client's ISN and
that the client knows the server's ISN. But the client has no way to confirm the server received
the third step, and more importantly, the server has no confirmation that its SYN-ACK arrived.

Without the third ACK: if the server's SYN-ACK is lost, the client never received it, so the
client does not know the server's ISN and cannot participate in the connection. The server,
however, has allocated resources and is waiting in SYN_RECEIVED. The third ACK serves as
confirmation to the server that the client received the SYN-ACK and that both ISNs are known
to both sides. Without it, the server would never know whether the client is ready.

**2c.**

1. Stale segment rejection. A TCP connection is identified by its 5-tuple. If a new connection
   reuses the same 5-tuple as a recently closed one and starts at ISN = 0, a segment that was
   delayed in the network from the old connection could have a sequence number that falls
   within the new connection's window and be accepted as valid data. A pseudo-random ISN
   makes this collision astronomically unlikely because the old segment's sequence number
   will almost certainly be outside the new connection's window.

2. Security. Predictable ISNs (such as always starting at 0 or using a simple counter) enabled
   TCP sequence-number prediction attacks. An off-path attacker who could guess the current
   ISN could inject forged segments into an established connection without being on the network
   path. Randomization defeats this class of attack because the attacker cannot guess the ISN.

**2d.**

Teardown sequence (client initiates):

1. Client sends FIN (seq = u). Client enters FIN_WAIT_1.
2. Server sends ACK (ack = u+1). Client enters FIN_WAIT_2. Server enters CLOSE_WAIT.
   The server may still send data at this point — the client-to-server direction is closed but
   not server-to-client.
3. Server finishes sending, sends FIN (seq = v). Server enters LAST_ACK.
4. Client sends ACK (ack = v+1). Client enters TIME_WAIT. Server receives ACK and enters
   CLOSED.

TIME_WAIT: the client waits for 2 x MSL (Maximum Segment Lifetime) before fully closing,
typically 2 to 4 minutes. Two reasons:

1. The final ACK may be lost. If the server does not receive the client's final ACK, the server
   retransmits its FIN. The client must still be around to re-send the ACK. TIME_WAIT keeps
   the connection state alive for one more FIN retransmission window.

2. Prevent 5-tuple reuse confusion. Any stale segments from the old connection still floating
   in the network will expire (TTL reaches 0) within one MSL. Waiting 2 x MSL guarantees
   those segments are gone before the 5-tuple can be reused by a new connection.

**2e.**

The server sends a RST segment back to the client. The RST flag signals that no socket is
listening on that port and the connection attempt should be immediately abandoned. The client
receives the RST and tears down its half-open connection state. This is different from a
firewall silently dropping the SYN — with RST, the client knows immediately that the port is
closed rather than waiting for a timeout.

---

## Problem 3 — Flow Control

### Part (a)

Explain the difference between flow control and congestion control. What does each protect,
and who sets each limit?

### Part (b)

A sender has the following state:
- Last byte sent: 4200
- Last byte ACKed: 3000
- Receiver's advertised window (rwnd): 2000 bytes

How many bytes can the sender still transmit right now? Show your reasoning.

### Part (c)

The receiver's application stops reading from the socket buffer. The buffer fills up completely.
What value does the receiver advertise in the Window field of its next ACK, and what does
the sender do in response? What mechanism prevents the connection from being stuck in
this state forever?

### Part (d)

A connection runs over a 1 Gb/s link with an RTT of 100 ms. What is the bandwidth-delay
product? Why does the standard 16-bit Window Size field create a bottleneck on this path,
and what TCP option fixes it?

---

**Solutions**

**3a.**

Flow control prevents the sender from overrunning the receiver's buffer. The receiver sets this
limit by advertising rwnd in every ACK — it tells the sender exactly how many bytes of free
buffer space are available. Flow control is end-to-end and receiver-driven.

Congestion control prevents the sender from overrunning the network. The sender infers
network capacity from packet loss signals and manages its own cwnd accordingly. No router
ever explicitly says "I am congested" — the sender detects congestion indirectly. Congestion
control is sender-driven and implicit.

The effective sending window is min(cwnd, rwnd), meaning both constraints apply
simultaneously. Flow control is about protecting the receiver; congestion control is about
protecting the shared network.

**3b.**

The sender's current outstanding (unacknowledged) data: 4200 - 3000 = 1200 bytes in flight.

The receiver has advertised a window of 2000 bytes beyond the last ACKed byte (3000), so
the sender is allowed to have bytes 3000 through 5000 in flight.

Bytes remaining to send: 2000 - 1200 = 800 bytes.

The sender can transmit 800 more bytes (up to sequence number 5000) before it must stop
and wait for more ACKs.

**3c.**

When the receive buffer is completely full, the receiver advertises rwnd = 0. The sender must
stop transmitting entirely — it is not allowed to send any data when rwnd = 0.

The connection would be stuck permanently if left alone: the receiver has no data to ACK
(since the sender stopped), so it sends no ACKs, so the sender never learns that buffer space
has freed up. TCP prevents this with the persist timer. When rwnd = 0, the sender
periodically sends a 1-byte probe segment (a window probe). This forces the receiver to send
an ACK with its current window advertisement. When the application eventually reads from
the buffer, the receiver will advertise a non-zero rwnd in response to the probe, and the sender
can resume transmission.

**3d.**

Bandwidth-delay product = 1 Gb/s x 0.1 s = 100 Mb = 12.5 MB.

To keep the pipe full, the sender needs 12.5 MB of data in flight at all times. The 16-bit
Window Size field can only advertise up to 65,535 bytes (~64 KB), which is roughly 200 times
smaller than the bandwidth-delay product. This caps throughput at about 64 KB / 0.1 s =
5.2 Mb/s on a 1 Gb/s link — a 200x reduction from what the link can support.

The Window Scale option (negotiated during the handshake) fixes this by multiplying the
advertised window by 2^n for a negotiated shift count n between 0 and 14. At n = 8 the
effective maximum is 65,535 x 256 = 16.7 MB, enough to cover this path. At n = 14 the
maximum is approximately 1 GB. Window Scale must be negotiated at the SYN/SYN-ACK
exchange — it cannot be added to an existing connection.

---

## Problem 4 — Retransmission and Fast Retransmit

### Part (a)

What is the retransmission timeout (RTO) and how is it computed from RTT samples? Write
out the Jacobson EWMA equations and explain what SRTT and RTTVAR represent.

### Part (b)

Why can't TCP simply use the most recent RTT sample as the timeout value?

### Part (c)

Five segments are sent. Segment 3 is lost. Segments 4 and 5 arrive at the receiver. Trace
what the receiver sends as ACKs after each of segments 4 and 5 arrive. What does the sender
do after receiving the third duplicate ACK?

### Part (d)

What is the difference between fast retransmit triggered by three duplicate ACKs, and
retransmission triggered by RTO expiry? Why does TCP treat these two loss signals
differently?

---

**Solutions**

**4a.**

TCP maintains two state variables updated with each new RTT sample R:

SRTT (smoothed RTT):
SRTT = (1 - alpha) * SRTT + alpha * R, where alpha = 1/8

RTTVAR (RTT variance):
RTTVAR = (1 - beta) * RTTVAR + beta * |SRTT - R|, where beta = 1/4

RTO is then set as:
RTO = SRTT + 4 * RTTVAR

SRTT is a low-pass filtered estimate of the mean RTT — it smooths out short-term fluctuations.
RTTVAR tracks how much the RTT varies around that mean. The factor of 4 in the RTO
formula ensures the timeout is set above nearly all observed RTT samples even when variance
is high, preventing spurious retransmissions under normal jitter.

**4b.**

RTT varies over time due to queueing, routing changes, and load. A single sample may be
an outlier — much higher or lower than typical. Using the raw sample directly would cause the
RTO to oscillate wildly: a brief spike in RTT would set an unnecessarily large timeout
(wasting time waiting), while a brief dip would set a timeout so small that normal-latency
packets trigger spurious retransmissions, injecting extra traffic into an already congested
network. The EWMA smooths out these fluctuations, and the variance term adds a safety
margin proportional to how unpredictable the RTT has been recently.

**4c.**

After segment 4 arrives (segment 3 is missing):
- Receiver sends duplicate ACK 3 (ack = seq of byte after segment 2's last byte, indicating
  it still wants segment 3). This is dup ACK 1.

After segment 5 arrives (segment 3 still missing):
- Receiver sends duplicate ACK 3 again. This is dup ACK 2.

If one more out-of-order segment arrived it would trigger dup ACK 3. Assuming the sender
also sent segment 6 (which arrives), that triggers dup ACK 3.

After three duplicate ACKs: the sender immediately retransmits segment 3 without waiting
for the RTO to expire. This is fast retransmit. The sender also sets ssthresh = cwnd / 2 and
cwnd = ssthresh and enters fast recovery.

**4d.**

Three duplicate ACKs signal a mild, localized loss: segments after the gap are still reaching
the receiver (the dup ACKs themselves prove this), so the network is not fully broken — just
one segment was dropped. TCP responds proportionally: halve cwnd, retransmit the missing
segment, and continue from there without resetting to slow start. The network is still capable
of delivering data, so there is no reason to be as conservative as if nothing were getting through.

RTO expiry is a much more severe signal. If the timeout fires, it means no ACK was received
for a prolonged period — the segment may be lost, or the network may be badly congested or
partially failed, or the path may have changed. TCP responds harshly: cwnd drops to 1 MSS
and slow start restarts from scratch. This is a full backoff because the network's current state
is unknown and the sender must re-probe carefully.

The asymmetry makes sense: three dup ACKs give the sender positive evidence that the
network is still partially functional; RTO expiry gives the sender no evidence at all.

---

## Problem 5 — Congestion Control: Slow Start and AIMD

### Part (a)

Starting from cwnd = 1 MSS and ssthresh = 16 MSS, trace the value of cwnd round by round
until cwnd reaches 20 MSS. Label each round as slow start (SS) or congestion avoidance (CA).

### Part (b)

Using the trace from Part (a), at round where cwnd = 20 MSS, three duplicate ACKs are
received. What are the new values of cwnd and ssthresh? What phase does TCP enter?

### Part (c)

Now, two rounds after the event in Part (b), a timeout fires. What are the new values of cwnd
and ssthresh? What phase does TCP enter?

### Part (d)

In steady-state congestion avoidance, TCP Reno's throughput can be approximated as:

    throughput ≈ (1.22 * MSS) / (RTT * sqrt(p))

where p is the packet loss rate. A connection has RTT = 50 ms, MSS = 1460 bytes, and
loss rate p = 0.001. Estimate the throughput. What loss rate would be needed to sustain
1 Gb/s on this path?

---

**Solutions**

**5a.**

ssthresh = 16, so slow start runs while cwnd < 16, congestion avoidance once cwnd >= 16.

| Round | Phase | cwnd (MSS) |
|-------|-------|------------|
| 0     | SS    | 1          |
| 1     | SS    | 2          |
| 2     | SS    | 4          |
| 3     | SS    | 8          |
| 4     | SS    | 16         |
| 5     | CA    | 17         |
| 6     | CA    | 18         |
| 7     | CA    | 19         |
| 8     | CA    | 20         |

Slow start doubles cwnd each round (1 -> 2 -> 4 -> 8 -> 16). At round 4, cwnd hits ssthresh
= 16 and switches to congestion avoidance, which adds 1 MSS per RTT linearly.

**5b.**

Three duplicate ACKs at cwnd = 20 MSS:

ssthresh = cwnd / 2 = 20 / 2 = 10 MSS
cwnd = ssthresh = 10 MSS

TCP enters fast recovery (not slow start). The missing segment is retransmitted immediately.
cwnd stays at 10 and congestion avoidance resumes from there — the sawtooth cuts in half
and continues climbing linearly.

**5c.**

Two rounds after fast recovery, cwnd has grown from 10 to 12 MSS (CA adds 1 per round).
A timeout fires at cwnd = 12 MSS:

ssthresh = cwnd / 2 = 12 / 2 = 6 MSS
cwnd = 1 MSS

TCP enters slow start. cwnd restarts exponential growth from 1, and will switch to congestion
avoidance again when it reaches ssthresh = 6.

This is the key difference from the dup-ACK event in Part (b): timeout drops cwnd all the
way to 1 and restarts slow start, whereas dup-ACKs only halve cwnd and skip slow start
entirely.

**5d.**

throughput = (1.22 * 1460) / (0.050 * sqrt(0.001))
           = 1781.2 / (0.050 * 0.03162)
           = 1781.2 / 0.001581
           ≈ 1,126,600 bytes/s
           ≈ 9.0 Mb/s

To sustain 1 Gb/s = 125,000,000 bytes/s:

125,000,000 = (1.22 * 1460) / (0.050 * sqrt(p))
sqrt(p) = 1781.2 / (125,000,000 * 0.050)
sqrt(p) = 1781.2 / 6,250,000
sqrt(p) ≈ 0.000000285
p ≈ (0.000000285)^2 ≈ 8.1 x 10^-14

This is essentially zero loss — TCP Reno cannot sustain 1 Gb/s on a 50 ms RTT path unless
the loss rate is extraordinarily small. This is the high-BDP problem: TCP Reno's AIMD
recovery is too slow relative to how long it takes to climb back up to full window size, so even
rare losses cause large throughput penalties.

---

## Problem 6 — Flow Control vs. Congestion Control Together

A TCP sender has the following state at a given moment:

- cwnd = 8 MSS
- ssthresh = 12 MSS
- rwnd advertised by receiver = 5 MSS
- Last byte ACKed = 10000
- Last byte sent = 13000
- MSS = 1000 bytes

### Part (a)

What is the effective window? How many bytes can the sender transmit right now?

### Part (b)

The receiver's buffer frees up and it advertises rwnd = 12 MSS. Does the sender immediately
use the full 12 MSS? Explain using cwnd.

### Part (c)

The sender then receives three duplicate ACKs. Update cwnd and ssthresh. With rwnd still
at 12 MSS, what is the new effective window?

### Part (d)

Explain in one paragraph why TCP needs both flow control and congestion control. Could you
get rid of one of them?

---

**Solutions**

**6a.**

Effective window = min(cwnd, rwnd) = min(8 MSS, 5 MSS) = 5 MSS = 5000 bytes.

Bytes currently in flight: last byte sent - last byte ACKed = 13000 - 10000 = 3000 bytes =
3 MSS.

Bytes the sender can still send: effective window - bytes in flight = 5000 - 3000 = 2000 bytes
= 2 MSS.

**6b.**

No. Even though rwnd = 12 MSS, cwnd = 8 MSS. The effective window is min(8, 12) = 8 MSS.
The sender is limited by its congestion window, not the receiver's buffer. cwnd represents the
sender's estimate of how much data the network can currently absorb — regardless of how much
buffer the receiver has, sending beyond cwnd risks overwhelming the network. The sender can
only transmit up to 8 MSS worth of unacknowledged data total.

**6c.**

Three duplicate ACKs trigger fast retransmit and fast recovery:
ssthresh = cwnd / 2 = 8 / 2 = 4 MSS
cwnd = ssthresh = 4 MSS

New effective window = min(cwnd, rwnd) = min(4 MSS, 12 MSS) = 4 MSS.

Even though rwnd is now generous, the congestion event cut cwnd in half, and that is the
binding constraint. The sender must rebuild cwnd slowly through congestion avoidance before
it can take advantage of the receiver's larger buffer.

**6d.**

Flow control and congestion control protect different things and cannot substitute for each
other. Flow control protects the receiver: without it, a fast sender could fill the receiver's
socket buffer, causing the kernel to drop arriving segments and forcing retransmissions. The
receiver explicitly tells the sender its limit via rwnd, so the sender knows exactly how much
buffer is available. Congestion control protects the shared network: without it, every TCP
sender would transmit as fast as the receiver allows, router queues would fill and drop packets,
and all flows would suffer. No single receiver can signal the network's overall congestion state
— the sender must infer it from loss. You cannot eliminate either: removing flow control
overruns receivers; removing congestion control causes network collapse.

---

*EC 441 – Boston University, Spring 2026*
