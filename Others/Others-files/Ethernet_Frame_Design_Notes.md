# 📘 Ethernet Frame Formats (Design Notes + Memory Map)

## 🔹 Why This Matters (The Problem)
**Problem:** Ethernet is just a Layer 2 delivery mechanism — it moves bytes between two MAC addresses on a
wire. But many different Layer 3 protocols (IP, IPX, AppleTalk, IS-IS) might all want to ride over the same
Ethernet segment at the same time. Something in the frame has to say **"here's what's actually inside this
payload"** so the receiving device knows which protocol stack should process it.

**Design Principle:** There have historically been two competing ways to solve this: **Ethernet II**
(simple — a single 2-byte field says the payload type directly) and **IEEE 802.3** (a separate,
standards-body version that originally used that same field just for *length*, and needed an extra header
— LLC — bolted on to identify the payload type instead).

---

## 🔹 The Two Frame Formats

### 1. Ethernet II (a.k.a. "DIX" Ethernet — the one used almost everywhere today)
```
| Preamble | SFD | Dest MAC | Src MAC | EtherType | Payload (46-1500 bytes) | FCS |
|  7 bytes | 1B  |  6 bytes | 6 bytes |  2 bytes  |                          | 4B  |
```
- The 2-byte field directly after the source MAC is the **EtherType** — it tells the receiver exactly
  which Layer 3 protocol is inside.
- Common EtherType values:
  - `0x0800` → IPv4
  - `0x0806` → ARP
  - `0x86DD` → IPv6
- This is the format almost all modern traffic uses — IP, ARP, and IPv6 all ride directly in Ethernet II.

### 2. IEEE 802.3 (needs an LLC header to identify the payload)
```
| Preamble | SFD | Dest MAC | Src MAC | Length | LLC Header | Payload | FCS |
|  7 bytes | 1B  |  6 bytes | 6 bytes | 2 bytes|  3+ bytes  |         | 4B  |
```
- The same 2-byte field is instead a **Length** field — how many bytes of payload follow.
- Because there's no "type" field anymore, 802.3 needs the **802.2 LLC header** stapled onto the front of
  the payload to say what protocol is actually inside.

### How a receiving NIC tells the two apart
The single 2-byte field right after the source MAC is dual-purpose, and its **value** decides how it's
interpreted:
- **Value ≥ 1536 (0x0600)** → it's an **EtherType** → this is an Ethernet II frame.
- **Value ≤ 1500** → it's a **Length** → this is an IEEE 802.3 frame.

👉 **Design Insight:** This works cleanly only because 1500 bytes (the standard Ethernet MTU) is always
smaller than 1536 — the two ranges never overlap, so there's no ambiguity. This is a deliberate,
elegant piece of backward-compatible protocol design, not a coincidence.

---

## 🔹 The LLC Header (802.2) — What It Actually Does
LLC exists purely to give 802.3 frames back the "what's inside this payload" ability that Ethernet II gets
for free from its EtherType field.

```
| DSAP (1B) | SSAP (1B) | Control (1-2B) |
```
- **DSAP** (Destination Service Access Point) and **SSAP** (Source Service Access Point) — each just 1
  byte, identifying the protocol at each end.
- **Control** — usually 1 byte for an "Unnumbered Information" (UI) frame, which is what almost all LLC
  traffic uses in practice.

**The limitation:** DSAP/SSAP are only **1 byte each** — that's only 256 possible protocol codes, and many
values are already reserved by the standards body. This isn't enough address space for the modern world of
protocols.

### LLC + SNAP — the fix for LLC's tiny address space
When **DSAP = SSAP = 0xAA**, this is a special reserved value meaning "look past me — there's a **SNAP**
(SubNetwork Access Protocol) header right after the LLC header, and *that's* where the real protocol ID
lives."
```
| LLC (DSAP=AA, SSAP=AA, Ctrl=03) | OUI (3B) | Protocol ID (2B) | Payload |
```
- **OUI** (Organizationally Unique Identifier) — 3 bytes, identifies the vendor/organization (the same OUI
  block used in the first half of a MAC address).
- **Protocol ID** — 2 bytes, vendor-specific protocol number (for OUI `00-00-00`, these values line up
  directly with standard EtherType numbers).

👉 **Design Insight:** SNAP is really just "borrowing" Ethernet II's EtherType concept and smuggling it
inside an 802.3/LLC frame, specifically to work around LLC's cramped 1-byte SAP field.

---

