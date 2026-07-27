# CCIE/CCDE — VPWS (Virtual Private Wire Service)
*Simple explanations, CCDE-level design depth, interview answers, CLI, and a concept memory map — covering all 11 VPWS labs.*

---

## 1. Basic VPWS (tLDP-Signaled Pseudowire)

**What:** A VPWS is a point-to-point L2 service — two ACs (attachment circuits) at different PEs, stitched together by a single MPLS pseudowire, signaled dynamically via **targeted LDP (tLDP)** between the two PE loopbacks.

**Two config styles, same result (IOS-XE):**
```
! Older style — pseudowire-class applied under the AC interface
pseudowire-class MPLS_CW
 encapsulation mpls
 control-word
!
int Gi4
 xconnect 11.11.11.11 511 pw-class MPLS_CW

! Newer style — l2vpn xconnect context
l2vpn xconnect context CE6_CE10
 member Gi4
 member 12.12.12.12 610 encapsulation mpls
```

**Local (same-PE) cross-connect:** doesn't need tLDP or even MPLS at all — just cross-connects two local ACs directly:
```
#IOS-XE
connect NAME Gi4 Gi5
! or, the more intuitive modern equivalent:
l2vpn xconnect context NAME
 member GigabitEthernet4
 member GigabitEthernet5
```
> **Why it matters (CCDE lens):** Recognizing when a "VPWS" is actually just a local switch-through (no MPLS/tLDP needed at all) versus a genuine multi-PE service is a basic but real design/troubleshooting distinction — don't waste time debugging LDP for a same-box cross-connect that never needed it.

**Control Word (CW) — a genuine platform-default trap:**

| Platform | Default CW behavior |
|---|---|
| IOS-XE | CW **on**, but with autosense — will adjust to match the peer |
| IOS-XR | CW **off**, and stays off unless explicitly configured via a `pw-class` |

> **Why it matters (CCDE lens):** Mixed-vendor or mixed-platform (XE+XR) VPWS deployments can come up looking "fine" at the xconnect/session level while actually running with CW disabled because IOS-XE's autosense quietly matched IOS-XR's off-by-default state. This matters because the CW disambiguates the payload framing (needed correctly for certain traffic patterns/OAM); silently ending up in a different CW state than intended is a subtle interop gotcha worth checking explicitly in any cross-platform L2VPN design, not assuming both ends "just work."

---

## 2. VPWS with Tag Manipulation

**What:** Controls how 802.1Q VLAN tags on the customer-facing AC are handled as frames enter/exit the pseudowire — pop (strip a tag before entering the PW), push (add a tag on egress), or transparently pass through — configured via the `service instance` / `encapsulation` and `rewrite` statements on the AC interface.

**Why it matters (CCDE lens):** This is the mechanism that lets a provider offer **VLAN normalization** as a managed service — the customer can use whatever VLAN ID is convenient for them at each site, and the PE rewrites it to whatever the provider's internal service tagging scheme requires (or strips it entirely for a port-based service), without the customer needing VLAN IDs to match end-to-end. Getting the rewrite direction (ingress vs egress, pop vs pop 1 symmetric) wrong is a common real-world cause of a PW coming up (control plane healthy) while customer traffic silently doesn't forward correctly (data plane broken) — another instance of the "control plane looks fine, forwarding doesn't work" class of problem seen elsewhere (LDP+static routes, transport address).
**Real-world example:** A provider onboarding two customer sites that each independently chose VLAN 100 for their WAN handoff, with no coordination between them — tag rewrite lets the provider terminate both as VLAN 100 locally while carrying them as fully distinct, isolated services across the shared MPLS core.

---

## 3. Redundant VPWS (IOS-XE & IOS-XR)

