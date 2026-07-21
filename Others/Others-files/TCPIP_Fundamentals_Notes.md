# TCP/IP Fundamentals — Notes
### Reviewed, corrected, and simplified

---

## 1. The TCP/IP Model

TCP/IP has **4 layers**. It's older and simpler than the 7-layer OSI model, but every layer in TCP/IP
just maps to one or more OSI layers. This mapping is worth knowing cold, because real job interviews
almost always ask for it.

| TCP/IP Layer | Maps to OSI Layers | What it does | Example protocols |
|---|---|---|---|
| 4. Application | 5, 6, 7 (Session, Presentation, Application) | Where user-facing services live | HTTP(S), DNS, SMTP, IMAP, POP3, FTP, SSH |
| 3. Transport | 4 (Transport) | End-to-end delivery between two hosts | TCP, UDP, SCTP |
| 2. Internet | 3 (Network) | Addressing and routing between networks | IP, ICMP, IGMP |
| 1. Link (Network Access) | 1, 2 (Physical, Data Link) | Getting bits onto the wire/air, and local delivery | Ethernet, Wi-Fi, ARP, PPP |

**Simple way to remember it:** Application = "what you're doing," Transport = "how reliably it gets
there," Internet = "where it's going," Link = "how it physically moves."

Note: ARP technically sits between Layer 1 and Layer 2 — it resolves an IP address (Layer 2/Internet) to
a MAC address (Layer 1/Link), so different books place it slightly differently. Don't overthink it.

---

## 2. Transport Layer — TCP vs UDP

### TCP (Transmission Control Protocol)
Reliable, ordered, connection-oriented. Used when correctness matters more than speed (web pages, email,
file transfer, database replication).

- **Three-way handshake** (connection setup):
  1. Client → Server: **SYN**
  2. Server → Client: **SYN-ACK**
  3. Client → Server: **ACK**
- **Sequence numbers**: every byte is numbered, so the receiver can reorder out-of-order packets and
  detect missing ones.
- **Flow control**: the receiver advertises a **window size** — "this is how much data I can accept right
  now" — so a fast sender doesn't overwhelm a slow receiver.
- **Congestion control**: separate from flow control — this protects the *network*, not just the
  receiver.
  - **Slow Start**: begin sending slowly, double the rate each round-trip until loss is seen.
  - **Congestion Avoidance**: once loss happens, back off and grow more cautiously.
  - **Fast Retransmit / Fast Recovery**: resend a lost packet as soon as it's detected (via duplicate
    ACKs), without waiting for a full timeout.
  - Modern stacks mostly use newer algorithms like **CUBIC** (Linux default) or **BBR** (Google, used in
    many cloud/CDN networks) instead of the classic 1988 Reno-style algorithm — worth knowing the names
    even at a high level.
- **Segmentation**: data larger than the **MSS (Maximum Segment Size)** is split into multiple segments,
  each with its own sequence number for reassembly.

### UDP (User Datagram Protocol)
Connectionless, no handshake, no guaranteed delivery, no ordering, no built-in congestion control.
Trades reliability for speed and low overhead.

- **Use cases**: VoIP, video streaming, online gaming, DNS queries, and most telemetry/monitoring traffic
  — anywhere a late packet is worse than a lost one.

---

## 3. Internet Layer — IP, Subnetting, NAT

### IPv4 vs IPv6
| | IPv4 | IPv6 |
|---|---|---|
| Address size | 32-bit | 128-bit |
| Address count | ~4.3 billion | Effectively unlimited (2^128) |
| Example | 192.168.1.1 | 2001:0db8:85a3::8a2e:0370:7334 |
| Needs NAT? | Usually yes (address scarcity) | No — enough addresses that every device can have a real one |
| Broadcast | Yes | No — uses **multicast** instead |
| Address resolution | ARP | **NDP** (Neighbor Discovery Protocol) |

### Subnetting
Splitting a network into smaller pieces using a **subnet mask** or **CIDR notation** (e.g. `/24`). This
reduces broadcast domain size, improves routing efficiency, and lets you allocate address space based on
how many hosts each site/segment actually needs, instead of wasting whole blocks.

### NAT (Network Address Translation)
Maps many private IPv4 addresses to one (or a few) public IPv4 addresses. It exists mainly to conserve
scarce IPv4 address space — it is **not really a security feature**, even though it has a side effect of
hiding internal addressing. IPv6 generally does not need NAT because address space isn't scarce (though
NAT66/NPTv6 exists for specific renumbering-avoidance use cases, not for address conservation).

---

## 4. Routing Protocols

Routing protocols fall into two categories:

- **IGP (Interior Gateway Protocol)** — routes *within* one organization/network:
  - **RIP**: distance-vector, hop-count based, simple but doesn't scale — rarely used today.
  - **OSPF**: link-state, fast convergence, widely used in enterprise and provider networks.
  - **EIGRP**: Cisco's advanced distance-vector protocol, fast convergence, common in Cisco-heavy
    enterprise networks.
  - **IS-IS**: link-state, similar to OSPF, the dominant choice in large service-provider backbones.
- **EGP (Exterior Gateway Protocol)** — routes *between* organizations:
  - **BGP**: path-vector protocol, the only routing protocol used between different Autonomous Systems
    (ASes) — this is literally what holds the global internet together.

