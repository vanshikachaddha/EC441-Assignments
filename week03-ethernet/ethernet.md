# Problem: Ethernet Addressing, Switching, and ARP

## Problem

Consider a local area network (LAN) with the following devices:

| Device | IP Address     | MAC Address          |
|--------|----------------|----------------------|
| Host A | 192.168.1.10   | AA:AA:AA:AA:AA:AA    |
| Host B | 192.168.1.20   | BB:BB:BB:BB:BB:BB    |
| Host C | 192.168.1.30   | CC:CC:CC:CC:CC:CC    |

All hosts are connected to a **learning Ethernet switch**.

Initially, the switch’s **MAC address table is empty**.

---

## Questions

1. Host A wants to send a packet to Host B. Describe the full process, including ARP.
2. What Ethernet frame is sent during ARP request and reply?
3. How does the switch update its MAC table during this process?
4. After the first transmission, what entries exist in the switch table?
5. If Host C now sends a frame to Host A, how does the switch handle it?

---

## Solution

### 1. Host A sends data to Host B

#### Step 1: Check ARP cache
Host A checks its ARP table for IP `192.168.1.20`.  
No entry exists → must use ARP.

#### Step 2: ARP Request (Broadcast)
Host A sends an ARP request:

- Destination MAC: FF:FF:FF:FF:FF:FF (broadcast)
- Source MAC: AA:AA:AA:AA:AA:AA
- Payload: "Who has 192.168.1.20?"

#### Step 3: Switch behavior
- Switch receives frame on Port A
- Learns:

AA:AA:AA:AA:AA:AA → Port A

- Forwards frame to all other ports (flooding)

#### Step 4: ARP Reply
Host B responds:

- Destination MAC: AA:AA:AA:AA:AA:AA
- Source MAC: BB:BB:BB:BB:BB:BB
- Payload: "192.168.1.20 is BB:BB:BB:BB:BB:BB"

#### Step 5: Switch updates
- Learns:

BB:BB:BB:BB:BB:BB → Port B

- Sends reply only to Host A

#### Step 6: Data transmission
Host A now sends actual data using:
- Destination MAC: BB:BB:BB:BB:BB:BB  
Switch forwards directly to Host B.

---

### 2. Ethernet Frames

#### ARP Request Frame
| Field            | Value                     |
|------------------|---------------------------|
| Destination MAC  | FF:FF:FF:FF:FF:FF         |
| Source MAC       | AA:AA:AA:AA:AA:AA         |
| Type             | ARP                       |
| Payload          | Who has 192.168.1.20?     |

#### ARP Reply Frame
| Field            | Value                          |
|------------------|--------------------------------|
| Destination MAC  | AA:AA:AA:AA:AA:AA              |
| Source MAC       | BB:BB:BB:BB:BB:BB              |
| Type             | ARP                            |
| Payload          | 192.168.1.20 is at BB:...      |

---

### 3. Switch Learning Process

Switches are self-learning:
- Learn source MAC → incoming port
- Flood unknown destinations
- Forward directly when destination is known

---

### 4. Final MAC Table

| MAC Address          | Port   |
|----------------------|--------|
| AA:AA:AA:AA:AA:AA    | Port A |
| BB:BB:BB:BB:BB:BB    | Port B |

---

### 5. Host C sends to Host A

#### Step 1: Learn source
Switch learns:

CC:CC:CC:CC:CC:CC → Port C


#### Step 2: Lookup destination
Destination MAC = AA:AA:AA:AA:AA:AA → already known

#### Step 3: Forward
Switch sends frame only to Port A (no flooding)

---

## Key Takeaways

- ARP maps IP addresses to MAC addresses  
- ARP requests are broadcast; replies are unicast  
- Switches learn MAC addresses dynamically  
- Unknown destination → flooding  
- Known destination → direct forwarding  

---

## Reflection

This problem shows how Ethernet switching and ARP work together:
- ARP resolves addressing at the network layer  
- Switches handle efficient delivery at the link layer  

Together, they enable seamless communication within a LAN.