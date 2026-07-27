# CCIE/CCDE — EVPN (Ethernet VPN)
*Simple explanations, CCDE-level design depth, interview answers, CLI, and a concept memory map — covering all 15 EVPN labs.*

---

## 1. EVPN VPWS (Single-Homed)

**What:** EVPN VPWS delivers the exact same point-to-point service as classic VPWS (VPWS notes, Section 1), but the discovery/signaling is done **entirely via BGP** (the EVPN AFI/SAFI, AFI 25 / SAFI 70) — **no LDP/tLDP session at all**, and critically, **no pseudowire object exists in the traditional sense**.
```
router bgp 100
 address-family l2vpn evpn
 neighbor 10.10.10.1 address-family l2vpn evpn
!
l2vpn xconnect group evpn-vpws p2p evpn1
 interface TenGigE0/1/0/2
 neighbor evpn evi 100 target 10 source 12
```
**Why it matters (CCDE lens):** This is the direct completion of the trajectory started in VPLS's "BGP-only signaling" (VPLS notes, Section 3) — EVPN removes LDP from L2VPN services entirely, using one consistent BGP-based control plane for discovery AND label distribution across VPWS, E-LAN, and (later) L3VPN-integrated services. The **source/target AC-ID pair** (not a manually-paired remote-PE-IP + PW-ID like classic VPWS) is what identifies the service — this is a fundamentally different mental model: you're not "building a pseudowire to a peer," you're "advertising that this AC participates in EVI X, AC-ID Y" and letting BGP figure out who else is in that EVI.
**Real-world example:** Any greenfield SP L2VPN buildout in the last several years defaults to EVPN VPWS over classic VPWS specifically to consolidate the control plane onto BGP already used for L3VPN — one signaling protocol, one address-family family to operate and troubleshoot, instead of LDP+BGP-AD's two-protocol dependency.

---

## 2. EVPN VPWS Multihomed (All-Active)

