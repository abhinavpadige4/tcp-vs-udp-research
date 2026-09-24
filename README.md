# TCP vs UDP — A Self-Contained Comparison

**TCP** (Transmission Control Protocol) and **UDP** (User Datagram Protocol) are the two primary **transport-layer** protocols in the Internet protocol suite. Both sit above IP (layer 3) and below the application layer, and both use a 16-bit port number to multiplex traffic to/from applications. They differ fundamentally in *what guarantees they make* about delivering data — and that single design choice drives almost every other difference between them.

- **TCP** is a **connection-oriented, reliable, ordered, byte-stream** protocol. It guarantees that every byte you send arrives, exactly once, in order, or it tells you the connection failed.
- **UDP** is a **connectionless, best-effort, unordered, datagram** protocol. It hands your packet to IP and moves on. If the packet is lost, duplicated, or reordered, UDP does not tell you.

Everything else — header size, latency, flow control, congestion control, error handling — flows from that trade-off.

---

## Side-by-Side Comparison

| Characteristic | TCP | UDP |
|---|---|---|
| **Connection type** | Connection-oriented. Requires a **three-way handshake** (SYN → SYN-ACK → ACK) before data flows, and a four-way teardown (FIN/ACK) at the end. | Connectionless. No handshake, no teardown. Every datagram is independent. |
| **Reliability** | Reliable. Every segment is **acknowledged**; unacknowledged segments are **retransmitted** (with exponential back-off). | Unreliable / best-effort. No ACKs, no retransmissions. Lost packets are simply lost. |
| **Ordering** | Guaranteed **in-order delivery**. Out-of-order segments are buffered and reassembled by sequence number. | **No ordering guarantee**. Packets may arrive out of order or be duplicated. |
| **Data model** | Continuous **byte stream** (no message boundaries). | **Datagrams** — each send is a discrete message with preserved boundaries. |
| **Flow control** | Yes — **sliding window** with advertised receiver window (rwnd) prevents a fast sender from overwhelming a slow receiver. | None. Sender is responsible for pacing. |
| **Congestion control** | Yes — **slow start, congestion avoidance, fast retransmit, fast recovery** (Reno/Cubic/BBR, etc.) adapt to network load. | None at the transport layer. Congestion is the application's problem. |
| **Error handling** | **End-to-end checksum** (16-bit) on header + data; corrupted segments are dropped and retransmitted. | **Optional 16-bit checksum** (may be zero = disabled, especially in IPv6). Corrupted datagrams are silently dropped. |
| **Header size** | **20 bytes minimum**, up to **60 bytes** with options (e.g., timestamps, SACK). | **8 bytes fixed** (src port, dst port, length, checksum). |
| **Typical latency** | Higher — handshake adds ~1 RTT; retransmissions add more. | Lower — no handshake, no retransmission stalls. |
| **Multiplexing** | 4-tuple: (src IP, src port, dst IP, dst port). | Same 4-tuple, but each datagram is independent. |
| **Broadcast / multicast** | Not supported (TCP is point-to-point). | Natively supported — one datagram can be sent to a broadcast or multicast group. |
| **Typical overhead** | Higher (headers + ACKs + retransmits). | Lower (small header, no ACKs). |
| **Best for** | Correctness-critical, latency-tolerant traffic. | Latency-sensitive, loss-tolerant, or many-to-many traffic. |

