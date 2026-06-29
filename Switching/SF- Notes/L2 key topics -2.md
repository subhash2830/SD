# Simplified Notes – Core L2, VPN, MPLS, EVPN, DCI & DCB

> Focus: fast recall for interviews and design discussions.

---

## Layer 2 Fundamentals

### 1. MAC Address Table (CAM)

- Stores **MAC → port** mappings.
- Learns source MAC on ingress; floods unknown unicast/broadcast/multicast in VLAN.
- CAM overflow → unknown unicast flooding (attack surface).

**Hook:** “CAM = who is where; overflow = flood.”

---

### 2. TCAM

- 3 states: **0, 1, X (don’t care)**.
- Used for **ACLs, QoS, routing lookups** at line rate.
- Finite and expensive; **TCAM exhaustion** forces software processing.

**Hook:** “TCAM = policy brain; carve wisely.”

---

### 3. STP (802.1D)

- Prevents L2 loops via **Root Bridge** and port roles/states.
- Root = **lowest Bridge ID (priority + MAC)**.
- States: **Blocking → Listening → Learning → Forwarding**; convergence ~30–50s.

**Hook:** “One root, one path, slow timers.”

---

### 4. RSTP (802.1w)

- Speeds convergence to sub-second.
- Roles: **Root, Designated, Alternate, Backup**.
- Uses **proposal/agreement** instead of timers.

**Hook:** “Alternate port ready = fast failover.”

---

### 5. MST (802.1s)

- Multiple VLANs → few **MST instances (MSTI)** to save CPU/memory.
- **Region** = same MST name, revision, VLAN–instance mapping.
- **IST (instance 0)** represents region to outside.

**Hook:** “Region + IST = single tree face to outside.”

---

### 6. PVST+ / Rapid PVST+

- One STP instance **per VLAN** (Cisco).
- Allows **per-VLAN root** for load sharing.
- High VLAN count → high overhead, MST solves this.

**Hook:** “Per-VLAN STP = flexible but heavy.”

---

### 7. STP Features (Access & Protection)

- **PortFast**: host ports go directly to Forwarding.
- **BPDU Guard**: shuts PortFast port on BPDU (rogue switch protection).
- **BPDU Filter**: suppress BPDUs (dangerous globally).
- **Root Guard**: stops port from becoming root port.
- **Loop Guard**: prevents unintended Designated state on BPDU loss.

**Hook:** “PortFast + Guard on access; Root/Loop Guard in core.”

---

## VLAN & Trunking

### 8. VLANs & 802.1Q

- VLAN = logical broadcast domain.
- 802.1Q adds **4-byte tag** (TPID 0x8100 + PCP + DEI + VID).
- **Native VLAN** untagged; mismatch → VLAN hopping risk.

**Hook:** “Tag everything; align native VLANs.”

---

### 9. VTP (Cisco)

- Distributes VLAN configs over trunks.
- Modes: **Server, Client, Transparent, Off**.
- **Revision number** risk: higher rev can wipe domain.

**Hook:** “VTP can be VLAN killer; many designs prefer Transparent/Off.”

---

### 10. QinQ (802.1ad)

- **Double-tagging**: outer S-tag (provider), inner C-tag (customer).
- Preserves customer VLAN space per customer.
- Adds 4 bytes → MTU implications.

**Hook:** “S-tag outside, C-tag inside; watch MTU.”

---

### 11. Private VLAN (PVLAN)

- Sub-segmentation inside a VLAN.
- Types:
  - **Primary** = container.
  - **Isolated** = only to promiscuous.
  - **Community** = within community + promiscuous.
- **Promiscuous** port = usually gateway.

**Hook:** “Same subnet, different talk rights.”

---

## VPN – Generic

### 12. VPN Basics

- Creates **secure tunnel** over untrusted network.
- Types:
  - **Site–to–Site**: fixed networks.
  - **Remote Access**: users.
- Provides **confidentiality, integrity, authentication**.

**Hook:** “Encrypted overlay over internet.”

---

### 13. IPsec

- L3 security protocol suite.
- Modes:
  - **Transport**: encrypt payload.
  - **Tunnel**: encrypt entire packet.
- Core pieces: **AH/ESP, IKE, SA, keys**.

**Hook:** “ESP + IKE = IPsec core.”

---

### 14. DMVPN (Cisco)

- Hub–spoke design with **on-demand spoke–spoke tunnels**.
- Uses **mGRE + NHRP + IPsec**.
- Phases:
  - Phase 1: hub–spoke only.
  - Phase 2/3: spoke–spoke allowed.

**Hook:** “Hub control; dynamic spoke mesh.”

---

### 15. FlexVPN (Cisco)

- IKEv2-based unified VPN framework.
- Replaces EZVPN/older IPsec designs.
- Supports **site-to-site, RA, DMVPN-like** scenarios.

**Hook:** “One IKEv2 framework for all VPN types.”

---

## MPLS – Core Concepts

### 16. MPLS Basics

- Inserts **label header** between L2 and L3.
- **Label switching** based on LFIB, not IP route table.
- Supports L3VPN, L2VPN, TE, etc.

**Hook:** “Forward by label, not IP.”

---

### 17. Label Stack & Operations

