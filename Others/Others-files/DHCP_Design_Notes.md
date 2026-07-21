# 📘 DHCP — Dynamic Host Configuration Protocol (Design Notes + Memory Map)

## 🔹 Why DHCP Exists (The Problem)
**Problem:** Every device on an IP network needs an IP address, subnet mask, default gateway, and DNS
server before it can talk to anyone. Manually typing this into every laptop, phone, and IoT device
doesn't scale, and humans will eventually type the same IP twice on two devices.

**Design Analogy:** DHCP is like a hotel front desk. You don't own your room number — you're handed one
when you check in, and you give it back when you check out, so the next guest can reuse it.

**Design Principle:** DHCP exists to make IP addressing automatic, conflict-free, and centrally managed,
instead of manual, error-prone, and per-device.

---

## 🔹 How DHCP Works — DORA
Four-message exchange between client and server:

1. **Discover** — client broadcasts "is any DHCP server out there?" (it has no IP yet, so it must
   broadcast).
2. **Offer** — a server responds with a candidate IP, lease time, and config options.
3. **Request** — client broadcasts back "I accept that offer" (broadcast again, so any *other* servers
   that also made offers know they were not chosen, and can release their offered address).
4. **Ack** — the chosen server confirms the lease. Client can now use the address.

👉 **Design Insight:** Steps 1 and 3 are broadcast on purpose — step 1 because the client has no address
yet, step 3 so competing DHCP servers back off cleanly. This is why DORA works even with multiple DHCP
servers on one segment.

---

## 🔹 Lease Lifecycle
- An IP address is **borrowed**, not owned — it's valid for a **lease time**.
- Before expiry, the client tries to **renew** (unicast request straight to the server it got the lease
  from) — this quietly happens in the background, invisible to the user.
- If the server is unreachable, the client eventually broadcasts to renew with *any* server.
- If the lease truly expires with no renewal, the address returns to the pool.
- **DHCP Release**: client can voluntarily give the address back early (e.g., on shutdown).

---

## 🔹 DHCP Message Types
| Message | Direction | Meaning |
|---|---|---|
| Discover | Client → broadcast | "Any DHCP server out there?" |
| Offer | Server → client | "Here's an IP you can have" |
| Request | Client → broadcast | "I accept this offer" (also tells other servers to stand down) |
| ACK | Server → client | "Confirmed, it's yours" |
| NAK | Server → client | "That address/lease is no longer valid" |
| Release | Client → server | "I'm done with this address" |
| Decline | Client → server | "I checked and this address is already in use — don't offer it again" |

---

## 🔹 DHCP in Network Design
- **DHCP Client**: any device requesting an address.
- **DHCP Server**: allocates addresses + options — can be a router, a dedicated appliance, or a cloud
  service.
- **DHCP Relay (`ip helper-address`)**: when the client and server are on different subnets, a router
  converts the client's local broadcast into a **unicast** relay to the real server, and inserts its own
  interface IP into the packet (the **GIADDR** field) so the server knows which subnet's pool to allocate
  from. This is *the* mechanism that lets one central DHCP server support dozens of remote subnets.

**Design Insight:** Without a relay agent, DHCP is entirely broadcast-based and cannot cross a router
boundary — this is exactly why every remote-branch VLAN needs either a local DHCP server or an
`ip helper-address` pointing at a central one.

---

## 🔹 Common DHCP Options
| Option | Meaning |
|---|---|
| 3 | Default Gateway |
| 6 | DNS Server(s) |
| 15 | Domain Name |
| 51 | Lease Time |
| 66 | TFTP Server (used for PXE network boot) |
| 150 | VoIP/IP Phone config (Cisco-specific) |

---

## 🔹 DHCP Security Risks
- **Rogue DHCP Server**: an attacker (or misconfigured device) answers DHCP requests with bad gateway/DNS
  info, silently redirecting or blackholing traffic — a very effective, low-effort man-in-the-middle setup.
- **DHCP Starvation**: attacker floods Discover messages with spoofed MACs until the entire pool is
  exhausted, so legitimate devices can't get an address at all (a DoS attack).
- **Defense — DHCP Snooping**: a switch feature that classifies ports as *trusted* (uplinks to the real
  DHCP server) or *untrusted* (access ports). Offer/ACK messages are only allowed from trusted ports —
  this kills rogue-server attacks at the switch, before they ever reach a client.
- DHCP snooping also builds a binding table (IP ↔ MAC ↔ port ↔ lease) that feeds **Dynamic ARP
  Inspection** and **IP Source Guard** — DHCP snooping is the foundation a lot of other L2 security
  features are built on top of.

---

## 🔹 Cisco Configuration Reference
```
! DHCP Server
Router(config)# ip dhcp pool LAN-POOL
Router(config-dhcp)# network 10.10.10.0 255.255.255.0
Router(config-dhcp)# default-router 10.10.10.1
Router(config-dhcp)# dns-server 10.10.10.2
Router(config-dhcp)# lease 7 0 0

! DHCP Client
Router(config)# interface Gi0/1
Router(config-if)# ip address dhcp

! DHCP Relay (on the router closest to the client subnet)
Router(config-if)# ip helper-address 10.0.0.5
```

---

## 🔹 Key Design Takeaways
- DHCP = automatic, conflict-free, centrally-managed IP addressing.
- DORA: Discover → Offer → Request → Ack (first and third are broadcast, by design).
- Leases are temporary — renewal happens quietly in the background.
- Relay agents (`ip helper-address`) are what let one server serve many remote subnets.
- DHCP is trust-based and unauthenticated by default — DHCP Snooping is the standard real-world fix.

---

## 🎤 Interview-Ready Answer
"DHCP automates IP addressing using a four-message exchange — Discover, Offer, Request, Ack — where the
first and third messages are deliberately broadcast so the client can find any server and so competing
servers know when their offer wasn't chosen. Addresses are leased, not owned, and renewal happens quietly
in the background before expiry. Since DHCP is broadcast-based, it can't cross a router by itself — that's
what `ip helper-address` relay agents are for. And because it's unauthenticated by default, DHCP Snooping
on switches is the standard defense against rogue servers and starvation attacks."

## 🧠 Memory Map (for recall)
```
DHCP → Auto IP address manager
   │
   ├─ Process (DORA) → Discover → Offer → Request → Ack
   │                      (bcast)          (bcast)
   ├─ Lease → Temporary, renewed quietly, returned to pool on expiry
   ├─ Cross-subnet → Relay agent (ip helper-address) unicasts to real server
   ├─ Options → GW(3), DNS(6), Lease(51), TFTP/PXE(66)
   ├─ Security risk → Rogue server, starvation attack
   ├─ Defense → DHCP Snooping (trusted/untrusted ports) → feeds DAI + IP Source Guard
   └─ Interview angle → Why (manual config doesn't scale) + How (DORA) + Design (relay, security)
```