**What:** Protects against PE failure (not just link failure) by defining a **primary and backup pseudowire** to two different remote PEs, with `pw-redundancy` grouping them so only one is active/forwarding at a time.
```
#IOS-XE
l2vpn redundancy pw-class
 backup pw-class BACKUP
!
int Gi4
 xconnect 11.11.11.11 511 pw-class PRIMARY
  backup peer 12.12.12.12 511 pw-class BACKUP
```
> **Why it matters (CCDE lens):** Basic VPWS (Section 1) only protects against a *link* failure on a single AC/PE via IGP/LDP reconvergence to the same remote PE — it does nothing if the **remote PE itself** fails, since there's only ever one target. Redundant VPWS is the design answer to "what if the far-end PE dies," and is the direct conceptual predecessor to EVPN's multihoming (all-active/single-active) — same underlying business requirement (protect against PE failure), older/simpler mechanism (one active PW at a time, not a true multi-active fabric). When comparing legacy L2VPN to EVPN in a design discussion, this is the exact feature EVPN multihoming supersedes.
**Real-world example:** A customer's critical site is dual-homed to two different PEs for physical diversity — Redundant VPWS ensures that if the primary PE (or its uplink) fails, traffic fails over to the backup PW without requiring any customer-side reconfiguration or STP/L2 loop-avoidance protocol running across the WAN.

---

## 4. VPWS with PW Interfaces

**What:** Instead of applying `xconnect` directly under a physical AC interface, define a standalone `interface pseudowireX` object and cross-connect it to the AC via `l2vpn xconnect context`. Functionally equivalent to the inline `xconnect` style, but as a **first-class interface object**.
> **Why it matters (CCDE lens):** A dedicated PW interface object can carry its own QoS policy, ACLs, and be referenced independently in other configuration (e.g., as a `preferred-path` target, or bound into other constructs) in a way an inline `xconnect` statement cannot as cleanly. This is a "know both syntaxes and when the extra abstraction is worth it" topic — for simple single-purpose VPWS, inline `xconnect` is less config; for anything needing independent policy attachment to the PW itself, the PW-interface style is the better design choice.

---

## 5. Manual VPWS (No Signaling)

**What:** Bypasses tLDP entirely — `signaling protocol none`, with the local/remote service labels manually and symmetrically hard-coded on both PEs from a statically-defined label range.
```
mpls label range 10000 10999 static 11000 11999
!
int pseudowire18
 encapsulation mpls
 signaling protocol none
 neighbor 6.6.6.6 18
 label 11001 61001                 ! local-label remote-label
```
**Why it matters (CCDE lens):** You lose the entire dynamic-signaling value proposition — no AC-status propagation to the far end (if the local AC goes down, the remote PE has no way to know and the PW stays "up" from its perspective), no dynamic MTU negotiation, nothing self-healing on a label range change. The only legitimate uses are (1) interop with a device that genuinely can't run tLDP, or (2) forcing a specific deterministic label for lab/test/documentation purposes — same category of trade-off as LDP Static Labels (see the LDP notes). **Never** a general production design pattern.
> **Verification gotcha:** Because AC status isn't signaled, shutting down the AC on one PE will NOT bring the PW down on the far end — a real risk if any failure-detection logic downstream assumes PW state reflects AC state.

---

## 6. VPWS with Sequencing

**What:** Adds a sequence number to each pseudowire frame so the egress PE can detect and discard/reorder frames that arrive out of order — relevant because MPLS core ECMP (see FAT-PW, Section 7) or any core-level reordering can, in principle, deliver frames out of transmission order.
**Why it matters (CCDE lens):** Sequencing exists specifically for payloads that are **order-sensitive** at Layer 2 (some legacy TDM/circuit-emulation traffic, certain synchronization-sensitive applications) where reordering — not just loss — causes application-level failure. For ordinary IP/Ethernet payloads riding a VPWS, upper-layer protocols (TCP) already tolerate reordering reasonably well, so sequencing is a targeted feature for specific service SLAs, not a default-on setting. There's a real interplay to reason about with FAT-PW: FAT-PW deliberately spreads flows across multiple ECMP paths for load-balancing, which can *increase* the chance of reordering within what the flow label considers "one flow" if hashing isn't clean — sequencing is the safety net for services that can't tolerate that risk.

---

## 7. Pseudowire Logging

**What:** Enables syslog messages specifically for PW/xconnect state transitions (up/down, AC status changes) — separate from generic interface logging.
**Why it matters (CCDE lens):** In a large L2VPN SP core running thousands of PWs, generic interface logging is too noisy/unfocused to correlate service-impacting events quickly. Dedicated PW state logging is a genuine **operational/NOC-visibility design decision** — the same category as ISIS's `log-adjacency-changes all` — small to configure, but a real gap in incident timelines if forgotten, especially when troubleshooting an intermittent customer-reported outage after the fact.