## 🔹 Real Protocol Examples
| Protocol | Frame Type Used | Why |
|---|---|---|
| **IPv4 / IPv6 / ARP** | Ethernet II | Simple, direct EtherType — no LLC/SNAP needed |
| **IS-IS** | IEEE 802.3 + plain LLC (no SNAP) | Uses a reserved DSAP/SSAP value (0xFE) directly — doesn't need SNAP's extra vendor/protocol ID space, since IS-IS only ever needs the one fixed SAP value |
| **Legacy IPX (Novell)** | IEEE 802.3 + LLC + SNAP (or raw 802.3 in very old "Ethernet_802.3 raw" mode) | Needed a real protocol identifier distinct from other LLC traffic on the wire |
| **Legacy AppleTalk (Phase 2)** | IEEE 802.3 + LLC + SNAP | Same reasoning as IPX — needed SNAP's vendor/protocol ID space |

**Correction worth flagging:** IS-IS does **not** use a SNAP protocol ID like `0xCC01` — it rides directly
on plain 802.2 LLC using the reserved DSAP/SSAP value `0xFE` (control byte `0x03`), with no SNAP header at
all. This is a common point of confusion since most other legacy LLC protocols *do* need SNAP.

---

## 🔹 Frame Size and the 802.1Q VLAN Tag
- **Minimum Ethernet frame**: 64 bytes (Dest MAC through FCS). If real payload is smaller than 46 bytes,
  padding is added — this minimum exists so collision detection on old shared-media Ethernet had enough
  time to work correctly (a legacy reason that still shapes the standard today).
- **Standard maximum frame**: 1518 bytes (1500-byte MTU + 18 bytes of headers/FCS).
- **802.1Q VLAN tag**: an extra 4 bytes inserted *between* the Source MAC and the EtherType/Length field,
  carrying the VLAN ID and priority (CoS) bits — this bumps the max frame size to 1522 bytes and is the
  single most common frame modification in any modern switched network.
- **Jumbo frames**: non-standard, vendor/deployment-specific, typically up to 9000 bytes — used in data
  centers and storage networks (iSCSI, vMotion, etc.) to reduce per-packet overhead for large transfers.

---

## 🔹 Key Design Takeaways
- Ethernet II uses a direct EtherType field — simple, and what virtually all modern IP traffic uses.
- IEEE 802.3 uses a Length field instead, and needs LLC (and often SNAP) bolted on to identify the payload.
- The receiver tells the two formats apart purely by whether the 2-byte field is ≥1536 (type) or ≤1500
  (length) — no overlap, no ambiguity, by design.
- SNAP exists solely to work around LLC's 1-byte SAP field being too small for the real world.
- IS-IS is the classic real-world example of plain 802.3+LLC (no SNAP) still in active, modern use.

---

## 🎤 Interview-Ready Answer
"There are two Ethernet frame formats sharing the same 2-byte field after the source MAC: Ethernet II uses
it directly as an EtherType — simple, and what nearly all modern IP/ARP/IPv6 traffic uses — while IEEE
802.3 uses it as a Length field instead, and needs an LLC header, and sometimes a SNAP header on top of
that, to identify the payload. The receiver tells them apart purely by value: 1536 or higher means
EtherType, 1500 or lower means Length, and those ranges never overlap by design. IS-IS is the classic
modern example still riding on plain 802.3 with LLC — no SNAP needed — while legacy protocols like IPX and
AppleTalk needed SNAP's extra OUI and Protocol ID fields to get enough address space."

## 🧠 Memory Map (for recall)
```
Ethernet Frame → How payload type is identified
   │
   ├─ Ethernet II → EtherType field (direct) → IPv4(0800), ARP(0806), IPv6(86DD)
   │                  used by: virtually all modern traffic
   │
   ├─ IEEE 802.3 → Length field (not type) → needs LLC header to identify payload
   │      │
   │      ├─ LLC (DSAP+SSAP+Control) → 1-byte SAP = limited protocol space
   │      │        used by: IS-IS (DSAP/SSAP = 0xFE, no SNAP needed)
   │      │
   │      └─ LLC + SNAP (DSAP=SSAP=0xAA) → adds OUI(3B) + Protocol ID(2B)
   │               used by: legacy IPX, AppleTalk Phase 2
   │
   ├─ Disambiguation rule → field value ≥1536 = EtherType, ≤1500 = Length
   │
   └─ Modern extras → 802.1Q VLAN tag (+4 bytes, between Src MAC and Type/Length),
                       Jumbo frames (~9000B, data center/storage use)
```
