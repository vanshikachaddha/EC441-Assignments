# Power, Bandwidth, and Reliability Tradeoffs in an AWGN Channel

## EC 441 – Physical Layer Analysis (Lecture 3)

---

# 1. Overview

Lecture 3 introduced the physical layer constraints that ultimately limit networking performance. While higher layers manipulate packets and protocols, all communication reduces to electromagnetic energy propagating through a noisy channel.

This artifact explores three central concepts from the lecture:

- Shannon capacity (rate limit)
- Energy per bit and BER (reliability)
- Multi-level signaling tradeoffs

The goal is to connect these ideas mathematically and conceptually.

---

# 2. Shannon Capacity Revisited

The Shannon–Hartley theorem states:

$$
C = B \log_2(1 + SNR)
$$

where:
- $B$ is bandwidth
- $SNR$ is linear

## Key Observations

1. Capacity increases linearly with bandwidth.
2. Capacity increases logarithmically with power.
3. High SNR yields diminishing returns.

This explains why wireless systems aggressively reuse spectrum and adapt modulation rather than simply increasing transmit power.

Importantly, Shannon capacity is not a practical operating point — it is a theoretical upper bound assuming optimal coding and infinite block lengths.

---

# 3. From SNR to $E_b/N_0$

SNR compares total signal power to total noise power:

$$
SNR = \frac{P}{N}
$$

However, reliability depends on energy per bit:

$$
E_b = \frac{P}{R_b}
$$

Thus:

$$
\frac{E_b}{N_0} = \frac{S}{N} \cdot \frac{B}{R_b}
$$

Increasing bit rate at fixed power reduces $E_b$ and therefore increases BER.

This highlights the tension between rate and reliability.

---

# 4. BER in AWGN

For bipolar signaling:

$$
P_b = Q\left(\sqrt{\frac{2E_b}{N_0}}\right)
$$

As $E_b/N_0$ increases, BER drops exponentially.

This produces the “waterfall effect,” where a few dB improvement drastically lowers error probability.

For example:

- $E_b/N_0 \approx 11$ dB → $BER \approx 10^{-6}$

This value is frequently used as a design threshold in digital systems.

---

# 5. Multi-Level Signaling

Increasing modulation order $M$:

- Improves spectral efficiency
- Decreases symbol spacing (for fixed power)
- Increases sensitivity to noise

Approximate symbol error probability:

$$
P_s \approx 2\left(1 - \frac{1}{M}\right)
Q\left(\frac{d_{min}}{2\sigma}\right)
$$

As $M$ increases, $d_{min}$ decreases unless transmit power increases.

Thus higher-order modulation requires higher SNR to maintain BER.

This is why WiFi dynamically adjusts modulation based on channel quality.

---

# 6. Extensive Worked Problem

## Given

- $B = 10$ MHz  
- $P_s = -80$ dBm  
- $N_0 = -174$ dBm/Hz  

---

## (a) Noise Power

$$
N = -174 + 10\log_{10}(10^7)
$$

$$
= -174 + 70
$$

$$
= -104 \text{ dBm}
$$

---

## (b) SNR

$$
SNR(dB) = -80 - (-104) = 24 \text{ dB}
$$

Linear:

$$
SNR = 10^{24/10} \approx 251
$$

---

## (c) Shannon Capacity

$$
C = 10^7 \log_2(252)
$$

$$
\log_2(252) \approx 7.98
$$

$$
C \approx 79.8 \text{ Mbps}
$$

---

## (d) Operation at 50 Mbps

Using:

$$
\frac{E_b}{N_0} = \frac{S}{N} \cdot \frac{B}{R_b}
$$

$$
= 251 \cdot \frac{10^7}{5 \times 10^7}
$$

$$
= 50.2
$$

Convert to dB:

$$
10\log_{10}(50.2) \approx 17 \text{ dB}
$$

---

## (e) BER Check

$$
P_b = Q(\sqrt{2 \times 50.2})
$$

$$
\sqrt{100.4} \approx 10
$$

$$
Q(10) \approx 10^{-23}
$$

Thus BER is far below $10^{-6}$.

The system operates comfortably below capacity with strong reliability margin.

---

## (f) What If We Switch to 16-PAM?

At fixed transmit power:

- Symbol spacing shrinks.
- Noise more easily causes symbol confusion.
- Required SNR must increase substantially.

Approximately +6 dB per doubling of modulation order is needed to maintain BER.

This demonstrates the bandwidth–power tradeoff.

---

# 7. Practical Networking Implications

Physical layer constraints directly influence networking behavior:

- Ethernet limits length due to attenuation and timing.
- WiFi adapts modulation based on SNR.
- Higher throughput modes require stronger signal quality.
- Error correction coding is essential near capacity.

Understanding Shannon and BER clarifies why throughput drops rapidly when signal strength decreases.

---

# 8. Reflection

This exercise illustrates the difference between theoretical limits and practical performance. Shannon capacity defines what is mathematically possible, but BER analysis determines what is operationally reliable. Increasing rate without increasing power reduces energy per bit, increasing errors. Similarly, increasing modulation order improves spectral efficiency but demands higher SNR.

Ultimately, physical-layer design is a balance between power, bandwidth, and reliability.