---

## 8. VPWS with FAT-PW (Flow-Aware Transport)

**What:** Adds an extra label at the bottom of the MPLS stack that encodes a **flow identifier** — its only purpose is to give core P routers something to hash on for ECMP, since it's discarded (not forwarded) by the egress PE.
**The core problem it solves:** Without FAT-PW, every frame in a given pseudowire shares the *same* bottom (service) label — and P routers hash ECMP decisions on the bottom label. That means **all traffic in a PW takes exactly one path** through the core, regardless of how many ECMP paths physically exist, wasting available core bandwidth for any single high-volume PW.
```
#IOS-XE
template type pseudowire FAT
 encapsulation mpls
 load-balance flow ip src-dst-ip
 load-balance flow-label both
!
l2vpn xconnect context CE1_CE8
 member Gi4 service-instance 1
 member 6.6.6.6 18 template FAT

#IOS-XR
l2vpn
 pw-class FAT
  encapsulation mpls
   load-balancing
    flow-label both
```
> **Why it matters (CCDE lens):** This directly parallels BFD's echo-packet design insight (LSRs don't need to understand upper-layer headers) — FAT-PW lets P routers hash on a bottom label instead of having to inspect L3/L4 headers underneath the MPLS stack, which they normally cannot (or should not) do for a pure L2 pseudowire payload. Negotiation *requires* LDP (signaled), which is exactly why Manual VPWS (Section 5) and FAT-PW are mutually exclusive design choices — you cannot get flow-aware ECMP without dynamic signaling to negotiate the flow-label capability between peers.
**Real-world example:** A single high-bandwidth VPWS (e.g., a data-center-interconnect circuit emulation service) between two PEs connected by 4 equal-cost core paths would, without FAT-PW, use only 1 of those 4 paths for 100% of its traffic — a real capacity-planning risk for any single "elephant" PW. FAT-PW is standard practice for any large point-to-point L2 service riding an ECMP-rich core.

---

## 9. MS-PW / Pseudowire Stitching

**What:** Joins two independently-signaled pseudowire segments at a **switching PE (S-PE)** so they behave as one logical end-to-end pseudowire (multi-segment pseudowire, MS-PW) — the S-PE swaps the service label between segments rather than terminating the service itself.
**Why it matters (CCDE lens):** This is the standard design pattern for **domain boundary crossing** — e.g., stitching a metro-access LDP domain to a separate core LDP/SR domain, or crossing an administrative/AS boundary — without requiring a single end-to-end signaling session (and therefore a single flat LDP/label domain) across the entire path. It's the L2VPN equivalent of BGP confederations or Inter-AS L3VPN Option B: a controlled interconnection point between otherwise-independent signaling domains, rather than one giant flat domain end to end.
**Real-world example:** A large SP with a separately-managed metro/access network (different team, potentially different equipment vendor/signaling capabilities) and a national core stitches PWs at a well-defined S-PE boundary router — each domain can evolve its internal MPLS/label design independently as long as the S-PE stitching point contract is maintained.

---

## 10. VPWS with BGP Auto-Discovery (BGP AD)

**What:** Instead of manually configuring the remote PE address and PW-ID on both ends (FEC 128, tLDP-signaled), PEs advertise a BGP L2VPN NLRI (per RFC 6074) that lets peers **auto-discover** each other for a given service — the actual pseudowire signaling still happens via LDP (FEC 129) once the peer is discovered, but the "who do I need a PW to" step is now BGP-driven instead of manually paired.
**Why it matters (CCDE lens):** This is the same "auto-discovery vs. manually-paired" design trade-off that shows up again in VPLS (BGP-AD for VPLS) and is foundational to why EVPN (an all-BGP control plane) is considered the modern successor — manually specifying every PE pair for every service does not scale past a handful of sites, especially as sites are added/removed over time. BGP AD moves the "who's in this service" question into the same BGP infrastructure already used for L3VPN, giving one consistent auto-discovery mechanism across services instead of a separate manual-pairing exercise per PW.
**Real-world example:** A provider with hundreds of VPWS customers, each needing to add/remove sites over time, uses BGP AD so that adding a new PE to a service is a single local configuration change (advertise into the BGP AD address-family) rather than requiring coordinated manual `xconnect` updates on every existing PE in that service.

