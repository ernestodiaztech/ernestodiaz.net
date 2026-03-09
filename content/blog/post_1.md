---
title: Day 1 of CCNA Study
date: 2026-03-01
description: This is a blog series of me studying for the CCNA. I learn best when reiterating the material or "teaching". The part focuses on the OSI & TCP/IP models.
tags:
    - ccna
toc: false
---

I learn best when writing things down and when teaching someone. This accomplishes both. A lot of sections might be hard to follow since I'm writing down things I feel like I should remember and can easily reference later.

## Networking Foundations & the OSI/TCP-IP Models

Alright, Part 1. I work as a network admin/engineer, so I deal with networking every single day. But when I started reviewing for the CCNA, I quickly realized the exam wants a level of precision about the foundational models that I honestly don't think about at work. I know that data goes in, packets come out, and if something breaks I can troubleshoot it. But can I tell you the exact PDU name at Layer 4 vs Layer 2? Can I list what the Presentation layer actually does? That's what the CCNA demands, so that's what I'm locking down first.

---

## The OSI Model

The OSI (Open Systems Interconnection) model is a conceptual framework. It doesn't describe how any real protocol suite works exactly. It's a reference model for understanding how data moves across a network in discrete, logical steps. The CCNA treats this as gospel, and questions will ask you to identify which layer a protocol operates at, or what happens at a specific layer during encapsulation.

Here's how I think about each layer, top to bottom:

{{% steps %}}

### Layer 7 - Application

This is the layer closest to the end user. It provides network services directly to applications (things like web browsers, email clients, and file transfer tools). It's not the application itself; it's the 'network interface' that the application uses.

Protocols that live here: HTTP, HTTPS, FTP, TFTP, SSH, Telnet, DNS, DHCP, SNMP, SMTP, POP3, IMAP.

{{< callout type="info" >}}
**Study Tip - Memorizing Layer 7 Protocols:** There are a lot of protocols at this layer, and the exam expects you to associate all of them with Layer 7 instantly. The technique that worked for me was grouping them by function:

- **Remote access:** SSH, Telnet
- **File transfer:** FTP, TFTP
- **Web:** HTTP, HTTPS
- **Email:** SMTP (send), POP3/IMAP (receive)
- **Infrastructure:** DNS (names), DHCP (IPs), SNMP (monitoring)

Once I mentally filed them into these buckets, I stopped mixing up which layer they belonged to. The trick is that all of these are Layer 7, the grouping just helps recall.
{{< /callout >}}

{{< callout type="warning" >}}
The exam loves to test whether you know that DNS and DHCP operate at Layer 7 (Application), not Layer 3. They use IP at Layer 3, but the protocols themselves are Application layer protocols. Don't confuse where a protocol operates vs what layers it depends on.
{{< /callout >}}

---

### Layer 6 - Presentation

This layer handles data formatting, encryption/decryption, and compression. It translates data between the format the application uses and the format the network uses. Think of it as the "translator."

Examples: SSL/TLS encryption, JPEG/GIF formatting, ASCII/EBCDIC conversion.

In modern networking, Layers 5-7 are often lumped together (which is exactly what the TCP/IP model does), but the CCNA still expects you to know what each one is responsible for individually.

{{< callout type="info" >}}
**Study Tip — Memorizing Layers 5-7:** These three layers are the hardest to keep straight because they're so abstract. The mental anchor I used is to think of a phone call:

- **Layer 7 (Application):** The *reason* for the call - "I need to order a pizza." The actual service being used.
- **Layer 6 (Presentation):** The *language* you speak - "Let's both speak English, and I'll encrypt my credit card number." Formatting and encryption.
- **Layer 5 (Session):** The *call itself* - dialing, keeping the line open, hanging up. Managing the conversation.

If an exam question mentions "formatting data," "encryption," or "compression" → Layer 6. If it mentions "establishing/maintaining/terminating a dialog" → Layer 5. If it mentions a specific protocol or network service → Layer 7.
{{< /callout >}}