- Label stack supports **multiple labels**.
- Operations:
  - **Push**: add label.
  - **Swap**: replace label.
  - **Pop**: remove label.

**Hook:** “Push at edge, swap in core, pop near egress.”

---

### 18. LDP – Label Distribution Protocol

- Advertises labels for IGP prefixes (best-effort).
- Builds LSPs along **IGP shortest paths**.
- Typically PE–PE transport for L3VPNs.

**Hook:** “LDP follows IGP; simple but no TE.”

---

### 19. RSVP-TE

- Builds **explicit LSPs** with bandwidth/constraint-based paths.
- Uses **PATH/RESV** messages; per-LSP state in core.
- Enables **traffic engineering**.

**Hook:** “RSVP-TE = strict path with state.”

---

### 20. MPLS L3VPN (RFC 4364)

- PE–PE VPN using **MP-BGP** + MPLS.
- Uses **VRFs**, route targets, route distinguishers.
- Data path: **outer transport label** + **inner VPN label**.

**Hook:** “Two labels: transport + service.”

---

### 21. MPLS L2VPN (VPWS / VPLS)

- **VPWS**: point-to-point Ethernet pseudowires.
- **VPLS**: multipoint Ethernet over MPLS.
- Transparent to customer; provider switches on pseudo MACs.

**Hook:** “VPWS = p2p; VPLS = p2mp LAN-like.”

---

## Segment Routing (SR)

### 22. SR-MPLS Basics

- Uses **label stack as SID stack** (Segments).
- No LDP/RSVP; SIDs derived from IGP.
- **Node-SID, Adj-SID, Prefix-SID**.

**Hook:** “IGP + SIDs = SR; controller optional but powerful.”

---

### 23. SRv6 Basics

- Uses **IPv6 addresses as SIDs**.
- SRH (Segment Routing Header) carries SID list.
- Service programming: SIDs can represent functions, not just hops.

**Hook:** “SID = IPv6 instruction.”

---

## EVPN – Modern L2/L3 VPN

### 24. EVPN Basics

- BGP-based control plane for **L2/L3 services**.
- Replaces flood-heavy L2VPN (VPLS) with **control-plane signaled MAC/IP**.
- Supports **multihoming with DF election**.

**Hook:** “EVPN = BGP for MAC + IP.”

---

### 25. EVPN Route Types (High Level)

- **RT-1**: per-EVI MAC/IP advertisement.
- **RT-2**: MAC/IP route.
- **RT-3**: Inclusive Multicast (BUM).
- **RT-5**: IP prefix route.
- **RT-7/8/9+**: specialized.

**Hook:** “RT-2 = MAC/IP; RT-3 = BUM, RT-5 = IP.”

---

### 26. EVPN-VXLAN DC

- Overlays L2/L3 fabrics on **IP underlay**.
- VTEPs: **leaf switches** encapsulate/decapsulate VXLAN.
- EVPN handles **MAC/IP learning and multihoming**.

**Hook:** “Leaf = VTEP; BGP EVPN = MAC route engine.”

---

## DCI – H-VPLS, PBB, OTV, MEF

### 44. H-VPLS

- **Spoke–hub** extension of VPLS.
- Spokes connect to hub; hub connects to core VPLS mesh.
- Reduces PWs; trade-off is suboptimal spoke–spoke path.

**Hook:** “Hub saves PWs; adds extra hop.”

---

### 45. PBB (802.1ah)

- **MAC-in-MAC**: C-MAC inside B-MAC.
- Provider core sees only **B-MAC**, improves MAC scale.
- Uses **B-VID + I-SID** to identify services.

**Hook:** “Hide customer MACs with backbone MAC.”

---

### 46. OTV (Cisco DCI)

- L2 extension over IP specifically for DC interconnect.
- Built-in **MAC routing** and **site-awareness**; stops STP across WAN.
- Competes conceptually with EVPN/VXLAN but Cisco-proprietary.

**Hook:** “OTV = DC VLAN stretch with built-in control.”

---

### 47. MEF Carrier Ethernet

- Service types:
  - **E-Line**: point-to-point.
  - **E-LAN**: multipoint.
  - **E-Tree**: rooted multipoint (hub–leaf).
  - **E-Access**: wholesale access.
- SLA attributes: CIR, EIR, CBS, EBS.

**Hook:** “E-Line p2p, E-LAN mesh, E-Tree hub–leaf.”

---

## Data Center Bridging (Lossless Ethernet)

### 48. Priority Flow Control (PFC – 802.1Qbb)

- Per-priority PAUSE (8 CoS queues).
- Used for **lossless flows** (FCoE, RoCE).
- Risk: **PFC storms / deadlock** if misconfigured.

**Hook:** “Pause only storage class; prevent deadlock.”

---

### 49. ETS (802.1Qaz)

- Bandwidth sharing between traffic classes.
- Strict-priority + ETS groups; ensures minimum BW for each class.
- Works with PFC to share converged links.

**Hook:** “Carve bandwidth slices per class.”

---

### 50. DCBX

- LLDP-based **auto-negotiation** of DCB settings.
- Exchanges PFC/ETS/app priorities between NIC and switch.
- Mis-match detection; can shut ports to protect fabric.

**Hook:** “DCBX = ensure consistent lossless config.”

---