**Static vs dynamic**: static routes are manually configured (simple, predictable, but doesn't react to
failures); dynamic routing protocols automatically discover paths and reroute around failures.

---

## 5. ARP and ICMP

- **ARP (Address Resolution Protocol)** — IPv4 only. Resolves an IP address to a MAC address by
  broadcasting "who has this IP?" on the local network; the owning device replies with its MAC address.
  IPv6 does not use ARP — it uses **NDP** instead, which runs over ICMPv6.
- **ICMP (Internet Control Message Protocol)** — used for diagnostics and error reporting, not for
  carrying user data. Examples: **Echo Request/Reply** (ping), **Time Exceeded** (traceroute, TTL hit 0),
  **Destination Unreachable**.

---

## 6. Quality of Service (QoS)

QoS controls *which traffic gets priority* when bandwidth is limited. Two mechanisms are often confused:

- **Traffic Policing**: enforces a rate limit by **dropping or re-marking** packets that exceed it —
  traffic isn't delayed, it's just discarded/marked immediately if it's over the limit.
- **Traffic Shaping**: enforces a rate limit by **buffering and delaying** excess traffic to smooth it
  out, rather than dropping it outright.

- **DiffServ**: the standard, scalable QoS model. Traffic is tagged with a **DSCP (Differentiated
  Services Code Point)** value in the IP header, and routers along the path treat traffic differently
  based on that marking (e.g., voice gets priority queuing over bulk file transfers).

---

## 7. Network Security Basics

- **Firewalls**: can filter at Layer 3/4 (IP address, port — "stateless" or "stateful" packet filtering)
  or Layer 7 (application-aware, inspecting actual content — e.g. blocking specific URLs or app
  behaviors).
- **IPsec**: encrypts and authenticates IP packets — commonly used for site-to-site or remote-access VPNs.
- **TLS** (the modern name — "SSL" is the deprecated predecessor): encrypts application-layer traffic,
  most commonly HTTPS.

---

## 8. IPv6 Adoption

- **Dual-stack**: a network runs IPv4 and IPv6 side by side, letting devices use whichever is available —
  the standard, safest way to migrate.
- Other transition tools exist too (tunneling methods like 6to4/NAT64, and translation), but dual-stack
  is the most common real-world approach today.

---

## 9. Load Balancing and High Availability

- **Round Robin**: sends each new request to the next server in line, in order.
- **Least Connections**: sends each new request to whichever server currently has the fewest active
  connections — usually a better real-world choice than round robin, since it accounts for uneven load.
- Common software load balancers: **HAProxy**, **NGINX**. Common hardware/cloud options: F5, AWS ALB/NLB.
- **High availability** in general means removing single points of failure — redundant links, redundant
  devices, and fast-failover routing/protection mechanisms.

---

## 10. SDN and NFV

- **SDN (Software-Defined Networking)**: separates the control plane (decision-making) from the data
  plane (forwarding), letting a centralized controller program the network via software instead of
  configuring each device by hand.
- **NFV (Network Functions Virtualization)**: takes functions that used to require dedicated hardware
  appliances (firewall, load balancer, router) and runs them as software/VMs on general-purpose servers —
  cheaper, more flexible, easier to scale up or down.

---

## 11. Troubleshooting Toolkit

| Tool | What it tells you |
|---|---|
| `ping` | Basic reachability and round-trip latency |
| `traceroute` / `tracert` | The hop-by-hop path a packet takes, and where it's being delayed or dropped (works by sending packets with increasing TTL) |
| Wireshark | Full packet capture and analysis — the deepest level of visibility |
| `netstat` / `ss` | What connections and ports are currently open on a host |
| MTR | Combines ping + traceroute into one continuously-updating view — often more useful than either alone |

---

## 12. Designing a Scalable Network — Key Considerations

- **Redundancy**: multiple physical paths and multiple routing peers (e.g., dual BGP uplinks to different
  providers) so no single link or device failure takes the network down.
- **Subnetting/addressing plan**: size subnets to actual need, keep broadcast domains small, plan address
  space growth in advance — this is much harder to fix later than to get right upfront.
- **VLANs**: segment traffic for security and to control broadcast domain size, independent of physical
  cabling.
- **Load balancers**: spread traffic across multiple servers/paths so no single one becomes a bottleneck
  or single point of failure.
- **Convergence time**: choose routing protocols and failover mechanisms based on how fast the business
  actually needs the network to recover from a failure — this is a real design trade-off, not just a
  technical checkbox.

---

## 🎤 Interview-Ready Answer
"TCP/IP is the 4-layer model the entire internet runs on — Application, Transport, Internet, and Link —
which maps cleanly onto OSI's 7 layers. TCP gives you reliable, ordered delivery via the three-way
handshake, sequence numbers, flow control, and congestion control; UDP trades all of that away for speed.
IP handles addressing and routing, NAT conserves scarce IPv4 space, and routing protocols split into IGPs
(within an AS — OSPF, EIGRP, IS-IS) and EGPs (between ASes — BGP). Everything else — QoS, security, SDN,
load balancing — is built on top of this foundation, not separate from it."

## Summary

TCP/IP is the foundation everything else in networking sits on top of. The core things worth having cold:
the 4-layer model and its OSI mapping, how TCP's three-way handshake and congestion control actually work,
the real difference between policing and shaping, and the difference between an IGP and an EGP. Everything
else (QoS, security, SDN, load balancing) is really just built on top of these fundamentals.