**What:** Multihoming a CE to two PEs is achieved via an **Ethernet Segment Identifier (ESI)** — a shared identifier configured identically on both PEs' bundle interfaces — rather than any inter-PE MC-LAG control protocol. The ESI is how the two PEs know "we're both connected to the same physical CE," purely through BGP-advertised routes, with **no direct peer-to-peer signaling link between the two PEs required for this purpose**.
```
#IOS-XR
evpn
 interface BE1
  ethernet-segment
   identifier type 0 00.11.11.22.22.33.33.44.00

#IOS-XE
interface Port-channel1
 evpn ethernet-segment 1
  identifier type 3 system-mac 1111.2222.3333
  redundancy all-active
```
**Why it matters (CCDE lens) — this is the single most important architectural point in EVPN:** Classic multihoming (MC-LAG, VPLS's Redundant-VPWS/H-VPLS-redundancy) requires a **proprietary inter-chassis control protocol** between the two redundant PEs to coordinate state — vendor-specific, complex, requiring a dedicated inter-chassis link. EVPN eliminates this entirely: the two PEs never need to talk to each other directly at all — they each independently advertise BGP routes (Type 1 per-ES route, Type 4 DF-election route) referencing the shared ESI, and remote PEs use standard BGP best-path/multipath logic to figure out that both are valid targets for that ESI. This is a genuine paradigm shift: multihoming becomes a **pure control-plane (BGP) function** instead of requiring dedicated inter-PE plumbing.
**A VPWS-specific nuance:** because point-to-point VPWS has no concept of BUM (broadcast/unknown-unicast/multicast) traffic, **DF (Designated Forwarder) election doesn't actually matter for VPWS all-active** — both PEs simply forward, and remote PEs load-balance across both via the label binding representing the ESI (not a specific PE). DF election becomes meaningful only once you introduce E-LAN/bridging (Section 4+), where BUM replication genuinely needs exactly one active forwarder to avoid duplication.
**Real-world example:** A CE dual-homed to two PEs at an IXP or critical customer site gets automatic load-sharing and PE-failure protection with zero MC-LAG configuration on either PE — just matching ESI values — collapsing what used to be a vendor-specific, fragile inter-chassis dependency into a standard, vendor-interoperable BGP mechanism.

---

## 3. EVPN VPWS Multihomed Single-Active

**What:** Same ESI mechanism as Section 2, but `load-balancing-mode single-active` restricts forwarding for that ESI to **exactly one** of the two connected PEs at a time — the other stands by as backup, activating only on failure.
```
interface Bundle-Ether7
 ethernet-segment
  identifier type 0 00.00.00.00.00.00.00.00.07
  load-balancing-mode single-active
```
**Why it matters (CCDE lens):** This is a genuine, deliberate design choice, not a fallback/limitation — single-active is chosen when the **access-side link itself** (or the CE's own capability) can't or shouldn't handle active-active load-sharing (e.g., a CE that isn't doing proper LACP-based hashing, or a service SLA that specifically wants a clean, single active path for troubleshooting/predictability rather than maximum bandwidth utilization). All-active maximizes bandwidth utilization across both PE uplinks; single-active maximizes simplicity/predictability at the cost of using only half the available capacity in steady state. A CCDE candidate should be able to justify choosing single-active over all-active on requirements grounds, not just describe the config difference.
**Operational requirement worth noting:** the lab requirement "if a PE loses both its core-facing interfaces, it should bring down the AC-side bundle member" describes **core isolation** — without this, a PE cut off from the core but still physically connected to the CE would keep advertising itself as viable, blackholing traffic sent to it. This is the EVPN-multihoming equivalent of BFD/IGP fast failure detection applied specifically to the access-to-core dependency chain.

---

## 4. Basic Single-Homed EVPN E-LAN

**What:** The EVPN equivalent of basic VPLS (VPLS notes, Section 1) — full any-to-any multipoint bridging across PEs — but again entirely BGP-signaled, with MAC addresses learned and advertised via **EVPN Type 2 (MAC/IP Advertisement) routes** instead of being learned only in the dataplane.
**Why it matters (CCDE lens) — the biggest conceptual leap from VPLS:** In classic VPLS, MAC learning is purely a **dataplane** function — a PE only learns a remote MAC when it actually receives dataplane traffic sourced from it, and only floods to find it via BUM traffic (standard bridge learning). In EVPN E-LAN, MAC/IP bindings are **advertised as BGP routes** — a PE can learn a remote MAC's location from the control plane *before* any dataplane traffic ever arrives, eliminating the "flood-and-learn" unknown-unicast flooding that VPLS relies on for initial MAC discovery. This is a genuinely different (and generally superior) learning model: BGP convergence properties (fast propagation, established RR infrastructure, established best-path/policy tooling) now apply to MAC mobility and reachability, not just IP routes.
**Real-world example:** When a host moves between sites (e.g., VM migration in a DC-interconnect EVPN design), the new location's MAC route is advertised via BGP and old bindings withdrawn — much faster and more deterministic and DC-mobility-aware than waiting for classic VPLS's flood-and-relearn dataplane behavior to catch up.

---

## 5. EVPN E-LAN Service Label Allocation

**What:** Governs how a PE allocates the MPLS label carried in EVPN Type 2 MAC routes — **per-EVI** (one shared label for the whole EVI, more label-space-efficient) vs. **per-MAC** (a distinct label per advertised MAC, enabling more granular per-flow treatment at the cost of consuming more label space).
**Why it matters (CCDE lens):** This is a direct **scale-vs-granularity trade-off**, the same category of decision as LDP's local-label-allocation-filtering (LDP notes, Section 7) — per-EVI allocation is the default and right choice for the overwhelming majority of deployments (label space is precious, and per-EVI is sufficient since forwarding decisions are made by MAC lookup in the bridge table regardless of label granularity); per-MAC allocation is a specialized choice only justified by a genuine per-MAC differentiated-treatment requirement (rare). A CCDE candidate should default to recommending per-EVI and be able to articulate specifically why you'd deviate.

---

## 6. EVPN E-LAN Ethernet Tag

**What:** The Ethernet Tag field in EVPN NLRIs allows a single EVI to represent **multiple distinct bridge domains/VLANs** (VLAN-aware bundle service model) rather than a strict 1:1 EVI-to-bridge-domain mapping — the tag disambiguates which specific VLAN/bridge-domain within the EVI a given route applies to.
**Why it matters (CCDE lens):** This is the EVPN mechanism that supports the **VLAN-Aware Bundle** service model (multiple customer VLANs sharing one EVI's control-plane state/route advertisements) as an alternative to the simpler **VLAN-Based** model (strict one EVI per VLAN/bridge-domain). VLAN-Aware Bundle reduces the number of distinct EVI/RT combinations needed for a customer with many VLANs — a direct scale lever, conceptually similar to QinQ's role in H-VPLS (VPLS notes, Section 8) letting one access construct carry many customer services, just expressed at the BGP control-plane layer instead of the dataplane tag layer.

---

## 7. EVPN E-LAN Multihomed

**What:** Combines Section 4's E-LAN bridging with Section 2/3's ESI-based multihoming — and this is where **DF election becomes operationally meaningful** (unlike VPWS multihoming, where it was largely a formality). With multiple PEs multihomed to the same ESI in a bridged/BUM-capable service, exactly one PE (the DF, elected via Type 4 routes, lowest-IP-wins by default) is responsible for forwarding BUM traffic onto that Ethernet Segment — preventing duplicate broadcast delivery to the CE.
**Why it matters (CCDE lens):** This is the direct parallel to VPLS's split-horizon-group loop-avoidance concern (VPLS notes, Section 1), but solved through an entirely different, BGP-native mechanism (DF election via Type 4 routes) rather than a dataplane flooding rule. Understanding *why* DF election matters here but was largely irrelevant for pure VPWS (Section 2/3) — the presence or absence of BUM traffic — is a specific, testable distinction.
**Split-horizon for known-unicast, aliasing for load-balancing:** EVPN also introduces per-EVI Type 1 "Ethernet Auto-Discovery" routes used for **aliasing** — allowing remote PEs to load-balance known-unicast traffic across all PEs connected to a multihomed ESI (not just the DF), even though only the DF forwards BUM. This is a subtlety worth knowing: DF election governs BUM forwarding only; unicast can still be load-shared across all active PEs on that ESI via aliasing.

---

## 8. EVPN E-LAN on XRv

**What:** The same E-LAN service model exercised specifically on the XRv platform, validating any XRv-specific behavior/CLI differences versus the CSR1000v (IOS-XE) results seen in earlier labs.
**Why it matters (CCDE lens):** As with the VPWS labs' explicit note that "multihoming appears to only be supported for bridged EVPN" on CSR1000v (not VPWS) at the tested software version, platform/software-version feature-parity gaps are a real, concrete design-validation concern for EVPN specifically — EVPN is a rapidly-evolving feature area, and assuming full feature parity across every platform/software combination without validating is a real risk in a production design, not just a lab curiosity.

---

## 9. EVPN IRB (Integrated Routing and Bridging)

**What:** Attaches a BVI/BDI routed interface to an EVPN bridge-domain (same underlying concept as VPLS+IRB, VPLS notes Section 10), but with EVPN's BGP control plane additionally supporting **Type 5 (IP Prefix) routes** for pure L3 reachability, and **route "stitching"** between the L3VPN (VPNv4/v6) and EVPN address-families at the VRF boundary.
```
vrf def A
 rd 65000:50
 address-family ipv4
  route-target both 65000:50
  route-target both 65000:50 stitching
  route-target import 65000:100
!
int bdi 10
 vrf forwarding A
 ip address 192.168.10.254 255.255.255.0
 mac-address 0000.1111.2222
!
router bgp 65000
 address-family ipv4 vrf A
  advertise l2vpn evpn
  redistribute connected
```
**Why it matters (CCDE lens):** This is the architectural culmination of the whole VPWS→VPLS→EVPN trajectory across all three note sets — EVPN IRB unifies L2 (bridging) and L3 (routing/VPN) services under **one BGP control plane**, with `advertise l2vpn evpn` explicitly translating VPNv4/v6 routes into EVPN Type 5 routes at the boundary, and a separate **stitching RT** controlling exactly which routes cross that L2↔L3 translation boundary (distinct from the normal RTs governing intra-address-family VPNv4/EVPN import/export). Note the shared `mac-address` configured identically across PEs (0000.1111.2222 in the example) — this is the **anycast gateway** mechanism: every PE hosting that bridge-domain answers ARP/ND with the *same* MAC, so a host's default-gateway ARP entry remains valid and reachable regardless of which PE it's actually attached to or which PE happens to be active — true active-active L3 gateway redundancy, the capability VPLS+IRB (typically single-active) could not cleanly provide.
**Real-world example:** A DC-interconnect / campus design where hosts on the same subnet are spread across multiple EVPN PEs (e.g., different racks/PoDs) — anycast-gateway IRB means every host's default gateway is reachable locally at its own attaching PE, with no suboptimal "trombone" routing to a single central gateway router the way classic VPLS+IRB would typically require.

---

## 10–13. EVPN-VPWS Multihomed IOS-XR: All-Active / Port-Active / Single-Active / Non-Bundle

**What these four variants cover, as a set:**

| Mode | Behavior |
|---|---|
| **All-Active** | Both PEs forward simultaneously (Section 2) — ECMP-style load-sharing to the CE |
| **Single-Active** | Only one PE forwards at a time per ESI (Section 3) — clean failover, no active-active complexity |
| **Port-Active** | A variant of active/standby specifically at **individual port granularity** rather than per-ESI/per-bundle — relevant when the AC isn't a LAG/bundle at all, just individual physical ports each independently designated active or standby |
| **Non-Bundle** | Demonstrates ESI-based multihoming **without a Port-Channel/Bundle-Ether AC at all** — a single physical interface per PE, with the ESI still providing the multihoming relationship purely through BGP, no LACP/bundling involved on the access side |

**Why it matters (CCDE lens):** The Non-Bundle case is the most conceptually important of the four to internalize: it proves that EVPN multihoming's ESI mechanism is **fundamentally independent of Layer 2 bundling/LACP** — the ESI is a BGP-control-plane construct, and bundling (Port-Channel/LACP) is an *optional* access-layer technique that can be layered on top of it for additional per-link resiliency, not a prerequisite for EVPN multihoming to function at all. This matters for real designs where the access-side CE device may not support LACP, or where a single-port-per-PE topology is simpler/cheaper and still needs PE-level (not port-level) redundancy.
**Design guidance:** Choose All-Active when the CE and access links can genuinely support active-active hashing and you want maximum bandwidth utilization; Single-Active or Port-Active when the CE/access design calls for a simpler, single-path-at-a-time model; Non-Bundle whenever LACP/bundling isn't available or desired on the access side but PE-level multihoming redundancy is still required.

---

## 14. PBB-EVPN (Provider Backbone Bridge EVPN) — Informational

**What:** Combines 802.1ah Provider Backbone Bridging (MAC-in-MAC encapsulation — wrapping the customer's entire Ethernet frame, including their MAC addresses, inside an outer provider B-MAC header) with EVPN as the control plane for distributing **B-MAC reachability** (not customer C-MAC reachability) across the core.
**Why it matters (CCDE lens):** The core scale benefit: core/P-router and even PE control-plane state scales with the number of **provider B-MACs** (one per PE, roughly), completely independent of how many thousands of individual **customer C-MACs** exist behind each PE — a genuine MAC-scale isolation technique for extremely large multi-tenant environments (e.g., a metro Ethernet aggregation network with an enormous customer MAC population) where even EVPN's improved (BGP-based, still per-customer-MAC) learning model would still need to carry every individual customer MAC as a distinct route. PBB-EVPN trades this scale benefit for additional encapsulation complexity (MAC-in-MAC) and is a specialized, less commonly deployed design pattern — correctly flagged as "informational" in this workbook rather than a mainstream recommendation for most SP designs today.
**Real-world example:** A very large metro Ethernet aggregation network serving tens of thousands of customer devices across a wide access footprint might consider PBB-EVPN specifically to decouple core BGP/control-plane scale from the (much larger and faster-growing) customer C-MAC population — a niche but real scale consideration for the largest access-aggregation designs.

---

## 15. CCDE-Style Interview Q&A

**Q1. What's the single biggest architectural difference EVPN introduces for multihoming compared to classic MC-LAG-based approaches?**
EVPN multihoming requires no direct inter-chassis control protocol or dedicated link between the two redundant PEs at all — both independently advertise standard BGP routes (Type 1 per-ES, Type 4 DF-election) referencing a shared Ethernet Segment Identifier, and remote PEs use ordinary BGP best-path/multipath logic to resolve reachability. Multihoming becomes a pure BGP control-plane function instead of requiring proprietary inter-chassis plumbing.

**Q2. Why does DF election matter for EVPN E-LAN multihoming but appear to be largely a formality for EVPN VPWS multihoming?**
DF election governs which PE forwards BUM (broadcast/unknown-unicast/multicast) traffic onto a shared Ethernet Segment, to prevent duplicate delivery. Point-to-point VPWS has no concept of BUM traffic at all — every frame has one specific unambiguous destination — so there's nothing for DF election to meaningfully arbitrate. Once you introduce bridging/E-LAN, genuine BUM replication risk exists, and DF election becomes operationally necessary.

**Q3. How does MAC learning fundamentally differ between classic VPLS and EVPN E-LAN?**
VPLS learns MACs purely in the dataplane — a PE only knows a remote MAC once it actually receives traffic sourced from it, relying on flood-and-learn for initial discovery. EVPN advertises MAC/IP bindings as BGP Type 2 routes, so a PE can learn a remote MAC's location from the control plane before any dataplane traffic arrives at all — faster convergence and no dependency on BUM flooding for initial MAC discovery.

**Q4. What does the "stitching" Route Target do in an EVPN IRB design, and how is it different from the normal RT?**
The normal RT governs import/export within the L3VPN (VPNv4/v6) or EVPN address-family itself. The stitching RT specifically controls which routes get translated across the L2↔L3 boundary — from VPNv4/v6 into EVPN Type 5 routes (via `advertise l2vpn evpn`) — a separate, deliberately distinct control point from ordinary intra-address-family route filtering.

**Q5. Why would you choose EVPN VPWS Multihomed Single-Active over All-Active, given All-Active uses more available bandwidth?**
Single-Active is the right choice when the access-side CE or link design can't or shouldn't handle genuine active-active load-sharing — e.g., a CE without proper multi-path hashing capability, or an SLA that values a simple, predictable single active path for troubleshooting over maximizing throughput. It's a deliberate simplicity/predictability trade-off against bandwidth utilization, not a fallback or limitation.

**Q6. What does the "Non-Bundle" EVPN multihoming variant prove architecturally that the bundle-based variants don't?**
It demonstrates that EVPN's ESI-based multihoming mechanism is fundamentally independent of Layer 2 bundling/LACP — a single physical port per PE can still participate in ESI-based multihoming purely through BGP. Bundling is an optional additional resiliency layer on top of EVPN multihoming, not a prerequisite for it.

**Q7. When would PBB-EVPN's MAC-in-MAC approach be justified over standard EVPN?**
When core/control-plane scale needs to be decoupled from a very large and fast-growing customer C-MAC population — PBB-EVPN's core state scales with the number of provider B-MACs (roughly one per PE) rather than every individual customer MAC. This is a niche, large-scale-access-aggregation consideration, not a mainstream default recommendation.

---

## 16. Memory Map

```
EVPN Core
│
├── Signaling Foundation (1)
│     ALL discovery + label distribution via BGP EVPN AFI/SAFI
│     NO LDP/tLDP anywhere — completes the trajectory started by
│     VPLS's "BGP-only signaling" (see VPLS notes, Section 3)
│
├── Multihoming — the central EVPN innovation (2, 3, 10-13)
│     ESI = shared BGP-advertised identifier, NO inter-PE control
│     protocol required (unlike classic MC-LAG)
│     ├─ All-Active (2): max bandwidth, DF election largely
│     │     irrelevant for VPWS (no BUM concept in p2p service)
│     ├─ Single-Active (3): simplicity/predictability over bandwidth
│     ├─ Port-Active: same idea at individual-port granularity
│     └─ Non-Bundle: PROVES the ESI mechanism doesn't need
│           LACP/bundling at all — pure BGP construct
│
├── E-LAN / Bridging (4, 5, 6, 7, 8)
│     MAC learning via BGP Type 2 routes — NOT flood-and-learn
│     (direct contrast with classic VPLS's dataplane-only learning)
│     Service Label Allocation (5): per-EVI vs per-MAC —
│           same scale-vs-granularity axis as LDP local allocation
│           filtering (see LDP notes, Section 7)
│     Ethernet Tag (6): VLAN-Aware Bundle — BGP-layer analogue
│           of QinQ's role in H-VPLS (see VPLS notes, Section 8)
│     Multihomed E-LAN (7): THIS is where DF election earns its
│           keep — BUM exists here, unlike VPWS
│           + Aliasing: unicast load-balances across ALL PEs on
│             an ESI even though only the DF handles BUM
│
├── L2/L3 Unification (9)
│     EVPN IRB: Type 5 routes + RT "stitching" bridges L3VPN
│     and EVPN address-families
│     Anycast gateway (same MAC on every PE) = TRUE active-active
│     L3 gateway redundancy — the capability VPLS+IRB couldn't
│     cleanly provide (see VPLS notes, Section 10)
│     └─ ARCHITECTURAL CULMINATION of the VPWS → VPLS → EVPN
│           trajectory traced across all three note sets
│
└── Extreme-Scale Variant (14)
      PBB-EVPN: decouples core scale from CUSTOMER MAC count via
      MAC-in-MAC — niche, informational, not mainstream default
```

**The one-sentence summary of why EVPN matters, tying together every legacy mechanism seen across the VPWS/VPLS/EVPN note set:** every scale or resiliency problem that VPWS and VPLS solved piecemeal — BGP AD for discovery, H-VPLS for topology scale, Redundant-VPWS/MC-LAG for PE-failure protection, VPLS+IRB for L2/L3 integration — EVPN solves **holistically, with one BGP-based control plane**, which is the concrete, defensible answer to "why EVPN" in any design interview.

---

## 17. CLI Cheat Sheet

| Purpose | Command |
|---|---|
| Enable BGP L2VPN EVPN AF | `router bgp <asn>` → `address-family l2vpn evpn` → `neighbor <ip> address-family l2vpn evpn` |
| EVPN VPWS xconnect (single-homed) | `l2vpn xconnect group <g> p2p <name>` → `interface <int>` → `neighbor evpn evi <id> target <t> source <s>` |
| Define ESI on a bundle (XR) | `evpn` → `interface <BEx>` → `ethernet-segment` → `identifier type 0 <10-byte-esi>` |
| Define ESI on a port-channel (XE) | `interface Port-channelX` → `evpn ethernet-segment <n>` → `identifier type 3 system-mac <mac>` → `redundancy all-active` |
| Set single-active load-balancing mode | `ethernet-segment` → `load-balancing-mode single-active` |
| Match LACP system ID across PEs (XR) | `lacp system mac <mac>` under the bundle |
| Match LACP device ID across PEs (XE) | `lacp device-id <mac>` |
| EVPN E-LAN bridge-domain association | `bridge-domain <id>` → `member vfi/evpn-instance` (EVI-based, platform syntax varies) |
| Anycast gateway IRB | `interface BDI<id>` → `mac-address <shared-mac>` (identical across all PEs hosting that BD) |
| Translate VPNv4/v6 into EVPN Type 5 | `address-family ipv4 vrf <name>` → `advertise l2vpn evpn` |
| Stitching RT (L2↔L3 boundary control) | `route-target both <asn>:<id> stitching` |
| Verify EVPN routes | `show bgp l2vpn evpn` |
| Verify Ethernet Segment state | `show evpn ethernet-segment [detail]` |
| Verify xconnect / EVPN-VPWS state | `show l2vpn xconnect detail` |
| Verify DF election | `show evpn ethernet-segment detail` (look for DF role per ES) |

---
*Source: CCIE-SP v5.1 Labs — EVPN section (15 labs): EVPN VPWS, EVPN VPWS Multihomed, EVPN VPWS Multihomed Single-Active, Basic Single-homed EVPN E-LAN, EVPN E-LAN Service Label Allocation, EVPN E-LAN Ethernet Tag, EVPN E-LAN Multihomed, EVPN E-LAN on XRv, EVPN IRB, EVPN-VPWS Multihomed IOS-XR (All-Active/Port-Active/Single-Active/Non-Bundle), PBB-EVPN (Informational). Some sub-topics supplemented with standard EVPN behavior (RFC 7432, RFC 8365) and Cisco EVPN-VPWS documentation where specific lab page content was not directly retrievable.*
