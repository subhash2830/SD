# 📘 ARP — Address Resolution Protocol (Design Notes + Memory Map)

## 🔹 Why ARP Exists (The Problem)
**Problem:** IP addresses are logical (Layer 3) — Ethernet only understands MAC addresses (Layer 2). A
frame can't be delivered on the wire without a destination MAC, so something has to translate "I want to
reach this IP" into "this is the physical address to send the frame to."

**Design Analogy:** IP address = a person's name. MAC address = their desk number in the office. ARP is
the office assistant who shouts "who sits under this name?" and writes the answer down for next time.

**Design Principle:** ARP is the glue between Layer 3 routing logic and Layer 2 forwarding reality —
without it, IP and Ethernet simply cannot talk to each other.

---

## 🔹 How ARP Works
1. **Cache check** — device checks its local ARP cache first, to avoid unnecessary broadcasting.
2. **Broadcast request** — if not cached, it broadcasts "Who has IP X? Tell me." to the whole local
   segment.
3. **Unicast reply** — only the device that owns that IP replies, directly, with its MAC address.
4. **Cache update** — the mapping is stored locally for reuse, with a timeout (typically a few minutes).

👉 **Design Insight:** ARP is stateless and unauthenticated — any device can claim to own any IP, and
nothing checks. This simplicity is exactly what makes it fast and lightweight, and exactly what makes it
insecure.

---

## 🔹 ARP in Network Design
- **Local scope only** — ARP never crosses a subnet boundary. For a remote destination, a host only ever
  ARPs for its **default gateway's** MAC — the gateway (router) handles everything past that.
- **Scalability concern** — in a large flat Layer 2 domain, ARP broadcasts multiply with every host,
  eating bandwidth and CPU on every device that has to process them. This is one of the concrete reasons
  designers break networks into smaller VLANs/subnets rather than one giant flat broadcast domain.
- **Evolution** — IPv6 replaces ARP entirely with **NDP (Neighbor Discovery Protocol)**, which uses
  multicast instead of broadcast and adds optional cryptographic protection (SEND) — a direct response to
  ARP's two biggest weaknesses.

---

## 🔹 ARP Variants
| Variant | What it does | Where it's used |
|---|---|---|
| **Gratuitous ARP (GARP)** | A device announces its own IP-to-MAC mapping unprompted, without being asked | Failover (HSRP/VRRP moving the virtual MAC to a new active router), VM live migration, duplicate-IP detection |
| **Proxy ARP** | A router answers an ARP request on behalf of a host that isn't actually on that segment | Legacy flat networks, some VPN/remote-access designs where hosts think they're on the same subnet |
| **Reverse ARP (RARP)** | A diskless host asks "what's my IP, given my MAC?" | Obsolete — fully replaced by DHCP |
| **Inverse ARP (InARP)** | Maps a WAN circuit ID (like a Frame Relay DLCI) to an IP address | Legacy Frame Relay / ATM WAN links |

---

## 🔹 Real-World Applications
- **Gateway resolution** — every single outbound packet to a remote network starts with the host ARPing
  for its default gateway.
- **Failover protocols** — HSRP/VRRP use Gratuitous ARP to instantly tell switches "the virtual gateway
  MAC has moved to me" the moment a failover happens.
- **Virtualization** — when a VM migrates between hypervisor hosts, it sends a GARP so upstream switches
  update their MAC tables and start forwarding to the new physical location immediately.
- **SDN / large data centers** — controllers often intercept and suppress ARP broadcasts entirely (ARP
  suppression), answering locally from a central table instead of flooding a huge VM farm with broadcast
  traffic.
- **Security monitoring** — IDS/IPS systems watch for abnormal ARP patterns (one MAC claiming many IPs, or
  rapid mapping changes) as a spoofing indicator.

---

## 🔹 ARP Security Risks
- **ARP Spoofing / Poisoning** — an attacker sends forged ARP replies claiming to own the gateway's IP,
  redirecting traffic through itself — the classic man-in-the-middle setup on a LAN.
- **ARP Flooding** — overwhelming a switch's CAM (MAC address) table with bogus entries can force it into
  a fail-open "hub mode," flooding all traffic to all ports.
- **Defenses:**
  - **Dynamic ARP Inspection (DAI)** — validates ARP packets against the DHCP snooping binding table;
    anything that doesn't match a known IP-MAC-port binding is dropped.
  - **Static ARP entries** — for a handful of truly critical devices (like the default gateway itself).
  - **Encrypted transport (HTTPS/SSH/TLS)** — doesn't stop the spoofing, but limits what an attacker who
    successfully MITMs the traffic can actually read or modify.

👉 **Design Takeaway:** ARP is insecure by design — it was built for a trusted 1980s LAN, not a modern
multi-tenant network. Real protection comes from DHCP Snooping + DAI + segmentation, not from ARP itself.

---

## 🔹 CLI Verification (Interview-Ready)
```
arp -a                              # View ARP cache (Windows/macOS/most CLIs)
ip neigh show                       # Linux ARP/neighbor table, with state (REACHABLE, STALE, etc.)
tcpdump -i eth0 arp                 # Watch live ARP traffic
netsh interface ip delete arpcache  # Flush ARP cache (Windows)
```

---

## 🔹 Key Design Takeaways
- ARP = the translator between IP (logical) and MAC (physical).
- Works only within the local subnet — remote traffic only ever needs the gateway's MAC.
- Simple, fast, but unauthenticated — insecure by design.
- Scalability is managed via segmentation (VLANs) or suppression (SDN, NDP in IPv6).
- **Interview angle:** always frame ARP as *why it exists* (bridging IP↔MAC) and *how it impacts design*
  (broadcast domain size, failover speed, security exposure).

---

## 🎤 Interview-Ready Answer
"ARP bridges Layer 3 logical addressing and Layer 2 physical forwarding by translating an IP address into
a MAC address — a device broadcasts 'who has this IP,' and the owner replies directly with its MAC, which
gets cached for reuse. It only ever works within a local subnet, so remote traffic just ARPs for the
default gateway. Because it's stateless and completely unauthenticated, it's a classic MITM vector via ARP
spoofing — mitigated in real networks with Dynamic ARP Inspection built on top of DHCP Snooping, plus
segmentation to limit broadcast domain size. IPv6 replaces it entirely with NDP, which adds multicast and
optional cryptographic protection."

## 🧠 Memory Map (for recall)
```
ARP → Translator (IP ↔ MAC)
   │
   ├─ Scope → Local subnet only (remote = ARP for the gateway, not the destination)
   ├─ Process → Cache check → Broadcast request → Unicast reply → Cache update
   ├─ Variants → Gratuitous (failover/VM), Proxy (legacy), Reverse (obsolete), InARP (WAN)
   ├─ Design issues → Broadcast storms at scale, zero authentication
   ├─ Security → Spoofing/Poisoning, Flooding → mitigated by DAI + DHCP Snooping + static entries
   └─ Evolution → IPv6 replaces ARP with NDP (multicast + optional crypto via SEND)
```