---

## 11. CCDE-Style Interview Q&A

**Q1. Two PEs — one IOS-XE, one IOS-XR — bring up a VPWS pseudowire session successfully, but you later discover the control word ended up disabled on both sides even though you explicitly configured it on the XE side. What happened?**
IOS-XE defaults to CW-on with autosense — it adjusts to match whatever the peer is doing. IOS-XR defaults to CW-off and stays off unless a `pw-class` explicitly enables it. If the XR side wasn't also explicitly configured, XE's autosense quietly matched XR's off state. You must explicitly configure the CW on **both** platforms to guarantee a specific outcome, not rely on XE's default-on setting alone.

**Q2. What's the practical difference between "Redundant VPWS" and what EVPN multihoming provides?**
Redundant VPWS protects against remote-PE failure with a primary/backup PW pair where only one is ever active — the same fundamental business requirement (protect against a PE failure, not just a link failure) that EVPN multihoming (all-active or single-active) solves with a more capable, BGP-driven, potentially multi-active mechanism. Redundant VPWS is the legacy, simpler predecessor to that same design goal.

**Q3. Why can't you get FAT-PW load balancing on a Manual (unsignaled) VPWS?**
FAT-PW's flow-label capability must be negotiated between peers via LDP signaling — Manual VPWS explicitly disables signaling (`signaling protocol none`), so there's no mechanism for the peers to agree the flow label is present and should be honored/discarded correctly. Manual VPWS and FAT-PW are mutually exclusive.

**Q4. When would you use MS-PW/pseudowire stitching instead of a single end-to-end signaled pseudowire?**
When you need to cross an administrative, vendor, or signaling-domain boundary — e.g., a separately-managed metro/access network meeting a national core — without forcing one flat, end-to-end LDP/label domain across both. The S-PE stitches two independently-signaled segments together, letting each domain evolve its internal design independently, similar in spirit to how Inter-AS L3VPN Option B avoids a single flat MPLS domain across an AS boundary.

