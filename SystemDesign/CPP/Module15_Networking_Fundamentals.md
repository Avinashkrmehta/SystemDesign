# Module 15: Networking Fundamentals

> Networking is the backbone of every distributed system. Before designing any high-level architecture, you must understand how data travels across networks — from the physical cables to the application protocols that power the web. This module covers the OSI and TCP/IP models, IP addressing, DNS, HTTP/HTTPS, WebSockets, and other essential protocols. These concepts are foundational for every HLD interview and real-world system design.

---

## 15.1 OSI Model (7 Layers)

> The **OSI (Open Systems Interconnection) Model** is a conceptual framework that standardizes how different networking protocols and technologies interact. It divides network communication into **7 layers**, each with a specific responsibility. While real-world networks use the TCP/IP model, the OSI model is invaluable for understanding, troubleshooting, and discussing networking concepts.

---

### The 7 Layers — Overview

```
+----------------------------+  Layer 7
|     Application Layer      |  (HTTP, FTP, SMTP, DNS, SSH)
+----------------------------+  Layer 6
|     Presentation Layer     |  (SSL/TLS, Encryption, Compression, JPEG, ASCII)
+----------------------------+  Layer 5
|       Session Layer        |  (Session management, NetBIOS, RPC)
+----------------------------+  Layer 4
|      Transport Layer       |  (TCP, UDP — port numbers, reliability)
+----------------------------+  Layer 3
|       Network Layer        |  (IP, ICMP, Routing — IP addresses)
+----------------------------+  Layer 2
|      Data Link Layer       |  (Ethernet, Wi-Fi, MAC addresses, Switches)
+----------------------------+  Layer 1
|      Physical Layer        |  (Cables, Signals, Hubs, Fiber optics)
+----------------------------+
```

**Mnemonic (top to bottom):** **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing
**Mnemonic (bottom to top):** **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

---

### Layer 1: Physical Layer

The **Physical Layer** deals with the raw transmission of bits (0s and 1s) over a physical medium.

**Responsibilities:**
- Transmitting raw bitstreams over physical media
- Defining electrical signals, voltages, pin layouts, cable specifications
- Bit synchronization (clock recovery)
- Data rate (bandwidth) and transmission mode (simplex, half-duplex, full-duplex)

**Physical Media:**

| Medium | Type | Speed | Distance | Use Case |
|--------|------|-------|----------|----------|
| Twisted Pair (Cat5e/Cat6) | Copper | 1-10 Gbps | ~100m | LAN, Ethernet |
| Coaxial Cable | Copper | Up to 10 Gbps | ~500m | Cable TV, older networks |
| Fiber Optic | Glass/Plastic | 10-100+ Gbps | Up to 100+ km | Long-distance, data centers |
| Wireless (Wi-Fi) | Radio waves | Up to 9.6 Gbps (Wi-Fi 6) | ~100m indoor | WLAN |

**Devices at this layer:** Hubs, Repeaters, Cables, Network Interface Cards (NICs)

**Key Point:** The Physical Layer has no concept of addresses, packets, or protocols — it only knows about bits and signals.

---

### Layer 2: Data Link Layer

The **Data Link Layer** provides node-to-node data transfer between two directly connected devices. It handles framing, physical addressing (MAC), error detection, and flow control on a local network segment.

**Responsibilities:**
- **Framing:** Encapsulates network layer packets into frames (adds header and trailer)
- **Physical Addressing (MAC):** Uses MAC (Media Access Control) addresses — 48-bit hardware addresses (e.g., `AA:BB:CC:DD:EE:FF`)
- **Error Detection:** Uses CRC (Cyclic Redundancy Check) to detect corrupted frames
- **Flow Control:** Prevents a fast sender from overwhelming a slow receiver
- **Media Access Control:** Determines which device can transmit on a shared medium (CSMA/CD for Ethernet, CSMA/CA for Wi-Fi)

**Sub-layers:**
1. **LLC (Logical Link Control):** Flow control, error detection, multiplexing
2. **MAC (Media Access Control):** Physical addressing, media access

**Frame Structure (Ethernet):**

```
+----------+----------+------+------+---------+-----+
| Preamble | Dest MAC | Src  | Type | Payload | FCS |
| (8 bytes)| (6 bytes)| MAC  |(2 B) |(46-1500)|(4 B)|
|          |          |(6 B) |      | bytes   |     |
+----------+----------+------+------+---------+-----+
```

**Devices at this layer:** Switches, Bridges

**Key Point:** Switches operate at Layer 2 — they use MAC address tables to forward frames only to the correct port, unlike hubs which broadcast to all ports.

---

### Layer 3: Network Layer

The **Network Layer** handles logical addressing (IP addresses) and routing — determining the best path for data to travel from source to destination across multiple networks.

**Responsibilities:**
- **Logical Addressing:** Assigns IP addresses to identify devices across networks
- **Routing:** Determines the optimal path for packets across interconnected networks
- **Packet Forwarding:** Moves packets from one router to the next toward the destination
- **Fragmentation and Reassembly:** Splits large packets to fit the MTU (Maximum Transmission Unit) of the link

**Key Protocols:**

| Protocol | Purpose |
|----------|---------|
| IP (IPv4/IPv6) | Logical addressing and routing |
| ICMP | Error reporting, diagnostics (ping, traceroute) |
| ARP | Maps IP addresses to MAC addresses |
| OSPF, BGP, RIP | Routing protocols |

**IP Packet Structure (simplified):**

```
+--------+--------+--------+--------+--------+---------+
| Version| Header | TTL    | Protocol| Source | Dest   |
| (4 bits)| Length |        |        | IP     | IP     |
+--------+--------+--------+--------+--------+---------+
|                     Payload (Data)                    |
+------------------------------------------------------+
```

**Devices at this layer:** Routers, Layer 3 Switches

**Key Point:** Routers operate at Layer 3 — they use routing tables and IP addresses to forward packets between different networks. Each hop decrements the TTL (Time To Live) to prevent infinite loops.

---

### Layer 4: Transport Layer

The **Transport Layer** provides end-to-end communication between processes on different hosts. It handles segmentation, flow control, error recovery, and multiplexing via port numbers.

**Responsibilities:**
- **Segmentation:** Breaks application data into segments (TCP) or datagrams (UDP)
- **Port Numbers:** Multiplexes multiple applications on the same host (e.g., port 80 for HTTP, port 443 for HTTPS)
- **Reliability (TCP):** Ensures data arrives correctly and in order
- **Flow Control:** Prevents sender from overwhelming receiver
- **Congestion Control:** Prevents sender from overwhelming the network

**Key Protocols:**

| Protocol | Connection | Reliable | Ordered | Use Case |
|----------|-----------|----------|---------|----------|
| TCP | Connection-oriented | Yes | Yes | HTTP, FTP, Email, SSH |
| UDP | Connectionless | No | No | DNS, Video streaming, Gaming, VoIP |

**Port Number Ranges:**

| Range | Name | Examples |
|-------|------|---------|
| 0-1023 | Well-known ports | HTTP (80), HTTPS (443), SSH (22), DNS (53), FTP (21) |
| 1024-49151 | Registered ports | MySQL (3306), PostgreSQL (5432), Redis (6379) |
| 49152-65535 | Dynamic/Ephemeral | Assigned by OS for client connections |

**Devices at this layer:** Firewalls (stateful), Load Balancers (Layer 4)

---

### Layer 5: Session Layer

The **Session Layer** manages sessions (connections) between applications. It establishes, maintains, and terminates communication sessions.

**Responsibilities:**
- **Session Establishment:** Setting up a communication channel
- **Session Maintenance:** Keeping the session alive, handling re-establishment after interruption
- **Session Termination:** Gracefully closing the session
- **Synchronization:** Checkpointing in long data transfers (so you can resume from the last checkpoint on failure)
- **Dialog Control:** Managing turn-taking in communication (half-duplex vs full-duplex)

**Examples:**
- NetBIOS (Network Basic Input/Output System)
- RPC (Remote Procedure Call)
- PPTP (Point-to-Point Tunneling Protocol)
- Session management in web applications (though often handled at the application layer in practice)

**Key Point:** In modern networking, the Session Layer is often merged with the Application Layer. TCP handles much of what the Session Layer describes.

---

### Layer 6: Presentation Layer

The **Presentation Layer** handles data translation, encryption, and compression — ensuring that data from the Application Layer of one system can be read by the Application Layer of another.

**Responsibilities:**
- **Data Translation:** Converting between different data formats (e.g., EBCDIC to ASCII, big-endian to little-endian)
- **Encryption/Decryption:** SSL/TLS encryption happens conceptually at this layer
- **Compression:** Reducing data size for efficient transmission (e.g., gzip, JPEG, MPEG)
- **Serialization:** Converting data structures into a transmittable format (e.g., JSON, XML, Protocol Buffers)

**Examples:**
- SSL/TLS (encryption)
- JPEG, GIF, PNG (image formats)
- MPEG, H.264 (video formats)
- ASCII, Unicode (character encoding)

**Key Point:** Like the Session Layer, the Presentation Layer is often absorbed into the Application Layer in practice. TLS, for example, sits between the Transport and Application layers in the TCP/IP model.

---

### Layer 7: Application Layer

The **Application Layer** is the closest layer to the end user. It provides network services directly to applications — this is where protocols like HTTP, FTP, SMTP, and DNS operate.

**Responsibilities:**
- Providing network services to user applications
- Defining protocols for specific application functions (web browsing, email, file transfer)
- User authentication and authorization at the protocol level

**Key Protocols:**

| Protocol | Port | Purpose |
|----------|------|---------|
| HTTP/HTTPS | 80/443 | Web browsing, REST APIs |
| FTP | 20/21 | File transfer |
| SMTP | 25/587 | Sending email |
| IMAP | 143/993 | Retrieving email |
| POP3 | 110/995 | Retrieving email |
| DNS | 53 | Domain name resolution |
| SSH | 22 | Secure remote access |
| DHCP | 67/68 | Dynamic IP assignment |
| SNMP | 161/162 | Network management |

**Devices at this layer:** Application-level firewalls (WAF), Load Balancers (Layer 7), Proxy Servers

---

### What Happens at Each Layer — Data Flow

When you send data, it travels **down** the OSI stack on the sender side and **up** the stack on the receiver side. Each layer adds its own header (encapsulation) on the way down and removes it (decapsulation) on the way up.