---

### Layer 5 - Session

The Session layer manages the dialog between two hosts. It establishes, maintains, and terminates sessions (conversations). It also handles synchronization, think checkpoints in a data transfer so that if something fails, you don't have to restart from the beginning.

Examples: NetBIOS, RPC (Remote Procedure Call), PPTP session management.

{{< callout type="info" >}}
**Tip:** For the exam, the quick way to remember Layers 5-7 is: **Session** = manages the conversation, **Presentation** = formats/encrypts the data, **Application** = provides the network service to the user's app. If a question describes "establishing a dialog" or "synchronization," it's Layer 5.
{{< /callout >}}

---

### Layer 4 - Transport

This is where things start getting real for someone who works in networking daily. The Transport layer provides end-to-end communication between hosts. The two major protocols here are TCP and UDP.

Key responsibilities:
- **Segmentation** - breaking application data into segments
- **Flow control** - making sure the sender doesn't overwhelm the receiver (TCP windowing)
- **Error recovery** - retransmitting lost segments (TCP)
- **Port numbering** - multiplexing multiple conversations over a single IP address

The PDU at this layer is called a segment (TCP) or datagram (UDP).

{{< callout type="info" >}}
**Study Tip - Memorizing PDU Names:** The PDU names trip people up because they sound similar. I used this sentence to lock them in:

"**D**on't **S**top **P**ouring **F**ree **B**eer" → **D**ata (L7-5), **S**egment (L4), **P**acket (L3), **F**rame (L2), **Bits** (L1)

{{< /callout >}}

---

### Layer 3 - Network

This is the routing layer. It handles logical addressing (IP addresses) and path determination (routing). Routers operate at this layer. Every time I look at a routing table at work, I'm working at Layer 3.

Key responsibilities:
- Logical addressing (IPv4 and IPv6)
- Routing and path selection
- Packet forwarding

The PDU at this layer is called a 'packet'.

Protocols: IP (IPv4/IPv6), ICMP, OSPF, EIGRP, ARP (though ARP is a weird one).

{{< callout type="warning" >}}
ARP is a headache for layer classification. It operates between Layer 2 and Layer 3. It resolves Layer 3 addresses (IP) to Layer 2 addresses (MAC). Some sources say Layer 2, some say Layer 3, and Cisco's official material tends to place it at Layer 3. For the CCNA, if forced to pick, go with 'Layer 3'. But the real answer is that it bridges both.
{{< /callout >}}

---

### Layer 2 - Data Link

This is the switching layer. It handles physical addressing (MAC addresses), frame formatting, error detection (not correction), and media access control. Switches operate here.

Layer 2 is actually split into two sublayers:
- **LLC (Logical Link Control)** - interfaces with Layer 3, handles flow control and error notification
- **MAC (Media Access Control)** - handles physical addressing and media access (CSMA/CD for Ethernet)

The PDU at this layer is called a 'frame'.

Protocols/standards: Ethernet (802.3), Wi-Fi (802.11), PPP, Frame Relay (legacy).

{{< callout type="info" >}}
**Study Tip - Remembering the L2 Sublayers:** I kept mixing up LLC and MAC until I used this trick: **LLC = "Logical Link to Layer 3"** (it interfaces *up* to the Network layer) and **MAC = "Media Access for the Cable"** (it interfaces *down* to the physical medium). LLC talks up, MAC talks down. Quick and clean.
{{< /callout >}}

---

### Layer 1 - Physical

The Physical layer deals with the actual transmission of raw bits over a physical medium. It defines things like cable types, connector types, voltage levels, signal encoding, and data rates.

Examples: Ethernet cables (Cat5e/Cat6), fiber optics, RJ-45 connectors, wireless radio frequencies.

The PDU at this layer is a 'bit'.

{{< callout type="info" >}}
A common exam question format is: "At which layer of the OSI model does [device/protocol] operate?" Here's a quick reference:

- **Hub/Repeater** → Layer 1
- **Switch/Bridge** → Layer 2
- **Router** → Layer 3
- **Firewall** → Varies (can be L3, L4, or L7 depending on type)
{{< /callout >}}

---

### Layer Mnemonic

Everyone has a mnemonic for remembering the layers. The classic one going from Layer 7 down is:

> **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing

(Application, Presentation, Session, Transport, Network, Data Link, Physical)

Or from Layer 1 up:

> **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

Use whichever sticks. I personally just drilled the layers by association with protocols and devices until I didn't need the mnemonic anymore.

---

### Study Tips for the OSI Model

What worked for me here was a two-pass approach. First, I read through the OCG (Wendell Odom's *CCNA 200-301 Official Cert Guide*, Volume 1, Chapters 1-3) to get the "Cisco-approved" version of everything. Then I watched a video walkthrough to hear someone explain it differently, sometimes a second explanation makes something click that reading alone didn't.

The key is to not just *read* the layers but to actively quiz yourself. After studying, close the book and try to write all 7 layers from memory with their PDU, at least two protocols each, and a device that operates there. If you can't do that without peeking, you're not ready to move on.

**Resources I used for this section:**

- **OCG Volume 1, Chapters 1-3** - This is the primary source. Wendell Odom covers the models in exhaustive detail. If something I wrote here doesn't match the OCG, go with the OCG since it's what the exam is based on.
- **Jeremy's IT Lab (YouTube)** - [Free CCNA Course, Day 3: OSI Model & TCP/IP Suite](https://www.youtube.com/watch?v=t-ai8JzhHuY) - Jeremy is methodical and visual. This is one of the best free explanations of the OSI model I've found. His entire CCNA playlist is gold.
- **Neil Anderson's CCNA Course (Udemy)** - The networking fundamentals section is solid and goes at a good pace for someone who already works in the field.
- **NetworkChuck (YouTube)** - [OSI Model Explained](https://www.youtube.com/watch?v=3kfO61Mensg) - More casual/entertaining style. Good if you want a quick refresher rather than a deep dive.
- **Cisco's Official Exam Topics** - [CCNA 200-301 Exam Topics](https://learningnetwork.cisco.com/s/ccna-exam-topics) - Always check what Cisco says is in scope. For this part, it's section 1.1 through 1.3.

{{% /steps %}}

---

## The TCP/IP Model — 4 Layers

The TCP/IP model is the 'practical' model. It's what the internet actually uses. Unlike the OSI model's 7 layers, the TCP/IP model condenses everything into 4 layers. The CCNA expects you to map between the two.

| TCP/IP Layer | OSI Equivalent | Key Protocols |
|---|---|---|
| Application | Application + Presentation + Session (5-7) | HTTP, DNS, DHCP, SSH, FTP, SMTP |
| Transport | Transport (4) | TCP, UDP |
| Internet | Network (3) | IP, ICMP, ARP, OSPF |
| Network Access | Data Link + Physical (1-2) | Ethernet, Wi-Fi, PPP |

The important thing to understand is that the TCP/IP model is not just "the OSI model with some layers merged." It was developed independently and is based on how protocols actually work in practice. But for the exam, being able to map between the two models is what matters.

---

### Why Two Models?

The OSI model is the *reference* model, it's great for teaching, discussing, and troubleshooting in a structured way. The TCP/IP model is the 'implementation' model, it describes what's actually running on your network. The CCNA tests both because:

1. You need OSI for precise troubleshooting communication ("this is a Layer 2 issue")
2. You need TCP/IP for understanding how real protocols interact

---

## Encapsulation & De-encapsulation

This is the process of data moving down the OSI stack (encapsulation) and back up the stack (de-encapsulation). Each layer adds its own header (and sometimes trailer) as data moves down.

Here's the full flow, top to bottom:

```
Layer 7-5:  Application creates DATA
Layer 4:    Transport adds TCP/UDP header    →  SEGMENT (TCP) or DATAGRAM (UDP)
Layer 3:    Network adds IP header           →  PACKET
Layer 2:    Data Link adds MAC header+trailer →  FRAME
Layer 1:    Physical converts to BITS         →  BIT STREAM
```

- **L5-7:** The application generates raw data. This could be an HTTP request, an email, a file, anything.
- **L4:** The Transport layer breaks the data into segments and adds a header containing source/destination port numbers, sequence numbers (TCP), and other control info.
- **L3:** The Network layer adds an IP header with source and destination IP addresses. Now it's a packet.
- **L2:** The Data Link layer wraps the packet in a frame by adding a MAC header (source and destination MAC addresses) and a trailer (FCS - Frame Check Sequence for error detection).
- **L1:** The Physical layer converts the frame into electrical signals, light pulses, or radio waves for transmission.

When the data arrives at the destination, the process reverses. Each layer strips its header, reads the relevant information, and passes the remaining data up to the next layer. This is de-encapsulation.

{{< callout type="info" >}}
**Study Tip:** Practice the "bottom-up troubleshooting" method and map it to encapsulation layers. When something isn't working, start at Layer 1 (is the cable plugged in? is the link light on?), then Layer 2 (is the MAC address in the CAM table? is the port in the right VLAN?), then Layer 3 (is the IP correct? can I ping the gateway?), then Layer 4 (is the port open? is a firewall blocking TCP?). This isn't just exam theory it's the actual troubleshooting methodology the CCNA expects you to describe, and it maps directly to the encapsulation layers you just learned.
{{< /callout >}}

---

## TCP vs UDP

This is one of the most heavily tested areas in the Network Fundamentals domain. You need to know the differences, the use cases, and the TCP three-way handshake.

{{% steps %}}

### TCP (Transmission Control Protocol)

TCP is*connection-oriented. Before any data is sent, the two hosts establish a connection using the three-way handshake:

```
1. SYN       →  Client sends SYN to server (sequence number = x)
2. SYN-ACK   ←  Server responds with SYN-ACK (seq = y, ack = x+1)
3. ACK       →  Client sends ACK (ack = y+1)
```

- **Line 1:** The client initiates the connection by sending a TCP segment with the SYN flag set. It includes an initial sequence number (ISN), let's call it `x`.
- **Line 2:** The server acknowledges the client's SYN by responding with both SYN and ACK flags set. It sends its own ISN (`y`) and acknowledges the client's sequence number by setting ack to `x+1`.
- **Line 3:** The client completes the handshake by sending an ACK back, acknowledging the server's sequence number (`y+1`). The connection is now established, and data transfer begins.

Key TCP features:
- **Reliability** - sequence numbers and acknowledgments ensure all data arrives in order
- **Flow control** - windowing adjusts how much data is sent before requiring an ACK
- **Error recovery** - lost segments are retransmitted
- **Ordered delivery** - segments are reassembled in the correct order at the destination

---

### UDP (User Datagram Protocol)

UDP is connectionless. No handshake. No acknowledgments. No sequencing. It just sends and hopes for the best.

Why use it? Speed. For applications where a retransmission would be worse than a lost packet (real-time voice, video, DNS lookups), UDP is the right choice. By the time TCP retransmits a lost voice packet, the conversation has already moved on.

### TCP vs UDP Comparison

| Feature | TCP | UDP |
|---|---|---|
| Connection type | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Ordered via sequence numbers | No ordering |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Header size | 20 bytes (minimum) | 8 bytes |
| Flow control | Yes (windowing) | No |
| Use cases | HTTP, HTTPS, FTP, SSH, SMTP | DNS, DHCP, TFTP, VoIP, video streaming |

{{< callout type="info" >}}
**Study Tip - Remembering TCP vs UDP:** The way I internalized this was by thinking about what each protocol *cares about*:

- **TCP = "I care about every single byte."** It counts packets (sequence numbers), demands receipts (ACKs), and resends anything that gets lost. Think of it like sending a legal document via certified mail (you need proof it arrived).
- **UDP = "Send it and forget it."** No tracking, no receipts, no retransmission. Think of it like shouting across a room (fast, but if they didn't hear you, too bad).

For the protocols that use each: ask yourself, *"Would it be a disaster if some data was lost?"* If yes → TCP. File transfers, web pages, and email can't afford missing data. If no → UDP. A dropped voice packet in a phone call is a tiny blip; retransmitting it 200ms later would be worse than skipping it.
{{< /callout >}}

{{< callout type="info" >}}
**💡 Tip:** DNS is a tricky one since it uses **UDP port 53** for standard queries (which are small and need to be fast) but falls back to **TCP port 53** for zone transfers and responses larger than 512 bytes. The exam knows this and may test it.
{{< /callout >}}

---

### TCP Connection Termination

Connections are torn down with a four-step process (sometimes called the four-way teardown):

```
1. FIN     →  Initiator sends FIN
2. ACK     ←  Receiver acknowledges the FIN
3. FIN     ←  Receiver sends its own FIN
4. ACK     →  Initiator acknowledges the receiver's FIN
```

- **Line 1:** One side decides to close the connection and sends a FIN (finish) flag.
- **Line 2:** The other side acknowledges receipt of the FIN.
- **Line 3:** When the other side is also ready to close, it sends its own FIN.
- **Line 4:** The initiator acknowledges, and the connection is fully closed.

This is less commonly tested than the three-way handshake, but it does show up occasionally.

{{% /steps %}}

---

## Common Protocols & Port Numbers

The CCNA expects you to know the well-known port numbers cold. These just need to be memorized. Here's the list I drilled:

| Protocol | Port(s) | Transport | Layer 7 Function |
|---|---|---|---|
| FTP Data | 20 | TCP | File transfer (data channel) |
| FTP Control | 21 | TCP | File transfer (control/commands) |
| SSH | 22 | TCP | Secure remote access |
| Telnet | 23 | TCP | Unsecure remote access |
| SMTP | 25 | TCP | Sending email |
| DNS | 53 | TCP/UDP | Name resolution |
| DHCP Server | 67 | UDP | IP address assignment (server listens) |
| DHCP Client | 68 | UDP | IP address assignment (client listens) |
| TFTP | 69 | UDP | Simple file transfer (no auth) |
| HTTP | 80 | TCP | Web traffic (unencrypted) |
| POP3 | 110 | TCP | Retrieving email |
| NTP | 123 | UDP | Time synchronization |
| SNMP | 161/162 | UDP | Network monitoring (161 queries, 162 traps) |
| HTTPS | 443 | TCP | Web traffic (encrypted) |
| Syslog | 514 | UDP | Log messaging |

{{< callout type="info" >}}
**Study Tip — Memorizing Port Numbers:** Pure memorization is painful, so I used **association stories** to make the key ones stick:

- **FTP (20/21):** "FTP is old enough to drink because it's **21**." (And 20 is its buddy carrying the actual data.)
- **SSH (22) & Telnet (23):** "SSH came first (**22**) because it's secure. Telnet is one step behind (**23**) because it's outdated and insecure."
- **DNS (53):** "DNS is the **high-five-three** of networking because everyone depends on it."
- **DHCP (67/68):** "DHCP is a **pair**, server gives (**67**), client takes (**68**)." I remembered 67 as the server because it's the lower/first number and the server has to be ready first.

These are dumb associations, but they worked for me. The best mnemonic is the one you don't have to think about.
{{< /callout >}}

---

### Port Ranges

For completeness, the CCNA also expects you to know the port number ranges:

- **Well-Known Ports:** 0 - 1023 (assigned to common protocols by IANA)
- **Registered Ports:** 1024 - 49151 (assigned to applications by request)
- **Dynamic/Ephemeral Ports:** 49152 - 65535 (used temporarily by client-side connections)

When your browser connects to a web server on port 443, *your* side of the connection uses a random ephemeral port (like 52847). The server responds back to that ephemeral port. This is how multiple browser tabs can connect to the same server simultaneously since each uses a different source port.

{{< callout type="info" >}}
**Study Tip:** I used [Anki](https://apps.ankiweb.net/) (free flashcard app with spaced repetition) to drill port numbers. Spaced repetition is scientifically proven to be more effective than re-reading. There are pre-made CCNA Anki decks out there.
{{< /callout >}}

---

## A Packet's Journey

To tie this all together, I walked through what actually happens when I open a browser and go to `https://example.com`. This exercise helped me connect the theory to something concrete.

```
Step 1:  Browser (Application layer) initiates HTTPS request
Step 2:  DNS resolves example.com → 93.184.216.34 (UDP port 53)
Step 3:  TCP three-way handshake to 93.184.216.34 on port 443
Step 4:  TLS handshake establishes encrypted session
Step 5:  HTTP GET request is created (Application data)
Step 6:  Transport layer segments the data, adds TCP header (src port: 52847, dst port: 443)
Step 7:  Network layer adds IP header (src: 192.168.1.50, dst: 93.184.216.34)
Step 8:  Data Link layer adds Ethernet header (src MAC: my NIC, dst MAC: default gateway)
Step 9:  Physical layer transmits bits over the wire
Step 10: At the gateway (router), L2 header is stripped, new L2 header added for next hop
Step 11: Process repeats hop by hop until the packet reaches the destination
Step 12: Destination de-encapsulates: bits → frame → packet → segment → data
```

- **Step 1-4:** All happening at Layers 5-7. The application starts the process, DNS resolves the name, and a TCP connection is established.
- **Step 5-6:** The actual HTTP request gets created and handed to Layer 4, which adds port numbers and sequence tracking.
- **Step 7:** Layer 3 adds IP addresses. Notice the destination is the server's IP, not the gateway's IP.
- **Step 8:** Layer 2 adds MAC addresses. The destination MAC is the **default gateway's MAC**, not the server's MAC. The frame only needs to get to the next hop.
- **Step 9:** Bits on the wire.
- **Step 10:** At each router hop, the Layer 2 header is stripped and replaced with new source/destination MACs for the next segment. The Layer 3 header (IP addresses) stays the same throughout the journey.
- **Step 11-12:** The process continues until the server receives the data and de-encapsulates it.

{{< callout type="error" >}}
**Danger:** A very common exam mistake: confusing *which address changes at each hop*. The **IP addresses (Layer 3) stay the same** from source to destination (assuming no NAT). The **MAC addresses (Layer 2) change at every hop** since each router strips the old frame and builds a new one with the next-hop MAC. If you get a question about "what is the destination MAC when a packet leaves Router A heading to Router B," the answer is **Router B's MAC**, not the final destination's MAC. This is one of the most frequently missed concepts.
{{< /callout >}}

{{< callout type="info" >}}
**Study Tip - Memorizing "What Changes Per Hop":** I used the **postal mail analogy** to make this stick permanently:

- **IP address = the address on the letter** - it stays the same the entire journey. "From: Alice in Dallas, To: Bob in Seattle." Every post office (router) reads this to decide where to send it next, but no one changes it.
- **MAC address = the delivery truck** - it changes at every stop. The letter goes from Alice's mailbox to the local post office (truck 1), then to a regional hub (truck 2), then to Seattle's post office (truck 3), then to Bob's street (truck 4). Different trucks, same letter.

{{< /callout >}}

---

## Verifying with Wireshark — A Quick Example

One of the best ways I reinforced these concepts was by capturing real traffic in Wireshark. Here's what a captured HTTP packet looks like (simplified):

```
Frame 1: 342 bytes on wire
  Ethernet II, Src: aa:bb:cc:11:22:33, Dst: dd:ee:ff:44:55:66
    Type: IPv4 (0x0800)
  Internet Protocol Version 4, Src: 192.168.1.50, Dst: 93.184.216.34
    Protocol: TCP (6)
    Time to Live: 64
  Transmission Control Protocol, Src Port: 52847, Dst Port: 443
    Flags: 0x018 (PSH, ACK)
    Seq: 1, Ack: 1
  [TLS/HTTP Application Data]
```

- **Line 1:** Total frame size - this is the complete Layer 2 frame including all headers and payload.
- **Lines 2-3:** The **Ethernet II header** (Layer 2) - source and destination MAC addresses, plus an EtherType field (0x0800 = IPv4) that tells the receiving device what Layer 3 protocol is inside.
- **Lines 4-6:** The **IP header** (Layer 3) - source and destination IP addresses, the protocol field (6 = TCP), and TTL (how many router hops before the packet is discarded).
- **Lines 7-9:** The **TCP header** (Layer 4) - source and destination ports, TCP flags (PSH = push data to application, ACK = acknowledging received data), and sequence/acknowledgment numbers.
- **Line 10:** The actual application payload - encrypted in this case because it's HTTPS/TLS.

---

## Key Exam Traps I Flagged During Study

After going through the OCG chapters and some practice questions on this topic, here are the specific traps I noticed the exam sets:

1. **"Which layer is responsible for logical addressing?"** - The answer is Layer 3 (Network), not Layer 2. Layer 2 handles *physical* addressing (MAC). The word "logical" = IP addresses = Layer 3.

2. **"Which layer provides end-to-end communication?"** - Layer 4 (Transport). Not Layer 3. Layer 3 handles routing between networks, but *end-to-end* reliability and session multiplexing is Layer 4.

3. **"Which PDU is used at Layer 2?"** - Frame. Not packet. The exam loves testing PDU names.

4. **"What protocol provides reliable delivery?"** - TCP. Not IP. IP is *unreliable* by design (best-effort). Reliability comes from TCP at Layer 4.

5. **"What port does DHCP use?"** - Trick: it uses **two** ports. 67 (server) and 68 (client). If the question asks for "the DHCP server port," the answer is 67.

{{< callout type="info" >}}
**Study Tip — Memorizing Exam Keywords:** The exam uses specific trigger words that map to specific layers. I made a cheat sheet of these keyword-to-layer associations and drilled them until they were automatic:

| If the question says... | It's asking about... |
|---|---|
| "logical addressing" | Layer 3 (IP) |
| "physical addressing" | Layer 2 (MAC) |
| "end-to-end" or "reliable delivery" | Layer 4 (TCP) |
| "path selection" or "routing" | Layer 3 |
| "segmentation" or "flow control" | Layer 4 |
| "encryption" or "data formatting" | Layer 6 |
| "establishing a session/dialog" | Layer 5 |
| "error detection" (not correction) | Layer 2 (FCS in frame trailer) |
| "bit transmission" or "signaling" | Layer 1 |

{{< /callout >}}

---

## Summary

This part was mostly review for me since I work in networking, but the precision the exam demands caught me off guard in a few spots. The PDU names, the exact layer classifications (especially for DNS and ARP), and the "destination MAC changes per hop but destination IP stays the same" concept.

**My study approach for Part 1 was:**
1. Read OCG Volume 1, Chapters 1-3 (took about 2 hours with notes)
2. Watched Jeremy's IT Lab Days 3-4 to hear a second explanation
3. Opened Wireshark and captured real traffic to see encapsulation in action
4. Made Anki cards for port numbers and layer classifications
5. Took 10-15 practice questions to find my gaps
6. Wrote this blog post to teach it back

That last point is real. Writing this post forced me to verify things I thought I knew. If you're following along with your own study journey, I'd genuinely recommend the "learn by teaching" approach. Start a blog, explain it to a coworker, or just talk through it out loud. You'll find your blind spots fast.