*Sources: [RFC 793 — TCP](https://datatracker.ietf.org/doc/html/rfc793), [RFC 768 — UDP](https://datatracker.ietf.org/doc/html/rfc768), [Wikipedia: TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol), [Wikipedia: UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol).*

---

## Typical TCP Use Cases

TCP is chosen whenever **data integrity matters more than raw speed**:

- **HTTP / HTTPS (web browsing)** — every web page, API call, and file download. Losing a byte of HTML or JSON would corrupt the response.
- **FTP / SFTP** — file transfers where a single dropped byte makes the file unusable.
- **SMTP, IMAP, POP3** — email protocols; a corrupted message is unacceptable.
- **SSH / Telnet** — interactive remote shells; every keystroke and every byte of output must arrive intact.
- **Database connections** (MySQL, PostgreSQL, MongoDB wire protocols) — queries and results must be exact.
- **SCP / rsync** — reliable file synchronization.
- **VPN tunnels** (OpenVPN in TCP mode, WireGuard-over-TCP) — encapsulated traffic must not be corrupted.
- **Print jobs (LPD), Gopher, IRC** — legacy text protocols that assume a reliable stream.

**Why TCP?** All of these applications would break or produce garbage if bytes were lost, reordered, or duplicated. The handshake and retransmission overhead is a small price for correctness.

---

## Typical UDP Use Cases

UDP is chosen whenever **freshness and low latency matter more than perfect delivery**:

- **DNS** — small queries where a lost packet is cheaper to retry at the application layer than to wait for TCP's handshake.
- **DHCP** — clients don't yet have an IP address, so they can't use TCP.
- **VoIP (SIP + RTP)** — a dropped audio frame is less noticeable than a 200 ms delay; retransmitting old audio is useless.
- **Video streaming** (YouTube, Netflix, Twitch, Zoom, WebRTC) — modern codecs tolerate small losses and hide them with forward error correction.
- **Online multiplayer games** — a stale position update is worse than a missing one; the game resyncs on the next tick.
- **TFTP** — a deliberately simple, connectionless file transfer for bootstrapping (PXE, Cisco routers).
- **SNMP** — network management polling; a missed poll is retried.
- **Multicast / broadcast** (mDNS, SSDP, NTP, IGMP) — one-to-many delivery that TCP cannot do.
- **NTP** — time synchronization; the protocol itself handles jitter and loss.
- **QUIC / HTTP/3** — see the modern-context section below.

**Why UDP?** In all of these, either (a) the payload is small enough that a retry is cheap, (b) old data is worthless once it's late, or (c) the protocol needs multicast/broadcast, which TCP fundamentally cannot provide.

---

## Modern Context: The Divide Is Blurring

The clean "TCP for reliability, UDP for speed" story is being rewritten by **QUIC** ([RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000)), a transport protocol that runs **on top of UDP** but provides TCP-like reliability, ordering, and congestion control — plus features TCP cannot easily add because it is already deployed in every network device and middlebox on the Internet. QUIC performs its handshake in **0-RTT or 1-RTT** (versus TCP's 1-RTT + TLS's additional 1-RTT), encrypts the transport headers themselves, and eliminates **head-of-line blocking** across independent streams, so a single lost packet no longer stalls unrelated data.

**HTTP/3** ([RFC 9114](https://datatracker.ietf.org/doc/html/rfc9114)) is the first major application protocol built on QUIC, and it is now the default on a large fraction of the web — Cloudflare reports that HTTP/3 is enabled for the majority of its customers and delivers measurably lower page-load times, especially on lossy mobile networks ([Cloudflare blog](https://blog.cloudflare.com/http3-the-past-present-and-future/)). Because QUIC lives in user space over UDP, new congestion controllers, encryption schemes, and multiplexing models can ship without waiting for kernel or router upgrades.

The practical takeaway is that the choice is no longer "TCP or UDP" but **"what transport semantics do I need, and where should they live?"** — TCP still dominates for legacy and server-to-server traffic, UDP is the substrate for real-time media and multicast, and QUIC-over-UDP is increasingly the default for new client-facing protocols that want both reliability and low latency.

---

## References

1. **RFC 793 — Transmission Control Protocol** (the original TCP specification, 1981). <https://datatracker.ietf.org/doc/html/rfc793>
2. **RFC 768 — User Datagram Protocol** (the original UDP specification, 1980). <https://datatracker.ietf.org/doc/html/rfc768>
3. **RFC 9000 — QUIC: A User-Datagram Transport Protocol for HTTP/3** (2021). <https://datatracker.ietf.org/doc/html/rfc9000>
4. **RFC 9114 — HTTP/3** (2022). <https://datatracker.ietf.org/doc/html/rfc9114>
5. **Wikipedia — Transmission Control Protocol**. <https://en.wikipedia.org/wiki/Transmission_Control_Protocol>
6. **Wikipedia — User Datagram Protocol**. <https://en.wikipedia.org/wiki/User_Datagram_Protocol>
7. **Cloudflare — HTTP/3: The Past, Present, and Future**. <https://blog.cloudflare.com/http3-the-past-present-and-future/>
8. **Kurose & Ross, *Computer Networking: A Top-Down Approach*** — Chapters 3 (Network Layer) and 4 (Transport Layer) for a textbook treatment of TCP and UDP.
9. **Tanenbaum & Wetherall, *Computer Networks*, 5th ed.** — Chapter 6 (Transport Layer Protocols).

---

*This document is self-contained: a reader with no prior networking background can understand the TCP/UDP differences from the comparison table, use-case lists, and modern-context paragraph above.*