```
Sender                                              Receiver
+-------------+                                +-------------+
| Application | → Data                         | Application |
+-------------+                                +-------------+
| Presentation| → Encrypted/Formatted Data     | Presentation|
+-------------+                                +-------------+
| Session     | → Session-managed Data         | Session     |
+-------------+                                +-------------+
| Transport   | → Segment (TCP) / Datagram(UDP)| Transport   |
+-------------+                                +-------------+
| Network     | → Packet (+ IP header)         | Network     |
+-------------+                                +-------------+
| Data Link   | → Frame (+ MAC header + FCS)   | Data Link   |
+-------------+                                +-------------+
| Physical    | → Bits (electrical/optical)     | Physical    |
+-------------+                                +-------------+
         ↓                                           ↑
    [Physical Medium: cables, wireless, fiber]
```

**Data Unit Names at Each Layer:**

| Layer | Data Unit Name |
|-------|---------------|
| Application, Presentation, Session | Data / Message |
| Transport | Segment (TCP) / Datagram (UDP) |
| Network | Packet |
| Data Link | Frame |
| Physical | Bits |

---

### OSI Model — Summary Table

| Layer | Name | Function | Protocols | Devices | Data Unit |
|-------|------|----------|-----------|---------|-----------|
| 7 | Application | User-facing services | HTTP, FTP, DNS, SMTP | WAF, L7 LB | Data |
| 6 | Presentation | Encryption, compression, format | SSL/TLS, JPEG, ASCII | — | Data |
| 5 | Session | Session management | NetBIOS, RPC | — | Data |
| 4 | Transport | End-to-end delivery, ports | TCP, UDP | Firewall, L4 LB | Segment/Datagram |
| 3 | Network | Routing, logical addressing | IP, ICMP, OSPF, BGP | Router | Packet |
| 2 | Data Link | Framing, MAC addressing | Ethernet, Wi-Fi, ARP | Switch, Bridge | Frame |
| 1 | Physical | Bit transmission | Ethernet (physical), USB | Hub, Repeater, Cable | Bits |

---

### Interview Relevance

- **"At which layer does a load balancer operate?"** — Layer 4 (TCP/UDP) or Layer 7 (HTTP). Layer 4 LBs route based on IP/port; Layer 7 LBs can inspect HTTP headers, URLs, cookies.
- **"What's the difference between a switch and a router?"** — Switch operates at Layer 2 (MAC addresses, local network); Router operates at Layer 3 (IP addresses, between networks).
- **"Where does TLS/SSL fit?"** — Conceptually at Layer 6 (Presentation), but in practice it sits between Layer 4 (Transport) and Layer 7 (Application).

---


## 15.2 TCP/IP Model

> The **TCP/IP Model** (also called the Internet Protocol Suite) is the practical networking model that powers the internet. Unlike the theoretical 7-layer OSI model, TCP/IP has **4 layers** and is what real-world networking implementations follow. Understanding TCP vs UDP is critical for system design — the choice between them affects latency, reliability, and architecture.

---

### The 4 Layers

```
+----------------------------+
|     Application Layer      |  (HTTP, FTP, SMTP, DNS, SSH, DHCP)
|                            |  ← Combines OSI Layers 5, 6, 7
+----------------------------+
|      Transport Layer       |  (TCP, UDP)
|                            |  ← Same as OSI Layer 4
+----------------------------+
|      Internet Layer        |  (IP, ICMP, ARP)
|                            |  ← Same as OSI Layer 3
+----------------------------+
|   Network Access Layer     |  (Ethernet, Wi-Fi, PPP)
|   (Link Layer)             |  ← Combines OSI Layers 1, 2
+----------------------------+
```

---

### OSI vs TCP/IP Comparison

| Feature | OSI Model | TCP/IP Model |
|---------|-----------|-------------|
| Layers | 7 | 4 |
| Developed by | ISO (International Organization for Standardization) | DARPA / DoD |
| Nature | Theoretical / Reference model | Practical / Implementation model |
| Session & Presentation | Separate layers | Merged into Application layer |
| Approach | Protocol-independent | Protocol-dependent (built around TCP/IP) |
| Usage | Teaching, troubleshooting | Actual internet communication |

---

### TCP (Transmission Control Protocol)

TCP is a **connection-oriented**, **reliable** protocol that guarantees data delivery in the correct order. It is the workhorse of the internet — HTTP, HTTPS, FTP, SMTP, and SSH all run on top of TCP.

---

#### Three-Way Handshake (Connection Establishment)

Before any data is exchanged, TCP establishes a connection using a **three-way handshake**:

```
Client                          Server
  |                                |
  |  ---- SYN (seq=x) -------->   |   Step 1: Client sends SYN
  |                                |            (I want to connect, my sequence number is x)
  |  <--- SYN-ACK (seq=y, ack=x+1)|   Step 2: Server responds with SYN-ACK
  |                                |            (OK, my sequence number is y, I acknowledge x+1)
  |  ---- ACK (ack=y+1) ------->  |   Step 3: Client sends ACK
  |                                |            (I acknowledge y+1, connection established)
  |                                |
  |  ===== DATA TRANSFER =====    |   Connection is now established
  |                                |
```

**Why three steps?**
- **SYN:** Client initiates, proposes its initial sequence number
- **SYN-ACK:** Server acknowledges and proposes its own sequence number
- **ACK:** Client acknowledges the server's sequence number
- Both sides now have synchronized sequence numbers for reliable data transfer

---

#### Four-Way Handshake (Connection Termination)

TCP uses a **four-way handshake** to gracefully close a connection:

```
Client                          Server
  |                                |
  |  ---- FIN ----------------->   |   Step 1: Client says "I'm done sending"
  |                                |
  |  <--- ACK -----------------   |   Step 2: Server acknowledges
  |                                |            (Server may still send data)
  |                                |
  |  <--- FIN -----------------   |   Step 3: Server says "I'm done too"
  |                                |
  |  ---- ACK ----------------->   |   Step 4: Client acknowledges
  |                                |
  |  === CONNECTION CLOSED ===     |
```

**Why four steps instead of three?**
- The connection is **full-duplex** — each direction is closed independently
- After the client sends FIN, the server may still have data to send
- The server sends its own FIN only when it's done sending

---

#### Reliability Mechanisms

TCP guarantees reliable, ordered delivery through several mechanisms:

**1. Sequence Numbers and Acknowledgments:**
- Every byte of data is assigned a sequence number
- The receiver sends ACK with the next expected sequence number
- If the sender doesn't receive an ACK within a timeout, it retransmits

```
Sender                          Receiver
  |  ---- Seq=1, Data="AB" --->   |
  |  <--- ACK=3 ---------------   |   (I received up to byte 2, send byte 3 next)
  |  ---- Seq=3, Data="CD" --->   |
  |  <--- ACK=5 ---------------   |
  |  ---- Seq=5, Data="EF" --->   |   (This packet is LOST!)
  |         ... timeout ...        |
  |  ---- Seq=5, Data="EF" --->   |   (Retransmit!)
  |  <--- ACK=7 ---------------   |
```

**2. Flow Control (Sliding Window):**
- The receiver advertises a **window size** — how much data it can buffer
- The sender limits unacknowledged data to the window size
- Prevents a fast sender from overwhelming a slow receiver

```
Window Size = 4 packets

Sender can send packets 1, 2, 3, 4 without waiting for ACK
  |  ---- Pkt 1 --->  |
  |  ---- Pkt 2 --->  |
  |  ---- Pkt 3 --->  |
  |  ---- Pkt 4 --->  |   (Window full — must wait for ACK)
  |  <--- ACK 3 ----  |   (Receiver got 1 and 2)
  |  ---- Pkt 5 --->  |   (Window slides: can now send 5 and 6)
  |  ---- Pkt 6 --->  |
```

**3. Congestion Control:**
TCP adjusts its sending rate based on network conditions to avoid overwhelming the network. The sender maintains a **congestion window (cwnd)** that limits how much unacknowledged data can be in flight.