**Q5. Why does BGP Auto-Discovery matter for VPWS scalability, if the actual PW is still signaled by LDP either way?**
BGP AD replaces the manual, per-PE-pair configuration of "who is my remote endpoint for this service" with automatic discovery via BGP — the operational bottleneck at scale isn't the label signaling itself, it's the O(n²) manual coordination of endpoint pairs as sites are added or removed. BGP AD (and later, EVPN's fully-BGP control plane) solves the discovery/scale problem, not the label-signaling problem.

**Q6. A VPWS PW status shows "up" on both PEs, but the customer reports no traffic is passing. The AC is confirmed up on both ends. What's a likely explanation specific to VPWS?**
A tag-manipulation (rewrite) misconfiguration — the PW control plane doesn't validate that VLAN tag handling matches customer expectations, only that the AC and PW segments are each individually up. This is a classic "control plane says healthy, data plane isn't" failure mode, alongside similar patterns like LDP-and-static-routes or LDP transport-address issues elsewhere in the MPLS stack.

---

## 12. Memory Map

```
VPWS Core
│
├── Basic Signaling & Setup (1)
│     tLDP-signaled (FEC 128) is the default/normal case
│     Control Word default differs by platform — real interop trap
│     └─ Local same-PE cross-connect needs NEITHER tLDP nor MPLS
│
├── Data-Plane Correctness (2)
│     Tag Manipulation ── "control plane up, forwarding still broken"
│     class of problem — same pattern as LDP/static-route and
│     transport-address issues in the LDP notes
│
├── Resiliency (3)
│     Redundant VPWS: protects against REMOTE PE failure
│     (Basic VPWS/IGP only protects against LINK failure)
│     └─ conceptual predecessor to EVPN multihoming
│
├── Interface Abstraction (4)
│     PW-interface style vs inline xconnect — same signaling result,
│     different ability to attach independent QoS/ACL/policy
│
├── Signaling Alternatives
│     Manual VPWS (5) ── no signaling, no AC-status propagation,
│     │                   same "escape hatch, doesn't scale" category
│     │                   as LDP Static Labels
│     └─ MUTUALLY EXCLUSIVE with FAT-PW (8), which REQUIRES signaling
│
├── Core Load-Balancing (8) + Ordering (6)
│     FAT-PW: adds a bottom flow-label so P routers can ECMP-hash
│     without inspecting L3/L4 — parallels BFD echo's "P routers
│     don't need to understand what's inside" design insight
│     Sequencing: safety net for order-sensitive payloads — becomes
│     MORE relevant precisely because FAT-PW spreads flows across
│     multiple core paths
│
├── Operational Visibility (7)
│     PW Logging ── same NOC-visibility pattern as ISIS
│     log-adjacency-changes all — easy to configure, easy to forget,
│     real gap in incident timelines if missing
│
└── Scale / Domain-Crossing Design (9, 10)
      MS-PW Stitching ── crossing SIGNALING/ADMIN domains
      (parallels Inter-AS L3VPN Option B's role for L3VPN)
      BGP AD ── crossing the SCALE problem of manual PE-pair config
      (direct conceptual predecessor to EVPN's all-BGP control plane)
```

**Recurring CCDE theme in VPWS:** legacy L2VPN mechanisms here (Redundant VPWS, BGP AD) each solve one narrow piece of a problem that **EVPN later solves holistically** with a single unified BGP control plane — understanding VPWS deeply is what makes the "why EVPN" argument concrete instead of buzzword-level in an interview.

---

## 13. CLI Cheat Sheet

| Purpose | Command |
|---|---|
| Local same-PE cross-connect (XE) | `connect <name> <int1> <int2>` or `l2vpn xconnect context <name>` with two local members |
| tLDP VPWS via pseudowire-class (XE, older style) | `pseudowire-class <name>` → `encapsulation mpls` / `control-word` → apply via `xconnect <peer> <pw-id> pw-class <name>` |
| tLDP VPWS via xconnect context (XE, newer style) | `l2vpn xconnect context <name>` → `member <int>` + `member <peer-ip> <pw-id> encapsulation mpls` |
| tLDP VPWS (XR) | `l2vpn xconnect group <name>` → `p2p <name>` → `interface <int>` + `neighbor <ip> pw-id <id> [pw-class <name>]` |
| Enable/force control word (XE) | `pseudowire-class` → `control-word` (or `template type pseudowire` → `control-word include`) |
| Enable control word (XR) | `l2vpn pw-class <name>` → `encapsulation mpls control-word` |
| Redundant VPWS backup peer (XE) | `xconnect <primary-ip> <id> pw-class <name>` → `backup peer <backup-ip> <id> pw-class <name>` |
| Preferred path via TE tunnel | `template type pseudowire <name>` → `preferred-path interface <tunnel>` (add `disable-fallback` to prevent reverting to LDP path) |
| Manual (unsignaled) PW | `mpls label range <start> <end> static <s2> <e2>` + `int pseudowireX` → `signaling protocol none` → `neighbor <ip> <id>` → `label <local> <remote>` |
| FAT-PW load balancing (XE) | `template type pseudowire` → `load-balance flow ip src-dst-ip` + `load-balance flow-label both` |
| FAT-PW load balancing (XR) | `l2vpn pw-class` → `encapsulation mpls` → `load-balancing flow-label both` |
| Verify PW/xconnect state | `show l2vpn xconnect` / `show l2vpn atom vc detail` (XE) / `show l2vpn xconnect detail` (XR) |
| Verify MPLS-TE tunnel used by PW | `show mpls traffic-eng tunnels` |
| Verify per-hop path via MPLS OAM | `ping mpls` / `traceroute mpls` |

---
*Source: CCIE-SP v5.1 Labs — VPWS section (11 labs): Basic VPWS, VPWS with Tag Manipulation, Redundant VPWS, Redundant VPWS (IOS-XR), VPWS with PW interfaces, Manual VPWS, VPWS with Sequencing, Pseudowire Logging, VPWS with FAT-PW, MS-PS (Pseudowire stitching), VPWS with BGP AD. Some sub-topics supplemented with standard MPLS L2VPN/pseudowire behavior (RFC 4447/6074) where specific lab page content was not directly retrievable.*