| Algorithm | Description |
|-----------|-------------|
| **Slow Start** | Start with a small congestion window, double it each RTT until threshold |
| **Congestion Avoidance** | After threshold, increase window linearly (additive increase) |
| **Fast Retransmit** | Retransmit after 3 duplicate ACKs (don't wait for timeout) |
| **Fast Recovery** | After fast retransmit, halve the window instead of resetting to 1 |

**Congestion Control — Step by Step:**

```
Phase 1: Slow Start (exponential growth)
  cwnd = 1 MSS (Maximum Segment Size, typically ~1460 bytes)
  RTT 1: cwnd = 1 → send 1 segment
  RTT 2: cwnd = 2 → send 2 segments
  RTT 3: cwnd = 4 → send 4 segments
  RTT 4: cwnd = 8 → send 8 segments
  ... doubles every RTT until ssthresh (slow start threshold) is reached

Phase 2: Congestion Avoidance (linear growth)
  Once cwnd >= ssthresh:
  RTT N:   cwnd = 16 → send 16 segments
  RTT N+1: cwnd = 17 → send 17 segments  (increase by 1 MSS per RTT)
  RTT N+2: cwnd = 18 → send 18 segments
  ... grows linearly until packet loss is detected

Phase 3: Packet Loss Detected
  Option A — Timeout (severe loss):
    ssthresh = cwnd / 2
    cwnd = 1 MSS
    Go back to Slow Start

  Option B — 3 Duplicate ACKs (mild loss, Fast Retransmit + Fast Recovery):
    ssthresh = cwnd / 2
    cwnd = ssthresh + 3 MSS
    Retransmit the lost segment
    Continue in Congestion Avoidance (skip Slow Start)
```

```
cwnd
  ^
  |          /\
  |         /  \        /\
  |        /    \      /  \       /
  |       /      \    /    \     /
  |      /        \  /      \   /
  |     /          \/        \ /
  |    /                      
  |   / ← Slow Start    ← Congestion Avoidance
  |  /
  | /
  +----------------------------------------→ Time
       ↑ loss          ↑ loss
```

**Modern Congestion Control Algorithms:**

| Algorithm | Description | Used By |
|-----------|-------------|---------|
| **TCP Reno** | Classic: slow start + congestion avoidance + fast retransmit/recovery | Legacy systems |
| **TCP Cubic** | Default in Linux; uses a cubic function for window growth; more aggressive recovery | Linux (default since 2.6.19) |
| **TCP BBR** | Google's algorithm; estimates bandwidth and RTT; doesn't rely on packet loss as signal | Google, YouTube |
| **TCP Vegas** | Proactive; detects congestion before packet loss by monitoring RTT changes | Research, some deployments |
| **QUIC (HTTP/3)** | Implements congestion control at application layer over UDP; per-stream control | Google, Cloudflare, Meta |

**BBR vs Cubic:**
- **Cubic** reacts to packet loss — it reduces the window when packets are dropped. This works poorly on high-bandwidth, high-latency links (long fat networks) where buffer bloat causes unnecessary loss.
- **BBR** (Bottleneck Bandwidth and Round-trip propagation time) proactively estimates the available bandwidth and optimal sending rate. It doesn't wait for packet loss. Google reported 2-25x throughput improvement on YouTube after deploying BBR.

**4. Checksum:**
- Every TCP segment includes a 16-bit checksum computed over the TCP header, payload, and a pseudo-header (source IP, dest IP, protocol, TCP length)
- If the checksum fails, the segment is silently discarded (sender will retransmit after timeout)
- The checksum catches most single-bit errors but is not cryptographically secure — TLS provides stronger integrity guarantees

**5. Ordering:**
- Sequence numbers ensure data is reassembled in the correct order
- Out-of-order segments are buffered and reordered before delivery to the application
- **Selective Acknowledgment (SACK):** An extension that lets the receiver tell the sender exactly which segments it has received, so the sender only retransmits the missing ones (instead of everything after the gap)

```
Without SACK:
  Sender sends: 1, 2, 3, 4, 5
  Receiver gets: 1, 2, _, 4, 5  (3 is lost)
  Receiver ACKs: 3 (I need segment 3)
  Sender retransmits: 3, 4, 5  (retransmits everything from 3 onward)

With SACK:
  Sender sends: 1, 2, 3, 4, 5
  Receiver gets: 1, 2, _, 4, 5  (3 is lost)
  Receiver ACKs: 3, SACK=4-5  (I need 3, but I already have 4 and 5)
  Sender retransmits: 3 only  (much more efficient)
```

**6. TCP Segment Structure (detailed):**

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |       |U|A|P|R|S|F|                                   |
| Offset| Rsrvd |R|C|S|S|Y|I|            Window Size            |
|       |       |G|K|H|T|N|N|                                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**TCP Flags:**

| Flag | Name | Purpose |
|------|------|---------|
| SYN | Synchronize | Initiate connection (three-way handshake) |
| ACK | Acknowledge | Acknowledge received data |
| FIN | Finish | Gracefully close connection |
| RST | Reset | Forcefully abort connection (error, rejected connection) |
| PSH | Push | Deliver data to application immediately (don't buffer) |
| URG | Urgent | Data contains urgent information (rarely used) |

**TCP States (Connection Lifecycle):**

```
Client States:                    Server States:
  CLOSED                            CLOSED
    ↓ (send SYN)                      ↓ (listen)
  SYN_SENT                          LISTEN
    ↓ (receive SYN-ACK, send ACK)     ↓ (receive SYN, send SYN-ACK)
  ESTABLISHED                       SYN_RECEIVED
    ↓ (send FIN)                      ↓ (receive ACK)
  FIN_WAIT_1                        ESTABLISHED
    ↓ (receive ACK)                   ↓ (receive FIN, send ACK)
  FIN_WAIT_2                        CLOSE_WAIT
    ↓ (receive FIN, send ACK)        ↓ (send FIN)
  TIME_WAIT                         LAST_ACK
    ↓ (wait 2*MSL)                    ↓ (receive ACK)
  CLOSED                            CLOSED
```

**TIME_WAIT State:**
- After closing, the client enters TIME_WAIT for 2×MSL (Maximum Segment Lifetime, typically 60 seconds)
- Purpose: ensure the final ACK reaches the server; prevent old duplicate segments from being confused with new connections
- **System design impact:** On high-traffic servers, thousands of connections in TIME_WAIT can exhaust ephemeral ports. Solutions: `SO_REUSEADDR`, `SO_REUSEPORT`, connection pooling, or tuning `tcp_tw_reuse`

---

### UDP (User Datagram Protocol)

UDP is a **connectionless**, **unreliable** protocol. It sends datagrams without establishing a connection, without guaranteeing delivery, and without ordering.

**UDP Datagram Structure:**

```
+------------------+------------------+
| Source Port (16) | Dest Port (16)   |
+------------------+------------------+
| Length (16)      | Checksum (16)    |
+------------------+------------------+
|              Payload               |
+------------------------------------+
```

**Characteristics:**
- **No handshake** — just send the datagram
- **No acknowledgment** — sender doesn't know if data arrived
- **No ordering** — datagrams may arrive out of order
- **No flow control** — sender can overwhelm the receiver
- **No congestion control** — sender can overwhelm the network
- **Low overhead** — only 8-byte header (vs 20+ bytes for TCP)
- **Fast** — no connection setup delay, no retransmission delay

---

### TCP vs UDP — Detailed Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery (ACK + retransmit) | Best-effort (no guarantee) |
| Ordering | In-order delivery | No ordering |
| Flow Control | Yes (sliding window) | No |
| Congestion Control | Yes (slow start, AIMD) | No |
| Header Size | 20-60 bytes | 8 bytes |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Latency | Higher (handshake + ACK) | Lower (no handshake) |
| Data Unit | Segment | Datagram |
| Broadcasting | No | Yes (multicast, broadcast) |
| Use Cases | HTTP, HTTPS, FTP, SSH, Email | DNS, Video streaming, Gaming, VoIP, IoT |

---

### When to Use TCP vs UDP

| Scenario | Choose | Why |
|----------|--------|-----|
| Web browsing (HTTP/HTTPS) | TCP | Need reliable, ordered delivery of web pages |
| File transfer (FTP, SCP) | TCP | Cannot afford lost or corrupted data |
| Email (SMTP, IMAP) | TCP | Messages must arrive complete and in order |
| Database connections | TCP | Queries and results must be reliable |
| DNS queries | UDP | Small, single request-response; speed matters; can retry at app level |
| Video streaming | UDP | Occasional lost frames are acceptable; low latency is critical |
| Online gaming | UDP | Low latency is critical; game state can tolerate some loss |
| VoIP (Voice over IP) | UDP | Real-time audio; retransmitting old audio is useless |
| IoT sensor data | UDP | Lightweight, frequent small messages; some loss is acceptable |
| Live broadcasting | UDP | Real-time delivery; buffering defeats the purpose |

**Key Insight for System Design:**
- Use **TCP** when correctness matters more than speed (APIs, databases, file transfer)
- Use **UDP** when speed matters more than correctness (real-time media, gaming, IoT)
- Many modern protocols build reliability on top of UDP when they need custom control (e.g., QUIC, which powers HTTP/3)

---

### TCP/IP in Practice — What Happens When You Visit a Website

```
1. Application Layer:  Browser creates HTTP GET request
2. Transport Layer:    TCP segments the request, adds port 443 (HTTPS)
3. Internet Layer:     IP adds source and destination IP addresses
4. Link Layer:         Ethernet adds MAC addresses, creates frame
5. Physical Layer:     Bits transmitted over the wire/wireless

   → Travels through switches (L2), routers (L3), firewalls (L4/L7)

6. At the server, the process reverses:
   Physical → Link → Internet → Transport → Application
   Server processes the HTTP request and sends back a response
```

---


## 15.3 IP Addressing & DNS

> Every device on the internet needs a unique address to send and receive data. **IP addresses** provide this logical addressing, while **DNS (Domain Name System)** translates human-readable domain names (like `google.com`) into IP addresses. Understanding IP addressing and DNS is essential for system design — DNS is involved in load balancing, failover, CDN routing, and service discovery.

---

### IPv4

**IPv4 (Internet Protocol version 4)** uses **32-bit addresses**, written as four octets in dotted-decimal notation.

**Format:** `A.B.C.D` where each octet is 0-255
**Example:** `192.168.1.100`
**Total addresses:** 2^32 = ~4.3 billion (not enough for all devices today)

**Address Classes (historical):**

| Class | Range | Default Subnet Mask | Networks | Hosts/Network | Use |
|-------|-------|---------------------|----------|---------------|-----|
| A | 1.0.0.0 – 126.255.255.255 | 255.0.0.0 (/8) | 126 | ~16.7 million | Large organizations |
| B | 128.0.0.0 – 191.255.255.255 | 255.255.0.0 (/16) | 16,384 | ~65,000 | Medium organizations |
| C | 192.0.0.0 – 223.255.255.255 | 255.255.255.0 (/24) | ~2 million | 254 | Small networks |
| D | 224.0.0.0 – 239.255.255.255 | — | — | — | Multicast |
| E | 240.0.0.0 – 255.255.255.255 | — | — | — | Reserved/Experimental |

**Private IP Ranges (RFC 1918) — not routable on the internet:**

| Range | CIDR | Class | Common Use |
|-------|------|-------|-----------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | A | Large corporate networks, cloud VPCs |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | B | Medium networks |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | C | Home networks, small offices |

**Special Addresses:**

| Address | Purpose |
|---------|---------|
| `127.0.0.1` | Loopback (localhost) — refers to the local machine |
| `0.0.0.0` | "Any" address — used by servers to listen on all interfaces |
| `255.255.255.255` | Broadcast — sends to all devices on the local network |
| `169.254.x.x` | Link-local — auto-assigned when DHCP fails |

---

### IPv6

**IPv6 (Internet Protocol version 6)** uses **128-bit addresses** to solve the IPv4 address exhaustion problem.

**Format:** Eight groups of four hexadecimal digits, separated by colons
**Example:** `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
**Shortened:** `2001:db8:85a3::8a2e:370:7334` (leading zeros and consecutive zero groups can be compressed)
**Total addresses:** 2^128 = ~3.4 × 10^38 (essentially unlimited)

**Key Differences from IPv4:**

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Address size | 32 bits | 128 bits |
| Notation | Dotted decimal (192.168.1.1) | Hexadecimal colon (2001:db8::1) |
| Total addresses | ~4.3 billion | ~3.4 × 10^38 |
| Header size | 20-60 bytes (variable) | 40 bytes (fixed) |
| NAT required | Yes (address shortage) | No (enough addresses) |
| IPSec | Optional | Built-in |
| Broadcast | Yes | No (uses multicast instead) |
| Auto-configuration | DHCP | SLAAC (Stateless Address Auto-Configuration) + DHCPv6 |

**Key Point for System Design:** While IPv6 adoption is growing, most system design discussions still focus on IPv4. However, understanding IPv6 is important for global-scale systems and cloud infrastructure.

---

### Subnetting Basics

**Subnetting** divides a large network into smaller, more manageable sub-networks (subnets). It improves security, reduces broadcast traffic, and enables efficient IP address allocation.

**Subnet Mask:** Determines which part of an IP address is the network portion and which is the host portion.

```
IP Address:    192.168.1.100
Subnet Mask:   255.255.255.0   (/24 in CIDR notation)

Binary:
IP:     11000000.10101000.00000001.01100100
Mask:   11111111.11111111.11111111.00000000
        |-------- Network --------|-Host--|

Network Address: 192.168.1.0    (all host bits = 0)
Broadcast:       192.168.1.255  (all host bits = 1)
Usable Hosts:    192.168.1.1 – 192.168.1.254  (254 hosts)
```

**CIDR (Classless Inter-Domain Routing) Notation:**

| CIDR | Subnet Mask | Usable Hosts | Example |
|------|-------------|-------------|---------|
| /8 | 255.0.0.0 | 16,777,214 | 10.0.0.0/8 |
| /16 | 255.255.0.0 | 65,534 | 172.16.0.0/16 |
| /24 | 255.255.255.0 | 254 | 192.168.1.0/24 |
| /25 | 255.255.255.128 | 126 | 192.168.1.0/25 |
| /26 | 255.255.255.192 | 62 | 192.168.1.0/26 |
| /28 | 255.255.255.240 | 14 | 192.168.1.0/28 |
| /30 | 255.255.255.252 | 2 | Point-to-point links |
| /32 | 255.255.255.255 | 1 | Single host route |

**Key Point for System Design:** In cloud environments (AWS VPC, GCP VPC), you define subnets using CIDR blocks. A `/16` VPC gives you 65,534 IPs; a `/24` subnet gives you 254 IPs. Plan your CIDR ranges carefully to avoid overlap and allow for growth.

---

### DNS (Domain Name System)

DNS is the **phonebook of the internet** — it translates human-readable domain names (e.g., `www.google.com`) into IP addresses (e.g., `142.250.80.46`).

---

#### DNS Resolution Process

There are two types of DNS queries:

**Recursive Query:** The DNS resolver does all the work on behalf of the client.
**Iterative Query:** The DNS resolver returns the best answer it has, and the client follows referrals.

**Full DNS Resolution Flow (Recursive):**

```
User types "www.example.com" in browser

1. Browser Cache
   → Check if the IP is cached in the browser
   → If found, use it. If not, continue.

2. OS Cache (Stub Resolver)
   → Check the OS DNS cache (/etc/hosts, systemd-resolved)
   → If found, use it. If not, continue.

3. Recursive Resolver (ISP or configured DNS like 8.8.8.8)
   → Check its own cache
   → If found, return it. If not, start resolving:

4. Root Name Server (.)
   → Resolver asks: "Where is .com?"
   → Root server responds: "Ask the .com TLD server at X.X.X.X"

5. TLD Name Server (.com)
   → Resolver asks: "Where is example.com?"
   → TLD server responds: "Ask the authoritative server at Y.Y.Y.Y"

6. Authoritative Name Server (example.com)
   → Resolver asks: "What is the IP for www.example.com?"
   → Authoritative server responds: "142.250.80.46" (with TTL)

7. Resolver caches the result and returns it to the client
8. Browser connects to 142.250.80.46
```

```
Browser → OS Cache → Recursive Resolver → Root NS → TLD NS → Authoritative NS
                                                                      |
                          ← ← ← ← ← IP Address (cached at each step) ←
```

---

#### DNS Record Types

| Record Type | Purpose | Example |
|-------------|---------|---------|
| **A** | Maps domain to IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | Maps domain to IPv6 address | `example.com → 2606:2800:220:1:248:1893:25c8:1946` |
| **CNAME** | Alias — points one domain to another | `www.example.com → example.com` |
| **MX** | Mail exchange — specifies mail servers | `example.com → mail.example.com (priority 10)` |
| **NS** | Name server — delegates a zone to a name server | `example.com → ns1.example.com` |
| **TXT** | Arbitrary text — used for verification, SPF, DKIM | `example.com → "v=spf1 include:_spf.google.com ~all"` |
| **SOA** | Start of Authority — zone metadata | Primary NS, admin email, serial number, refresh intervals |
| **SRV** | Service locator — specifies host and port for a service | `_sip._tcp.example.com → 5060 sipserver.example.com` |
| **PTR** | Reverse DNS — maps IP to domain | `34.216.184.93.in-addr.arpa → example.com` |
| **CAA** | Certificate Authority Authorization | Specifies which CAs can issue certificates for the domain |

**Key Records for System Design:**
- **A / AAAA:** Basic domain-to-IP mapping
- **CNAME:** Useful for CDN integration (point your domain to a CDN domain)
- **NS:** Delegation — used when you want a subdomain managed by a different DNS provider
- **MX:** Email routing
- **TXT:** Domain verification (Google, AWS), email security (SPF, DKIM, DMARC)

---

#### DNS Caching (TTL)

**TTL (Time To Live)** specifies how long a DNS record can be cached before it must be re-queried.

| TTL Value | Duration | Use Case |
|-----------|----------|----------|
| 60 seconds | 1 minute | Frequently changing IPs, failover scenarios |
| 300 seconds | 5 minutes | Common default, good balance |
| 3600 seconds | 1 hour | Stable services |
| 86400 seconds | 1 day | Very stable, rarely changing records |

**Caching happens at multiple levels:**
1. **Browser cache** — Chrome caches DNS for up to 1 minute
2. **OS cache** — Operating system DNS resolver cache
3. **Recursive resolver cache** — ISP or public DNS (8.8.8.8) caches based on TTL
4. **CDN/Edge cache** — CDN providers cache DNS responses

**Key Points for System Design:**
- **Low TTL** (60s): Use during migrations, failovers, or when IPs change frequently. Increases DNS query load.
- **High TTL** (3600s+): Use for stable services. Reduces DNS load but makes changes slow to propagate.
- **Before a migration:** Lower TTL days in advance, perform the migration, then raise TTL back.
- **DNS propagation delay:** Even after changing a record, old cached values persist until TTL expires across all caches worldwide.

---

### DNS in System Design

DNS plays a critical role beyond simple name resolution:

| Use Case | How DNS is Used |
|----------|----------------|
| **Load Balancing** | DNS round-robin returns multiple A records; clients connect to different IPs |
| **GeoDNS** | Returns different IPs based on the client's geographic location (route to nearest data center) |
| **Failover** | Health-checked DNS removes unhealthy server IPs from responses |
| **CDN Routing** | CNAME to CDN domain; CDN's DNS routes to nearest edge server |
| **Service Discovery** | SRV records or internal DNS for microservice discovery |
| **Blue-Green Deployment** | Switch DNS from blue to green environment |

**DNS Security Threats:**

| Threat | Description | Mitigation |
|--------|-------------|-----------|
| **DNS Spoofing / Cache Poisoning** | Attacker injects false DNS records into a resolver's cache, redirecting users to malicious servers | DNSSEC (DNS Security Extensions) — digitally signs DNS records |
| **DNS Amplification Attack (DDoS)** | Attacker sends small DNS queries with a spoofed source IP; DNS servers send large responses to the victim | Rate limiting, response rate limiting (RRL), BCP38 (ingress filtering) |
| **DNS Tunneling** | Attacker encodes data in DNS queries/responses to exfiltrate data or bypass firewalls | Monitor for unusually large or frequent DNS queries; DNS firewall |
| **DNS Hijacking** | Attacker compromises DNS settings (registrar, router) to redirect traffic | Registrar lock, DNSSEC, monitor DNS records |
| **Typosquatting** | Registering domains similar to popular ones (e.g., `gogle.com`) | Domain monitoring, CAA records |

**DNSSEC (DNS Security Extensions):**
- Adds digital signatures to DNS records
- Resolvers can verify that a DNS response hasn't been tampered with
- Chain of trust from root zone down to individual records
- Does NOT encrypt DNS queries (use DNS-over-HTTPS or DNS-over-TLS for privacy)

**DNS Privacy:**

| Protocol | Description | Port |
|----------|-------------|------|
| **DNS-over-HTTPS (DoH)** | DNS queries sent over HTTPS — encrypted and looks like normal web traffic | 443 |
| **DNS-over-TLS (DoT)** | DNS queries sent over TLS — encrypted but on a dedicated port | 853 |
| **DNSCrypt** | Encrypts DNS traffic between client and resolver | 443/5443 |

**NAT (Network Address Translation) — Related Concept:**

NAT allows multiple devices on a private network to share a single public IP address. It's essential for understanding how private IPs (10.x.x.x, 192.168.x.x) communicate with the internet.

```
Private Network:                    Internet:
  Device A (192.168.1.10) ─┐
  Device B (192.168.1.11) ─┼── NAT Router (Public IP: 203.0.113.5) ──── Server (93.184.216.34)
  Device C (192.168.1.12) ─┘
```

**How NAT Works:**
1. Device A sends a packet: Source=192.168.1.10:5000, Dest=93.184.216.34:80
2. NAT router rewrites: Source=203.0.113.5:12345, Dest=93.184.216.34:80 (maps internal IP:port to external IP:port)
3. Server responds to 203.0.113.5:12345
4. NAT router looks up the mapping, rewrites: Dest=192.168.1.10:5000
5. Device A receives the response

**NAT Types:**

| Type | Description | Use Case |
|------|-------------|----------|
| **Static NAT** | One-to-one mapping (private IP ↔ public IP) | Servers that need a fixed public IP |
| **Dynamic NAT** | Pool of public IPs assigned on demand | Organizations with multiple public IPs |
| **PAT (Port Address Translation)** | Many-to-one mapping using port numbers (most common) | Home routers, corporate networks |

**Key Point for System Design:** NAT is why you can't directly connect to a device behind a home router from the internet. This affects peer-to-peer applications (WebRTC, gaming) which use techniques like STUN, TURN, and ICE to traverse NAT.

---


## 15.4 HTTP/HTTPS

> **HTTP (HyperText Transfer Protocol)** is the foundation of data communication on the web. It's a request-response protocol that operates at the Application Layer (Layer 7). **HTTPS** is HTTP secured with TLS/SSL encryption. Understanding HTTP deeply is essential for API design, web architecture, caching strategies, and system design interviews.

---

### HTTP Methods

HTTP defines several **methods** (also called verbs) that indicate the desired action on a resource:

| Method | Purpose | Idempotent | Safe | Request Body | Use Case |
|--------|---------|-----------|------|-------------|----------|
| **GET** | Retrieve a resource | Yes | Yes | No (typically) | Fetch data, read operations |
| **POST** | Create a new resource / submit data | No | No | Yes | Create user, submit form, trigger action |
| **PUT** | Replace a resource entirely | Yes | No | Yes | Update entire user profile |
| **PATCH** | Partially update a resource | No* | No | Yes | Update just the email field |
| **DELETE** | Remove a resource | Yes | No | Optional | Delete a user |
| **HEAD** | Same as GET but returns only headers | Yes | Yes | No | Check if resource exists, get metadata |
| **OPTIONS** | Describe communication options | Yes | Yes | No | CORS preflight, discover allowed methods |

**Key Definitions:**
- **Safe:** Does not modify server state (GET, HEAD, OPTIONS)
- **Idempotent:** Multiple identical requests produce the same result as a single request (GET, PUT, DELETE, HEAD, OPTIONS)
- **POST is NOT idempotent:** Sending the same POST twice may create two resources

**Common Interview Question:** "What's the difference between PUT and PATCH?"
- **PUT** replaces the entire resource — you must send all fields
- **PATCH** updates only the specified fields — partial update

---

### HTTP Status Codes

Status codes indicate the result of an HTTP request:

**1xx — Informational:**

| Code | Meaning | Description |
|------|---------|-------------|
| 100 | Continue | Server received headers, client should send body |
| 101 | Switching Protocols | Server is switching to a different protocol (e.g., WebSocket upgrade) |

**2xx — Success:**

| Code | Meaning | Description |
|------|---------|-------------|
| 200 | OK | Request succeeded (GET, PUT, PATCH) |
| 201 | Created | Resource successfully created (POST) |
| 202 | Accepted | Request accepted for processing (async operations) |
| 204 | No Content | Success but no response body (DELETE) |

**3xx — Redirection:**

| Code | Meaning | Description |
|------|---------|-------------|
| 301 | Moved Permanently | Resource permanently moved to new URL (SEO-friendly) |
| 302 | Found | Temporary redirect (original URL should still be used) |
| 304 | Not Modified | Resource hasn't changed (use cached version) |
| 307 | Temporary Redirect | Like 302 but preserves HTTP method |
| 308 | Permanent Redirect | Like 301 but preserves HTTP method |

**4xx — Client Errors:**

| Code | Meaning | Description |
|------|---------|-------------|
| 400 | Bad Request | Malformed request syntax, invalid parameters |
| 401 | Unauthorized | Authentication required (missing or invalid credentials) |
| 403 | Forbidden | Authenticated but not authorized (insufficient permissions) |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | HTTP method not supported for this resource |
| 408 | Request Timeout | Server timed out waiting for the request |
| 409 | Conflict | Request conflicts with current state (e.g., duplicate resource) |
| 413 | Payload Too Large | Request body exceeds server limits |
| 429 | Too Many Requests | Rate limit exceeded |

**5xx — Server Errors:**

| Code | Meaning | Description |
|------|---------|-------------|
| 500 | Internal Server Error | Generic server error |
| 502 | Bad Gateway | Upstream server returned invalid response (proxy/LB issue) |
| 503 | Service Unavailable | Server temporarily overloaded or under maintenance |
| 504 | Gateway Timeout | Upstream server didn't respond in time |

**Key Codes for System Design:**
- **200, 201, 204** — Success responses for CRUD operations
- **301 vs 302** — Permanent vs temporary redirect (affects caching and SEO)
- **401 vs 403** — Authentication vs Authorization failure
- **429** — Rate limiting response (include `Retry-After` header)
- **502, 503, 504** — Common in distributed systems; indicate upstream/infrastructure issues

---

### Important HTTP Headers

**Request Headers:**

| Header | Purpose | Example |
|--------|---------|---------|
| `Host` | Target domain (required in HTTP/1.1) | `Host: api.example.com` |
| `Authorization` | Authentication credentials | `Authorization: Bearer <token>` |
| `Content-Type` | Format of request body | `Content-Type: application/json` |
| `Accept` | Desired response format | `Accept: application/json` |
| `User-Agent` | Client identification | `User-Agent: Mozilla/5.0...` |
| `Cookie` | Send stored cookies | `Cookie: session_id=abc123` |
| `If-None-Match` | Conditional request (ETag) | `If-None-Match: "etag-value"` |
| `If-Modified-Since` | Conditional request (date) | `If-Modified-Since: Wed, 21 Oct 2025 07:28:00 GMT` |

**Response Headers:**

| Header | Purpose | Example |
|--------|---------|---------|
| `Content-Type` | Format of response body | `Content-Type: application/json` |
| `Set-Cookie` | Store a cookie on the client | `Set-Cookie: session_id=abc123; HttpOnly; Secure` |
| `Cache-Control` | Caching directives | `Cache-Control: max-age=3600, public` |
| `ETag` | Resource version identifier | `ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"` |
| `Location` | Redirect URL (with 3xx status) | `Location: https://example.com/new-path` |
| `Retry-After` | When to retry (with 429 or 503) | `Retry-After: 60` |
| `Access-Control-Allow-Origin` | CORS — allowed origins | `Access-Control-Allow-Origin: *` |

---

### Caching Headers Deep Dive

HTTP caching is critical for performance in system design:

**`Cache-Control` Directives:**

| Directive | Meaning |
|-----------|---------|
| `public` | Response can be cached by any cache (browser, CDN, proxy) |
| `private` | Response can only be cached by the browser (not CDN/proxy) |
| `no-cache` | Must revalidate with server before using cached version |
| `no-store` | Do not cache at all (sensitive data) |
| `max-age=N` | Cache is valid for N seconds |
| `s-maxage=N` | Like max-age but for shared caches (CDN, proxy) |
| `must-revalidate` | Once stale, must revalidate before use |
| `immutable` | Resource will never change (use for versioned assets) |

**ETag-based Caching Flow:**

```
First Request:
Client → GET /api/users/1
Server → 200 OK
         ETag: "abc123"
         Body: { "name": "Alice" }

Second Request (with ETag):
Client → GET /api/users/1
         If-None-Match: "abc123"

If resource hasn't changed:
Server → 304 Not Modified (no body — use cached version)

If resource has changed:
Server → 200 OK
         ETag: "def456"
         Body: { "name": "Alice Updated" }
```

---

### Cookies and Sessions

**Cookies** are small pieces of data stored by the browser and sent with every request to the same domain.

**Cookie Attributes:**

| Attribute | Purpose |
|-----------|---------|
| `HttpOnly` | Cannot be accessed by JavaScript (prevents XSS theft) |
| `Secure` | Only sent over HTTPS |
| `SameSite=Strict` | Not sent with cross-site requests (prevents CSRF) |
| `SameSite=Lax` | Sent with top-level navigations but not embedded requests |
| `Domain` | Which domains receive the cookie |
| `Path` | Which paths receive the cookie |
| `Max-Age` / `Expires` | Cookie lifetime |

**Session Management Approaches:**

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Server-side sessions** | Session ID in cookie, session data on server (memory/Redis) | Secure, server controls data | Requires shared session store for horizontal scaling |
| **JWT (JSON Web Token)** | Token contains encoded user data, signed by server | Stateless, no server-side storage | Larger payload, can't revoke easily, token size grows |
| **Encrypted cookies** | Session data encrypted and stored in cookie | Stateless, simple | Limited size (4KB), encryption overhead |

**Key Point for System Design:** For horizontally scaled systems, server-side sessions require a shared session store (Redis, Memcached). JWTs are stateless but harder to revoke. Many systems use a hybrid: short-lived JWTs + refresh tokens stored server-side.

---

### HTTPS and TLS/SSL Handshake

**HTTPS = HTTP + TLS (Transport Layer Security)**

TLS encrypts the communication between client and server, providing:
- **Confidentiality:** Data is encrypted (can't be read by eavesdroppers)
- **Integrity:** Data can't be tampered with in transit
- **Authentication:** Server proves its identity via certificates

**TLS 1.2 Handshake (detailed):**

```
Client                                    Server
  |                                          |
  |  ---- ClientHello ------------------>    |  (supported cipher suites, TLS version, client random)
  |                                          |
  |  <--- ServerHello -------------------   |  (chosen cipher suite, server random)
  |  <--- Certificate -------------------   |  (server's X.509 certificate with public key)
  |  <--- ServerKeyExchange -------------   |  (DH parameters for key exchange, if using DHE/ECDHE)
  |  <--- CertificateRequest -----------   |  (optional: request client certificate for mutual TLS)
  |  <--- ServerHelloDone ---------------   |
  |                                          |
  |  ---- Certificate (optional) -------->   |  (client certificate, if requested)
  |  ---- ClientKeyExchange ------------>    |  (pre-master secret encrypted with server's public key,
  |                                          |   or client's DH public value)
  |  ---- CertificateVerify (optional) ->   |  (proof client owns the certificate)
  |  ---- ChangeCipherSpec ------------->    |  (switching to encrypted communication)
  |  ---- Finished (encrypted) --------->   |  (hash of all handshake messages — integrity check)
  |                                          |
  |  <--- ChangeCipherSpec ---------------  |
  |  <--- Finished (encrypted) -----------  |  (server's hash of all handshake messages)
  |                                          |
  |  ===== Encrypted Data Transfer =====    |
```

**Key Exchange Process:**
1. Client and server exchange random values (client random, server random)
2. Server sends its certificate (contains public key, signed by a Certificate Authority)
3. Client verifies the certificate chain (is it signed by a trusted CA? Is it expired? Does the domain match?)
4. Client generates a pre-master secret, encrypts it with the server's public key, sends it
5. Both sides derive the **master secret** from: pre-master secret + client random + server random
6. Both sides derive **session keys** from the master secret (separate keys for encryption and MAC, for each direction)
7. All subsequent data is encrypted with the session keys

**Forward Secrecy (Perfect Forward Secrecy — PFS):**
- With RSA key exchange: if the server's private key is compromised in the future, all past recorded traffic can be decrypted
- With **DHE/ECDHE** key exchange: each session uses a unique ephemeral key pair. Even if the server's private key is compromised, past sessions remain secure
- TLS 1.3 **mandates** forward secrecy — only ECDHE key exchange is allowed

**Mutual TLS (mTLS):**
- Standard TLS: only the server proves its identity (server certificate)
- Mutual TLS: both client AND server present certificates and verify each other
- Used in microservice-to-microservice communication (zero-trust networks)
- Service mesh (Istio, Linkerd) automates mTLS between services

**TLS 1.3 Handshake (faster):**

```
Client                                    Server
  |                                          |
  |  ---- ClientHello ------------------>    |  (supported cipher suites, key shares for ALL
  |        + Key Share                       |   supported groups — sent speculatively)
  |                                          |
  |  <--- ServerHello -------------------   |  (chosen cipher suite, server key share)
  |  <--- EncryptedExtensions -----------   |  (encrypted from this point!)
  |  <--- Certificate -------------------   |
  |  <--- CertificateVerify -------------   |
  |  <--- Finished -----------------------  |
  |                                          |
  |  ---- Finished ---------------------->   |
  |                                          |
  |  ===== Encrypted Data Transfer =====    |
  
  Total: 1 RTT (vs 2 RTT in TLS 1.2)
```

**TLS 1.3 Improvements:**
- **Faster:** 1-RTT handshake (vs 2-RTT in TLS 1.2)
- **0-RTT resumption:** Returning clients can send data immediately (with replay attack risk — server must handle idempotency)
- **Simpler:** Removed insecure cipher suites (RC4, DES, 3DES, MD5, SHA-1, static RSA, static DH)
- **More secure:** Forward secrecy is mandatory (only ECDHE)
- **Encrypted earlier:** Server certificate is encrypted (not visible to eavesdroppers)

**Certificate Chain of Trust:**

```
Root CA (pre-installed in OS/browser trust store)
  └── Intermediate CA (signed by Root CA)
       └── Server Certificate (signed by Intermediate CA)
            └── Your domain: api.example.com
```

- Browsers and operating systems ship with a list of trusted Root CAs (~100-150 CAs)
- Let's Encrypt provides free, automated certificates (used by ~300 million websites)
- Certificate Transparency (CT) logs provide public audit trail of all issued certificates

**Key Point:** TLS adds latency (handshake overhead). This is why HTTP/2 and HTTP/3 use connection multiplexing — to amortize the TLS handshake cost across many requests.

---

### CORS (Cross-Origin Resource Sharing)

**CORS** is a security mechanism that controls which web pages can make requests to your API from a different origin (domain, protocol, or port). Browsers enforce the **Same-Origin Policy** by default — JavaScript on `https://frontend.com` cannot make requests to `https://api.backend.com` unless the backend explicitly allows it via CORS headers.

**Same-Origin Policy:**
Two URLs have the same origin if they share the same **protocol**, **host**, and **port**:

| URL A | URL B | Same Origin? | Why |
|-------|-------|-------------|-----|
| `https://example.com/a` | `https://example.com/b` | Yes | Same protocol, host, port |
| `https://example.com` | `http://example.com` | No | Different protocol |
| `https://example.com` | `https://api.example.com` | No | Different host (subdomain) |
| `https://example.com` | `https://example.com:8080` | No | Different port |

**How CORS Works — Simple Requests:**

For simple requests (GET, HEAD, POST with standard content types), the browser sends the request directly and checks the response headers:

```
Browser (https://frontend.com) → GET https://api.backend.com/users
  Origin: https://frontend.com

Server responds:
  Access-Control-Allow-Origin: https://frontend.com   ← Browser allows the response
  (or Access-Control-Allow-Origin: *                   ← Allow any origin)
```

**How CORS Works — Preflight Requests:**

For complex requests (PUT, DELETE, custom headers, JSON content type), the browser sends a **preflight OPTIONS request** first:

```
Step 1: Preflight
  Browser → OPTIONS https://api.backend.com/users
    Origin: https://frontend.com
    Access-Control-Request-Method: DELETE
    Access-Control-Request-Headers: Authorization, Content-Type

  Server → 204 No Content
    Access-Control-Allow-Origin: https://frontend.com
    Access-Control-Allow-Methods: GET, POST, PUT, DELETE
    Access-Control-Allow-Headers: Authorization, Content-Type
    Access-Control-Max-Age: 86400  (cache preflight for 24 hours)

Step 2: Actual Request (only if preflight succeeds)
  Browser → DELETE https://api.backend.com/users/123
    Origin: https://frontend.com
    Authorization: Bearer <token>

  Server → 200 OK
    Access-Control-Allow-Origin: https://frontend.com
```

**CORS Headers:**

| Header | Direction | Purpose |
|--------|-----------|---------|
| `Access-Control-Allow-Origin` | Response | Which origins are allowed (`*` or specific origin) |
| `Access-Control-Allow-Methods` | Response | Which HTTP methods are allowed |
| `Access-Control-Allow-Headers` | Response | Which request headers are allowed |
| `Access-Control-Allow-Credentials` | Response | Whether cookies/auth headers are allowed |
| `Access-Control-Max-Age` | Response | How long to cache preflight results (seconds) |
| `Access-Control-Expose-Headers` | Response | Which response headers the browser can access |
| `Origin` | Request | The origin making the request |

**Key Point for System Design:** CORS is a browser-only security mechanism. Server-to-server requests (microservices, backend APIs) are not affected by CORS. Configure CORS at the API Gateway or reverse proxy level to avoid duplicating it in every service.

---

### HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC)

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Year | 1997 | 2015 | 2022 |
| Transport | TCP | TCP | **QUIC (over UDP)** |
| Multiplexing | No (one request per connection) | Yes (multiple streams per connection) | Yes |
| Head-of-Line Blocking | Yes (at TCP level) | Partially (at TCP level) | **No** (QUIC handles per-stream) |
| Header Compression | No | HPACK | QPACK |
| Server Push | No | Yes | Yes |
| Connection Setup | TCP + TLS (2-3 RTT) | TCP + TLS (2-3 RTT) | **1 RTT (0-RTT for resumption)** |
| Binary Protocol | No (text-based) | Yes | Yes |
| Encryption | Optional (HTTP) | Practically required | **Always encrypted (built into QUIC)** |

**HTTP/1.1 Problems:**
- **Head-of-Line (HOL) Blocking:** Only one request can be in-flight per connection. Browsers open 6-8 parallel connections as a workaround.
- **Redundant Headers:** Headers are sent in full with every request (often repetitive)
- **No Multiplexing:** Each request needs its own connection or must wait

**HTTP/2 Solutions:**
- **Multiplexing:** Multiple requests/responses over a single TCP connection
- **Header Compression (HPACK):** Reduces header overhead significantly
- **Stream Prioritization:** Important resources can be prioritized
- **Server Push:** Server can proactively send resources the client will need
- **Still has TCP HOL blocking:** If a TCP packet is lost, ALL streams are blocked until retransmission

**HTTP/3 (QUIC) Solutions:**
- **Built on UDP:** Avoids TCP's HOL blocking entirely
- **Per-stream flow control:** A lost packet only blocks its own stream, not others
- **Faster connection setup:** QUIC combines transport and crypto handshake (1-RTT, 0-RTT for resumption)
- **Connection migration:** Connections survive IP changes (e.g., switching from Wi-Fi to cellular)
- **Always encrypted:** TLS 1.3 is built into the protocol

---

### Keep-Alive Connections

**HTTP/1.0:** Each request opened a new TCP connection (expensive — 3-way handshake + TLS for every request).

**HTTP/1.1 Keep-Alive:** Connections are persistent by default — multiple requests can reuse the same TCP connection.

```
Without Keep-Alive (HTTP/1.0):
  TCP Handshake → Request 1 → Response 1 → Close
  TCP Handshake → Request 2 → Response 2 → Close
  TCP Handshake → Request 3 → Response 3 → Close

With Keep-Alive (HTTP/1.1):
  TCP Handshake → Request 1 → Response 1
                → Request 2 → Response 2
                → Request 3 → Response 3
                → Close (after idle timeout)
```

**Headers:**
- `Connection: keep-alive` (default in HTTP/1.1)
- `Connection: close` (explicitly close after response)
- `Keep-Alive: timeout=5, max=100` (keep connection open for 5s, max 100 requests)

**Key Point for System Design:** Keep-alive reduces latency and server load by avoiding repeated TCP/TLS handshakes. However, persistent connections consume server resources (memory, file descriptors). Load balancers and reverse proxies must manage connection pooling carefully.

---


## 15.5 WebSockets

> **WebSockets** provide **full-duplex, bidirectional communication** over a single, long-lived TCP connection. Unlike HTTP's request-response model, WebSockets allow both the client and server to send messages at any time without the overhead of establishing new connections. This makes them ideal for real-time applications.

---

### Full-Duplex Communication

**Half-Duplex (HTTP):** Communication goes one way at a time — client sends a request, waits for a response.
**Full-Duplex (WebSocket):** Both client and server can send messages simultaneously, at any time.

```
HTTP (Half-Duplex):
Client → Request  → Server
Client ← Response ← Server
Client → Request  → Server
Client ← Response ← Server

WebSocket (Full-Duplex):
Client ←→ Server  (both can send at any time)
  |  "Hello"  →  |
  |  ← "Hi"      |
  |  "Update" →  |
  |  ← "Data1"   |
  |  ← "Data2"   |   (server pushes without client asking)
  |  "Bye"    →  |
```

**Key Characteristics:**
- **Persistent connection:** Opened once, stays open until explicitly closed
- **Low overhead:** After the initial handshake, messages have minimal framing (2-14 bytes vs hundreds of bytes for HTTP headers)
- **Bidirectional:** Server can push data to client without being asked
- **Event-driven:** Both sides react to incoming messages asynchronously

---

### WebSocket Handshake (Upgrade from HTTP)

WebSocket connections start as a regular HTTP request and are **upgraded** to the WebSocket protocol:

```
Client → Server (HTTP Upgrade Request):
  GET /chat HTTP/1.1
  Host: server.example.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13

Server → Client (HTTP 101 Switching Protocols):
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

  === WebSocket connection established ===
  === Both sides can now send frames ===
```

**How it works:**
1. Client sends a regular HTTP GET request with `Upgrade: websocket` header
2. Server responds with `101 Switching Protocols` if it supports WebSockets
3. The TCP connection is now "upgraded" — HTTP is no longer used on this connection
4. Both sides communicate using WebSocket frames (binary or text)

**The `Sec-WebSocket-Key` / `Sec-WebSocket-Accept` exchange:**
- Client sends a random base64-encoded key
- Server concatenates it with a magic GUID (`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`), hashes with SHA-1, and returns the base64 result
- This proves the server understands the WebSocket protocol (not just any HTTP server)
- This is NOT for security — it prevents accidental WebSocket connections from non-WebSocket clients

---

### WebSocket Frame Format

After the handshake, all communication uses **WebSocket frames** — lightweight binary envelopes around the actual data:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|     Masking-key (0 or 4 bytes)                                |
+---------------------------------------------------------------+
|     Payload Data                                              |
+---------------------------------------------------------------+
```

**Frame Fields:**

| Field | Size | Description |
|-------|------|-------------|
| FIN | 1 bit | 1 = this is the final fragment of a message; 0 = more fragments follow |
| Opcode | 4 bits | Frame type: 0x1 = text, 0x2 = binary, 0x8 = close, 0x9 = ping, 0xA = pong |
| MASK | 1 bit | 1 = payload is masked (client → server frames MUST be masked) |
| Payload length | 7 bits | 0-125: actual length; 126: next 2 bytes are length; 127: next 8 bytes are length |
| Masking key | 0 or 4 bytes | XOR key for unmasking payload (only if MASK=1) |
| Payload | variable | The actual message data |

**Frame Types (Opcodes):**

| Opcode | Type | Purpose |
|--------|------|---------|
| 0x0 | Continuation | Continuation of a fragmented message |
| 0x1 | Text | UTF-8 text data (JSON messages, chat text) |
| 0x2 | Binary | Binary data (images, protobuf, audio) |
| 0x8 | Close | Connection close request (includes status code) |
| 0x9 | Ping | Heartbeat request (server or client) |
| 0xA | Pong | Heartbeat response (must respond to ping with pong) |

**Key Point:** WebSocket frames have only 2-14 bytes of overhead (vs hundreds of bytes for HTTP headers). This makes WebSockets extremely efficient for frequent, small messages like chat or real-time updates.

**WebSocket Close Codes:**

| Code | Meaning |
|------|---------|
| 1000 | Normal closure |
| 1001 | Going away (server shutting down, browser navigating away) |
| 1002 | Protocol error |
| 1003 | Unsupported data type |
| 1006 | Abnormal closure (no close frame received — connection dropped) |
| 1008 | Policy violation |
| 1009 | Message too big |
| 1011 | Unexpected server error |

---

### WebSocket Connection Lifecycle

```
1. HTTP Upgrade Handshake
   Client → GET /ws (Upgrade: websocket)
   Server → 101 Switching Protocols
   
2. Open State — bidirectional communication
   Client ←→ Server (text/binary frames)
   
   Heartbeat (keep-alive):
   Server → Ping frame (every 30s)
   Client → Pong frame (response)
   
3. Closing Handshake
   Client → Close frame (code: 1000, reason: "done")
   Server → Close frame (code: 1000, reason: "acknowledged")
   TCP connection closed
   
4. Error / Abnormal Close
   Network drops → No close frame → Code 1006
   Client must implement reconnection logic with exponential backoff
```

**Reconnection Strategy (Exponential Backoff):**

```
Attempt 1: Wait 1 second, reconnect
Attempt 2: Wait 2 seconds, reconnect
Attempt 3: Wait 4 seconds, reconnect
Attempt 4: Wait 8 seconds, reconnect
Attempt 5: Wait 16 seconds, reconnect
...
Max wait: 60 seconds (cap the backoff)
Add jitter: ±random(0, 1 second) to prevent thundering herd
```

---

### Use Cases

| Use Case | Why WebSockets? |
|----------|----------------|
| **Chat applications** | Real-time message delivery to all participants |
| **Live notifications** | Server pushes notifications instantly (no polling) |
| **Live sports scores / stock tickers** | Continuous real-time data updates |
| **Online gaming** | Low-latency bidirectional communication for game state |
| **Collaborative editing** | Real-time sync of document changes (Google Docs) |
| **Live dashboards** | Real-time metrics, monitoring data |
| **IoT device communication** | Continuous sensor data streaming |
| **Live auctions** | Real-time bid updates |

---

### WebSocket vs Long Polling vs Server-Sent Events (SSE)

When you need real-time updates from server to client, there are several approaches:

**1. Short Polling:**
```
Client → GET /updates (every 5 seconds)
Server → 200 OK (no new data)
Client → GET /updates (5 seconds later)
Server → 200 OK (no new data)
Client → GET /updates (5 seconds later)
Server → 200 OK { "message": "new data!" }
```
- Simple but wasteful — many empty responses
- High latency (up to polling interval)
- High server load from frequent requests

**2. Long Polling:**
```
Client → GET /updates
Server → ... holds connection open ...
Server → ... waits for new data ...
Server → 200 OK { "message": "new data!" }  (responds when data is available)
Client → GET /updates  (immediately reconnects)
Server → ... holds connection open again ...
```
- More efficient than short polling — fewer empty responses
- Near real-time (responds as soon as data is available)
- Still has overhead of re-establishing HTTP connection after each response
- Server must manage many open connections

**3. Server-Sent Events (SSE):**
```
Client → GET /events (Accept: text/event-stream)
Server → 200 OK (Content-Type: text/event-stream)
         data: {"message": "update 1"}

         data: {"message": "update 2"}

         data: {"message": "update 3"}
         ... connection stays open, server keeps pushing ...
```
- **Unidirectional:** Server → Client only
- Built on HTTP (works with existing infrastructure, proxies, load balancers)
- Automatic reconnection built into the browser API
- Text-based only (no binary)
- Simple to implement

**4. WebSocket:**
```
Client ←→ Server (full-duplex after upgrade)
```
- **Bidirectional:** Both sides can send at any time
- Lowest latency and overhead after connection is established
- Supports binary and text data
- Requires WebSocket-aware infrastructure (load balancers, proxies)

---

**Comparison Table:**

| Feature | Short Polling | Long Polling | SSE | WebSocket |
|---------|--------------|-------------|-----|-----------|
| Direction | Client → Server | Client → Server | Server → Client | Bidirectional |
| Latency | High (polling interval) | Medium | Low | Lowest |
| Connection overhead | High (new connection each poll) | Medium (reconnect after each response) | Low (single connection) | Lowest (single connection) |
| Server resource usage | Low per request, high total | Medium (many open connections) | Medium | Low (efficient framing) |
| Protocol | HTTP | HTTP | HTTP | WebSocket (over TCP) |
| Binary support | Yes (via HTTP) | Yes (via HTTP) | No (text only) | Yes |
| Browser support | Universal | Universal | Good (no IE) | Good (no IE) |
| Infrastructure compatibility | Excellent | Good | Good | Requires WS-aware LB/proxy |
| Automatic reconnection | N/A (client controls) | Client must implement | Built-in | Client must implement |
| Best for | Simple, infrequent updates | Near real-time, moderate traffic | Server-push notifications, feeds | Chat, gaming, collaborative apps |

**Decision Guide:**
- **Need bidirectional communication?** → WebSocket
- **Only server-to-client updates?** → SSE (simpler than WebSocket)
- **Can't use WebSocket/SSE (infrastructure constraints)?** → Long Polling
- **Infrequent updates, simplicity is key?** → Short Polling

---

### WebSocket Scaling Challenges

WebSockets introduce unique challenges for system design:

**1. Connection State:**
- Each WebSocket connection is stateful and tied to a specific server
- Unlike HTTP (stateless), you can't simply round-robin requests across servers
- Need sticky sessions or a connection registry

**2. Load Balancing:**
- Layer 7 load balancers must support WebSocket upgrade
- Sticky sessions or connection-aware routing required
- Nginx, HAProxy, and AWS ALB support WebSocket

**3. Horizontal Scaling:**
- If User A is connected to Server 1 and User B to Server 2, how does A send a message to B?
- Solution: Use a **pub/sub system** (Redis Pub/Sub, Kafka) as a message bus between servers

```
User A → Server 1 → Redis Pub/Sub → Server 2 → User B
```

**4. Connection Limits:**
- Each WebSocket connection consumes a file descriptor and memory
- A single server might handle 10K-100K concurrent connections (depending on resources)
- Plan capacity accordingly

**5. Heartbeats / Keep-Alive:**
- WebSocket connections can be silently dropped by proxies, firewalls, or NAT devices
- Implement ping/pong frames to detect dead connections
- Typical interval: every 30-60 seconds

---


## 15.6 Other Protocols

> Beyond HTTP and WebSockets, several other protocols are important for system design. These protocols power email, file transfer, IoT communication, and message queuing systems. Understanding when and why to use each protocol is key to making informed architectural decisions.

---

### FTP (File Transfer Protocol)

**FTP** is a standard protocol for transferring files between a client and server over a TCP connection.

**Key Characteristics:**

| Feature | Details |
|---------|---------|
| Port | 21 (control), 20 (data) |
| Transport | TCP |
| Authentication | Username/password (sent in plaintext!) |
| Modes | Active mode, Passive mode |
| Secure variants | FTPS (FTP over TLS), SFTP (SSH File Transfer Protocol) |

**Active vs Passive Mode:**

```
Active Mode:
  Client opens port 21 → Server (control connection)
  Server opens port 20 → Client (data connection)
  Problem: Client firewall may block incoming connection from server

Passive Mode:
  Client opens port 21 → Server (control connection)
  Client opens random port → Server's random port (data connection)
  Both connections initiated by client — firewall-friendly
```

**FTP vs Modern Alternatives:**

| Protocol | Security | Use Case |
|----------|----------|----------|
| FTP | None (plaintext) | Legacy systems only — avoid for new designs |
| FTPS | TLS encryption | Secure file transfer (FTP + TLS) |
| SFTP | SSH encryption | Secure file transfer (runs over SSH, port 22) |
| SCP | SSH encryption | Simple secure copy (no directory listing) |
| S3 / Object Storage | HTTPS | Cloud-native file storage — preferred for modern systems |

**Key Point for System Design:** FTP is largely replaced by object storage (S3), SFTP, or HTTP-based file upload APIs in modern architectures. You'll rarely design a system using FTP, but understanding it helps when dealing with legacy integrations.

---

### SMTP, IMAP, POP3 (Email Protocols)

Email uses multiple protocols for different functions:

**SMTP (Simple Mail Transfer Protocol) — Sending Email:**

| Feature | Details |
|---------|---------|
| Port | 25 (relay), 587 (submission with TLS), 465 (implicit TLS) |
| Transport | TCP |
| Direction | Client → Server, Server → Server |
| Purpose | Sending and relaying email |

```
Email Sending Flow:
  Sender's Client → (SMTP) → Sender's Mail Server
                              → (SMTP) → Recipient's Mail Server
                                          → Stored in mailbox
  Recipient's Client ← (IMAP/POP3) ← Recipient's Mail Server
```

**IMAP (Internet Message Access Protocol) — Reading Email:**

| Feature | Details |
|---------|---------|
| Port | 143 (plaintext), 993 (TLS) |
| Transport | TCP |
| Purpose | Access and manage email on the server |
| Key feature | Emails stay on server; supports folders, search, flags |
| Multi-device | Yes — changes sync across all devices |

**POP3 (Post Office Protocol v3) — Downloading Email:**

| Feature | Details |
|---------|---------|
| Port | 110 (plaintext), 995 (TLS) |
| Transport | TCP |
| Purpose | Download email from server to client |
| Key feature | Typically downloads and deletes from server |
| Multi-device | No — email is on one device after download |

**IMAP vs POP3:**

| Feature | IMAP | POP3 |
|---------|------|------|
| Email storage | Server (synced) | Client (downloaded) |
| Multi-device access | Yes | No |
| Offline access | Partial (cached) | Full (downloaded) |
| Server storage needed | More | Less |
| Bandwidth | More (syncing) | Less (download once) |
| Modern usage | Preferred | Legacy |

**Key Point for System Design:** When designing notification or email systems, you'll use SMTP for sending (often via services like AWS SES, SendGrid, or Mailgun). IMAP/POP3 are relevant when building email clients or email processing pipelines.

---

### MQTT (Message Queuing Telemetry Transport) — IoT

**MQTT** is a lightweight publish-subscribe messaging protocol designed for constrained devices and low-bandwidth, high-latency networks. It's the de facto standard for **IoT (Internet of Things)** communication.

**Key Characteristics:**

| Feature | Details |
|---------|---------|
| Port | 1883 (plaintext), 8883 (TLS) |
| Transport | TCP |
| Pattern | Publish-Subscribe |
| Message size | Minimal overhead (2-byte fixed header) |
| QoS Levels | 0 (at most once), 1 (at least once), 2 (exactly once) |
| Designed for | Low bandwidth, unreliable networks, constrained devices |

**Architecture:**

```
+----------+                    +----------+
| Publisher | ---- publish ---> |  MQTT    | ---- deliver ---> +------------+
| (Sensor) |   topic: temp/    |  Broker  |   topic: temp/    | Subscriber |
+----------+   payload: 72°F   | (Server) |   payload: 72°F   | (Dashboard)|
                                +----------+                    +------------+
                                     |
                                     | ---- deliver --->  +------------+
                                     |   topic: temp/     | Subscriber |
                                     |   payload: 72°F    | (Alert Svc)|
                                     |                    +------------+
```

**QoS Levels:**

| QoS | Name | Guarantee | Use Case |
|-----|------|-----------|----------|
| 0 | At most once | Fire and forget — no acknowledgment | Sensor data where occasional loss is OK |
| 1 | At least once | Acknowledged — may deliver duplicates | Important data where duplicates are acceptable |
| 2 | Exactly once | Four-step handshake — guaranteed once | Critical data (billing, commands) |

**MQTT Features:**
- **Retained Messages:** Broker stores the last message on a topic; new subscribers get it immediately
- **Last Will and Testament (LWT):** If a client disconnects unexpectedly, the broker publishes a predefined message
- **Topic Wildcards:** `sensors/+/temperature` (single level), `sensors/#` (multi-level)
- **Persistent Sessions:** Broker stores subscriptions and queued messages for offline clients

**Key Point for System Design:** MQTT is ideal for IoT systems with thousands of devices sending small, frequent messages. For server-to-server messaging in microservices, use Kafka or RabbitMQ instead.

---

### AMQP (Advanced Message Queuing Protocol) — Message Queuing

**AMQP** is an open standard protocol for message-oriented middleware. **RabbitMQ** is the most popular AMQP implementation.

**Key Characteristics:**

| Feature | Details |
|---------|---------|
| Port | 5672 (plaintext), 5671 (TLS) |
| Transport | TCP |
| Pattern | Producer → Exchange → Queue → Consumer |
| Reliability | Acknowledgments, persistence, transactions |
| Designed for | Enterprise messaging, microservice communication |

**AMQP Architecture:**

```
+----------+     +-----------+     +-------+     +----------+
| Producer | --> | Exchange  | --> | Queue | --> | Consumer |
+----------+     +-----------+     +-------+     +----------+
                  |  Types:   |
                  |  - Direct |     +-------+     +----------+
                  |  - Fanout | --> | Queue | --> | Consumer |
                  |  - Topic  |     +-------+     +----------+
                  |  - Headers|
                  +-----------+
```

**Exchange Types:**

| Type | Routing Logic | Use Case |
|------|--------------|----------|
| **Direct** | Routes to queue with matching routing key | Task distribution, RPC |
| **Fanout** | Routes to ALL bound queues (broadcast) | Notifications, event broadcasting |
| **Topic** | Routes based on routing key pattern matching (wildcards) | Log routing (e.g., `*.error`, `payment.#`) |
| **Headers** | Routes based on message header attributes | Complex routing rules |

**AMQP vs MQTT:**

| Feature | AMQP | MQTT |
|---------|------|------|
| Complexity | Complex (exchanges, bindings, queues) | Simple (topics only) |
| Message size | Larger overhead | Minimal overhead |
| Target | Enterprise, microservices | IoT, constrained devices |
| Routing | Flexible (exchange types) | Simple (topic-based) |
| QoS | Acknowledgments, transactions | 3 QoS levels |
| Broker | RabbitMQ, ActiveMQ | Mosquitto, HiveMQ, EMQX |

---

### Protocol Summary for System Design

| Protocol | Layer | Transport | Use Case | When to Use in System Design |
|----------|-------|-----------|----------|------------------------------|
| HTTP/HTTPS | Application | TCP | Web, APIs | REST APIs, web services, microservice communication |
| WebSocket | Application | TCP | Real-time bidirectional | Chat, gaming, live updates, collaborative editing |
| gRPC | Application | HTTP/2 (TCP) | Microservice RPC | Low-latency inter-service communication |
| DNS | Application | UDP (TCP for large) | Name resolution | Load balancing, service discovery, failover |
| FTP/SFTP | Application | TCP | File transfer | Legacy file transfer, batch processing |
| SMTP | Application | TCP | Email sending | Notification systems, email services |
| MQTT | Application | TCP | IoT messaging | IoT device communication, sensor data |
| AMQP | Application | TCP | Message queuing | Async microservice communication, event-driven systems |
| TCP | Transport | — | Reliable delivery | When you need guaranteed, ordered delivery |
| UDP | Transport | — | Fast, unreliable delivery | Streaming, gaming, DNS, when speed > reliability |
| QUIC | Transport | UDP | Modern web | HTTP/3, low-latency web communication |
| IP | Network | — | Routing | Fundamental — every internet communication uses IP |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| OSI Model | 7 layers — understand what happens at each layer and which protocols/devices operate there |
| TCP/IP Model | 4 layers — the practical model that powers the internet |
| TCP | Reliable, ordered, connection-oriented — use for APIs, databases, file transfer |
| UDP | Fast, unreliable, connectionless — use for streaming, gaming, DNS |
| Three-Way Handshake | SYN → SYN-ACK → ACK — establishes TCP connection |
| IPv4 vs IPv6 | 32-bit vs 128-bit addresses; IPv4 exhaustion drove IPv6 adoption |
| Subnetting | CIDR notation (/24 = 254 hosts); critical for cloud VPC design |
| DNS | Hierarchical resolution: Browser → OS → Resolver → Root → TLD → Authoritative |
| DNS Records | A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), NS (nameserver), TXT (verification) |
| HTTP Methods | GET (read), POST (create), PUT (replace), PATCH (partial update), DELETE (remove) |
| HTTP Status Codes | 2xx (success), 3xx (redirect), 4xx (client error), 5xx (server error) |
| HTTPS/TLS | Encryption + integrity + authentication; TLS 1.3 is faster (1-RTT) |
| HTTP/2 | Multiplexing, header compression, server push — over TCP |
| HTTP/3 (QUIC) | Built on UDP, no HOL blocking, faster connection setup, connection migration |
| WebSockets | Full-duplex, bidirectional, persistent — for real-time apps |
| SSE | Server-to-client only, simpler than WebSocket, built on HTTP |
| Long Polling | HTTP-based near real-time — fallback when WebSocket/SSE aren't available |
| MQTT | Lightweight pub/sub for IoT — minimal overhead, QoS levels |
| AMQP | Enterprise messaging — exchanges, queues, routing (RabbitMQ) |

---

## Interview Tips for Module 15

1. **Know the OSI layers** — interviewers often ask "at which layer does X operate?" (e.g., load balancer, firewall, TLS)
2. **TCP vs UDP** is a classic question — know the trade-offs and when to use each
3. **Three-way handshake** — be able to draw and explain SYN → SYN-ACK → ACK
4. **DNS resolution** — understand the full flow from browser to authoritative nameserver
5. **HTTP status codes** — know the important ones (200, 201, 301, 302, 400, 401, 403, 404, 429, 500, 502, 503)
6. **HTTP/2 vs HTTP/3** — understand multiplexing, HOL blocking, and why QUIC uses UDP
7. **WebSocket vs SSE vs Long Polling** — know when to use each for real-time features
8. **TLS handshake** — understand the basics of how HTTPS works
9. **DNS in system design** — DNS-based load balancing, GeoDNS, failover, CDN routing
10. **Caching headers** — `Cache-Control`, `ETag`, `If-None-Match` — critical for performance discussions


