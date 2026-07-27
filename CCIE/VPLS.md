# CCIE/CCDE — VPLS (Virtual Private LAN Service)
*Simple explanations, CCDE-level design depth, interview answers, CLI, and a concept memory map — covering all 16 VPLS labs.*

---

## 1. Basic VPLS with LDP (Manual Full Mesh)

**What:** VPLS extends VPWS from point-to-point to **multipoint** — a full mesh of pseudowires between every PE participating in a given VPLS instance (VFI), making the PEs behave collectively like one giant Ethernet switch (bridge-domain) across the WAN. Each PE learns remote MACs dynamically over the PW mesh exactly like a switch learns MACs on a trunk.
**Why it matters (CCDE lens):** The foundational scale problem of VPLS starts here: a full mesh of N PEs requires N(N-1)/2 pseudowires, manually configured on the base/LDP-signaled model — this is the exact motivation for every later section in this file (BGP AD, H-VPLS, BGP signaling). Also, because VPLS is a real bridge domain, all normal Ethernet loop-avoidance concerns apply — but you **cannot run STP across the WAN mesh** the normal way (it would block PW links unpredictably); split-horizon among PW-mesh members is VPLS's substitute loop-prevention mechanism (a PE never re-floods a frame received from one PW mesh member out to another PW mesh member — only out to local ACs).
**Real-world example:** A small VPLS with 4–5 PE sites is still practical to hand-configure as a full mesh; this doesn't scale past roughly a dozen or so sites without becoming an operational burden — which is precisely when BGP AD or H-VPLS enters the design conversation.

---

## 2. VPLS with LDP and BGP (BGP Auto-Discovery)

**What:** Replaces manually-configured PW mesh membership with **BGP L2VPN/VPLS auto-discovery** — PEs advertise a BGP route (auto-generated RD/RT/AGI in `<ASN>:<VPN-ID>` format) via a route-reflector; the tLDP pseudowire mesh is then built automatically to whichever PEs are discovered as members, using **FEC 129** (vs. FEC 128 for a manually-configured PW) to carry the extra source/target parameters needed for discovery.

**Why it matters (CCDE lens):** This is the direct scale-fix for Section 1's N² manual-mesh problem — adding a new PE to a service becomes "advertise into BGP," and the existing PEs auto-discover and build the new PW to it. Critically, **the actual data-plane signaling is still LDP (tLDP)** — BGP only solves discovery, not label distribution — so this is a genuinely separate concern from Section 3's BGP-signaling approach, a distinction interviewers specifically probe. The VPN-ID (from the BGP AGI) becomes the VCID and **must match exactly across all PEs**, even if different Route Targets are used to constrain which PEs join which sub-topology — a real config-consistency requirement that's easy to get subtly wrong at scale.
**Real-world example:** A provider growing a VPLS service from 5 to 50 sites over time uses BGP AD specifically so each new PE addition is a local, one-PE configuration change rather than a coordinated N-router mesh update — the same underlying motivation as VPWS's BGP AD (see VPWS notes, Section 10), just applied to a multipoint service.

---

## 3. VPLS with BGP Only (Full BGP Signaling, No LDP)

**What:** Goes one step further than Section 2 — BGP carries **both discovery AND the actual service label** (RFC 6624-style), so **no LDP/tLDP session is needed at all** between PEs for this service. The VE-ID under the VFI (`ve id <n>`) replaces the manually-paired tLDP FEC as the mechanism identifying each PE's position in the service.

**Platform config-philosophy difference — a genuine CCDE-relevant detail:**

| | IOS-XE | IOS-XR |
|---|---|---|
| Default signaling | LDP | — |
| To use BGP signaling | `suppress-signaling-protocol ldp` under the BGP neighbor (positive: "turn LDP off") | `signaling ldp disable` under the neighbor-group's l2vpn address-family (XR uses negative logic throughout: disable the OTHER protocol to select this one) |
| Per-PW feature templates (CW, etc.) | **Not available** when using pure BGP signaling — template-based knobs (e.g., `control-word exclude`) only work with `signaling ldp`, even when BGP AD is also in use | Same limitation |

**Why it matters (CCDE lens):** This is the real trade-off to articulate in a design review: full-BGP signaling eliminates an entire protocol (LDP) from the per-service control plane, consolidating everything onto BGP — architecturally cleaner, and a direct conceptual stepping stone toward EVPN (which goes all the way and eliminates LDP/tLDP for L2 services entirely, using BGP for MAC/discovery AND label distribution via the EVPN NLRI). The cost is losing LDP-only feature knobs (CW toggling via template, and other pseudowire-class-attached features) — a genuine feature-parity gap you must check against requirements before committing to a full-BGP-signaled design.
**Real-world example:** A greenfield SP deliberately choosing BGP-only signaling for VPLS specifically to reduce protocol surface area in the core (one less session type to secure, monitor, and troubleshoot per service) accepts the reduced per-PW feature flexibility as a worthwhile trade for operational simplicity — the same philosophy that ultimately leads a mature design toward EVPN.

---

## 4. Hub and Spoke VPLS

**What:** Uses VPLS's split-horizon group mechanism to force spoke PEs to communicate **only through a hub PE**, never directly with each other — even though the underlying PW mesh may physically still connect them — by placing spoke-facing PWs into a split-horizon group (frames received from one group member are never re-flooded to another member of the *same* group; only hub-facing PWs, in a different group, can re-flood freely).
**Why it matters (CCDE lens):** This is the L2 equivalent of an L3VPN hub-and-spoke design (see Intra-AS L3VPN hub topics) — the business driver is identical: force all inter-site traffic through a central point for policy enforcement (firewalling, inspection, centralized internet breakout) rather than allowing full any-to-any reachability. The mechanism differs (split-horizon groups at L2 vs. route-target import/export asymmetry at L3), but the design *intent* — deliberately reducing an any-to-any topology to a controlled hub-and-spoke — is the same pattern across both service types, a good "compare and contrast" interview answer.
**Real-world example:** A retail chain wants all store-to-store traffic to pass through a central data center firewall/inspection point rather than allowing direct any-to-any store connectivity — Hub-and-Spoke VPLS enforces this at the L2 service level without relying on the CE routers/switches to cooperate.

---

## 5. Tunnel L2 Protocols over VPLS

**What:** Controls whether L2 control protocols (CDP, LLDP, STP, etc.) arriving on a customer AC are **terminated at the PE** (consumed locally, not forwarded) or **tunneled transparently** across the VPLS core to the far-end CE, preserving the customer's illusion of a single flat L2 segment.
**Why it matters (CCDE lens):** This is a genuine per-protocol design decision, not an all-or-nothing switch — e.g., terminating CDP at the PE (so the PE and CE see each other directly, useful for the provider's own NMS/topology discovery) while tunneling LLDP through transparently (so the customer's own two CE devices can discover each other as if directly connected) requires per-protocol configuration, and getting the protocol scope wrong is a subtle but real customer-visible issue (e.g., a customer's LLDP-based topology tool suddenly "loses" visibility of a remote site after a VPLS migration).
**Real-world example:** A managed-CE service where the provider terminates CDP for their own NMS visibility into the customer's device, but tunnels the customer's proprietary L2 keepalive/STP traffic transparently, so the customer's own network behaves exactly as if it were one physical LAN.

---

## 6. Basic H-VPLS (Hierarchical VPLS)

**What:** Splits VPLS into two tiers — **n-PEs (network PEs)**, which form the full-mesh PW core exactly as in Section 1, and **u-PEs (user-facing PEs)**, which attach to the core via a **single spoke pseudowire** to one n-PE rather than joining the full mesh themselves.
**Why it matters (CCDE lens):** This is the SECOND scale answer to Section 1's N² mesh problem — a structurally different one from BGP AD (Section 2). BGP AD keeps a genuine full mesh but automates its maintenance; H-VPLS instead **architecturally reduces the mesh size** by only meshing the small number of n-PEs, letting the (potentially much larger) population of access-facing u-PEs each need only one spoke connection. These are complementary, not competing, techniques — a mature large-scale design commonly uses BOTH: BGP AD to automate the (now much smaller) n-PE mesh, and H-VPLS to keep the access-layer u-PE count from ever entering the mesh math at all.
**Real-world example:** A national SP with 500 small access sites and 8 regional core routers uses H-VPLS so the 500 access PEs each need exactly one spoke PW (to their nearest regional n-PE), while only the 8 n-PEs need a mesh among themselves — turning an infeasible 500² mesh problem into an 8² one.

---

## 7. H-VPLS with BGP

**What:** Combines Section 6's hierarchical topology with Section 2/3's BGP-based signaling — the n-PE core mesh is BGP-AD-discovered (or fully BGP-signaled), while u-PEs still connect via a single spoke PW to their n-PE.
**Why it matters (CCDE lens):** Demonstrates that these design axes (hierarchy vs. flat, and LDP-signaled vs. BGP-signaled/discovered) are **orthogonal** — you choose independently along each axis based on the specific scale/operational problem you're solving (mesh size vs. signaling protocol consolidation), not as a single bundled decision. A CCDE-level design conversation should treat "how do we scale the topology" and "how do we scale/simplify the signaling" as two separate questions with independently chosen answers.

---

## 8. H-VPLS with QinQ

**What:** Uses 802.1Q tunneling (QinQ, an outer "S-tag" wrapping the customer's own inner C-tag) at the u-PE access edge so that the u-PE can aggregate **many distinct customer VLANs** into a single spoke pseudowire to the n-PE, while keeping each customer's VLAN space fully private/untouched.
**Why it matters (CCDE lens):** This solves a real capacity/scale problem at the access edge: without QinQ, each customer VLAN needing distinct treatment would need its own spoke PW (or its own bridge-domain mapping) — QinQ lets a single physical spoke PW carry an entire aggregation switch's worth of distinct customer services, with the outer tag providing the SP's own multiplexing/demux key independent of whatever VLAN IDs the customers happen to have chosen (directly reusing the same "VLAN normalization" value proposition as VPWS Tag Manipulation, just applied at multipoint/aggregation scale instead of point-to-point).
**Real-world example:** A metro-Ethernet aggregation switch (u-PE) serving 40 different small-business tenants in one building uses QinQ so all 40 tenants' traffic rides one spoke PW to the n-PE, each isolated by its own inner customer VLAN, without needing 40 separate spoke pseudowires.

---

## 9. H-VPLS with Redundancy

**What:** Dual-homes a u-PE to **two different n-PEs** via two spoke PWs (primary/backup, similar in spirit to VPWS's Redundant VPWS) so a single n-PE failure doesn't isolate every u-PE/customer behind it.
**Why it matters (CCDE lens):** In a hierarchical design, the n-PEs become genuine single points of failure for every u-PE homed to them — the more aggressively you scale via hierarchy (Section 6), the more consequential each n-PE failure becomes (blast radius grows with the number of u-PEs served). This is the direct resiliency answer to that concentration risk — the same "hierarchy improves scale but concentrates failure impact, so you must explicitly re-add redundancy at the concentration point" pattern that shows up broadly in network design (route reflectors, area border routers, etc.), not unique to VPLS.
**Real-world example:** Losing an n-PE in a non-redundant H-VPLS design could isolate dozens of access sites simultaneously — H-VPLS with Redundancy is standard practice for any n-PE tier serving a meaningful number of downstream u-PEs, precisely to bound that blast radius.

---

## 10. VPLS with Routing (IRB)

**What:** Attaches an **Integrated Routing and Bridging (IRB)** interface (a BVI/routed interface) to the VPLS bridge-domain, giving the PE itself an IP presence directly on the customer's L2 segment — allowing the PE to act as a routed gateway (e.g., default gateway for the VPLS-attached subnet) rather than being purely a transparent L2 switch.
**Why it matters (CCDE lens):** This is the point where a "pure" L2 VPLS service starts blending into L3 territory — a real design fork: do you want the PE to remain a transparent bridge (customer manages their own routing/gateway entirely), or do you want the SP to offer a routed gateway service directly on the VPLS segment (simplifying the customer's design, but coupling the SP more tightly into the customer's L3 architecture)? This is conceptually the direct predecessor to EVPN's IRB/anycast-gateway model, which does the same thing but with active-active gateway redundancy across multiple PEs rather than a single IRB instance.
**Real-world example:** An SP offering a managed "virtual router" service where the customer's VPLS segment's default gateway lives on the PE itself, simplifying the customer's edge design (no CE routing needed at all — just L2 switches) at the cost of the SP now being responsible for that L3 function.

---

## 11. VPLS MAC Protection (MAC Limiting / Security)

**What:** Bounds the number of MAC addresses a PE will learn per AC/service-instance or per bridge-domain, and defines an action (drop new MACs, shut the interface, etc.) when the limit is exceeded — the WAN-service equivalent of switchport port-security.
**Why it matters (CCDE lens):** In a shared multipoint service, one customer site with a MAC-flooding condition (malicious or accidental — e.g., a misbehaving device, a bridging loop on the customer side, or a deliberate MAC-flood attack) can exhaust the PE's MAC table capacity, degrading the service for **every other site** in that VPLS instance, or even other unrelated services sharing PE resources. MAC limiting is a blast-radius containment control, directly analogous to CoPP/LPTS protecting the control plane (see the Security notes) but protecting the **data-plane MAC table resource** instead.
**Real-world example:** A single compromised or misconfigured customer device causing a MAC flood on one VPLS-attached site is contained to that site (dropped once its own limit is hit) rather than exhausting the PE's shared MAC table and degrading service for every other tenant sharing that PE.

---

## 12. Basic E-TREE

**What:** Implements the MEF E-Tree service model within VPLS — designates ACs as either **Root** (can communicate with everyone) or **Leaf** (can only communicate with Root ports, never with other Leaf ports) — essentially a per-AC-granular split-horizon rule, finer-grained than the whole-PW-group split-horizon used in Hub-and-Spoke VPLS (Section 4).
**Why it matters (CCDE lens):** E-Tree solves the SAME hub-and-spoke business requirement as Section 4, but at **AC granularity within a single PE** rather than at PW-group granularity across the mesh — useful when the hub/leaf distinction needs to be enforced even between two leaf sites that happen to be homed to the *same* PE (Hub-and-Spoke's PW-group split-horizon alone wouldn't stop two local ACs on the same PE from reaching each other directly). Knowing which mechanism (Hub-and-Spoke's group split-horizon vs. E-Tree's per-AC root/leaf) applies at which granularity is the specific interview-relevant distinction.
**Real-world example:** A managed multi-tenant building service where every tenant (leaf) needs internet/service access via a single ISP handoff (root), but tenants must never reach each other directly, even two tenants who happen to connect through the very same aggregation PE — E-Tree enforces this at the AC level regardless of PE topology.

---

## 13. VPLS with LDP/BGP-AD and XRv RR

**What:** Same LDP-signaled + BGP-auto-discovered model as Section 2, but specifically validated/exercised with an IOS-XR router reflector — reinforcing that the RR's role here is purely BGP-control-plane (reflecting the L2VPN/VPLS address-family NLRI); the RR itself never needs to participate in the data-plane PW mesh at all.
**Why it matters (CCDE lens):** This is a good platform-flexibility data point for a design review — the RR function for VPLS BGP AD can be centralized on a platform/location chosen purely for BGP-RR scale/reliability characteristics, completely decoupled from where the actual VPLS data-plane PEs live. The RR is a pure control-plane function here, same as it is for any other BGP address-family.

---

## 14. VPLS with Storm Control

**What:** Rate-limits broadcast/multicast/unknown-unicast (BUM) traffic per AC or per bridge-domain, dropping excess above a configured threshold — protecting the shared multipoint service from broadcast storms.
**Why it matters (CCDE lens):** VPLS is fundamentally a shared **broadcast domain** across potentially many sites and a WAN core — a broadcast storm at any single site (loop, misconfigured device, malicious traffic) doesn't just affect that site, it **replicates across every PE and AC** in the service (that's what a bridge domain does by design), turning a single-site problem into a service-wide outage. Storm control is essential, not optional, damage-containment for any real production VPLS — directly complementary to MAC Protection (Section 11): storm control bounds the *rate* of BUM traffic, MAC limiting bounds the *table growth* from unicast learning; both address different blast-radius risks inherent to sharing one broadcast domain across many independent customer sites.
**Real-world example:** A customer's own switch develops an internal bridging loop — without storm control, the resulting broadcast storm floods across the entire VPLS core to every other site in that service; with storm control, the storm is rate-limited at ingress to that one AC, containing the damage to the originating site's own link.

---

## 15. CCDE-Style Interview Q&A

**Q1. What's the practical difference between "VPLS with LDP and BGP" (BGP AD) and "VPLS with BGP Only," if both use BGP?**
BGP AD (LDP+BGP) uses BGP only for PE discovery — the actual pseudowire label signaling is still done by tLDP (FEC 129). BGP-only signaling eliminates tLDP entirely, with BGP carrying both discovery and the service label. The practical cost of going full-BGP is losing LDP-only per-PW feature knobs (like toggling the control word via a template) — a real feature-parity trade-off to check before committing to a design.

**Q2. You have 500 access sites and 8 core routers. Would you use BGP Auto-Discovery or H-VPLS to make this scale, and why not both?**
Both, typically — they solve different scale problems. BGP AD automates maintaining a full mesh but doesn't change the size of that mesh. H-VPLS architecturally shrinks the mesh to just the n-PE tier (8 routers) while the 500 access u-PEs each need only one spoke PW. Using BGP AD to automate the now-small 8-router n-PE mesh, combined with H-VPLS to keep the 500 u-PEs out of the mesh math entirely, is the standard large-scale pattern.

**Q3. Why does hierarchy (H-VPLS) increase the importance of n-PE redundancy specifically?**
Hierarchy concentrates failure impact — every u-PE homed to a given n-PE loses connectivity if that single n-PE fails, and the blast radius grows with however many u-PEs are homed to it. This is the general "hierarchy improves scale but concentrates failure impact at the aggregation point" pattern, which is why H-VPLS with Redundancy (dual-homing u-PEs to two n-PEs) becomes standard practice rather than optional as the design scales.

**Q4. What's the difference between Hub-and-Spoke VPLS's split-horizon groups and E-Tree's root/leaf model?**
Hub-and-Spoke split-horizon operates at PW-group granularity — it prevents re-flooding between members of the same group, typically across the mesh. E-Tree operates at individual AC granularity and can enforce root/leaf separation even between two leaf sites homed to the *same* PE, which group-based split-horizon alone cannot do. They solve the same underlying business requirement (controlled hub-and-spoke reachability) at different granularities.

**Q5. Why is storm control considered essential rather than optional for a production VPLS service?**
Because VPLS is a genuine shared broadcast domain across every PE and AC in the service — a broadcast storm originating at any single site replicates to every other site by design (that's what bridging does), turning a single-site fault into a service-wide outage. Storm control contains that blast radius at the point of ingress, complementary to MAC limiting, which addresses the separate risk of MAC-table exhaustion from unicast learning.

**Q6. What's the conceptual link between "VPLS with Routing" (IRB) and EVPN?**
VPLS+IRB is the direct predecessor to EVPN's integrated routing/anycast-gateway model — both give the PE(s) a routed L3 presence directly on the L2 service. The key evolution in EVPN is that the gateway function can be active-active across multiple PEs simultaneously (anycast gateway), whereas classic VPLS+IRB typically has a single active routing instance for that segment.

---

## 16. Memory Map

```
VPLS Core
│
├── Foundational Model (1)
│     Full-mesh PW = O(N²) scale ceiling
│     Split-horizon (whole-PW-group) = VPLS's native loop prevention
│     └─ THE central scale problem every other section responds to
│
├── Signaling-Axis Solutions (2, 3)
│     BGP AD (2): BGP for DISCOVERY only, tLDP still signals labels
│     BGP-only (3): BGP for discovery AND labels — no LDP at all
│     └─ direct stepping stone toward EVPN's fully-BGP control plane
│           (loses LDP-only per-PW feature templates as a trade-off)
│
├── Topology-Axis Solution (6, 7, 8, 9)
│     H-VPLS: architecturally SHRINKS the mesh (only n-PEs mesh)
│     — ORTHOGONAL to the signaling axis (2/3); can combine (7)
│     QinQ (8): lets ONE spoke PW carry many customer VLANs —
│           access-scale version of VPWS's tag-manipulation idea
│     Redundancy (9): DIRECT CONSEQUENCE of H-VPLS's concentration
│           risk — hierarchy demands redundancy at the n-PE tier
│
├── Controlled-Reachability Variants (4, 12)
│     Hub-and-Spoke (4): PW-GROUP granularity split-horizon
│     E-Tree (12): PER-AC granularity root/leaf — finer-grained
│           version of the SAME underlying business requirement
│
├── Protocol Transparency (5)
│     Tunnel L2 Protocols: per-PROTOCOL choice (terminate vs tunnel)
│     — not all-or-nothing; real customer-visible design decision
│
├── Blast-Radius Containment (11, 14)
│     MAC Protection (11): bounds MAC-TABLE growth (unicast risk)
│     Storm Control (14): bounds BUM traffic RATE (broadcast risk)
│     └─ both exist BECAUSE VPLS is one shared broadcast domain —
│           a single-site fault otherwise becomes service-wide
│
├── L2→L3 Bridge (10)
│     VPLS + IRB: PE gets a routed presence on the L2 segment
│     └─ direct predecessor to EVPN's anycast-gateway/IRB model
│
└── RR Placement (13)
      Confirms BGP-AD RR role is PURE control-plane — decoupled
      from where actual data-plane PEs physically live
```

**Recurring CCDE theme:** VPLS's evolution across these 16 labs — manual mesh → BGP-automated discovery → BGP-only signaling → hierarchical topology → IRB-based routing — is literally the historical design trajectory that leads to EVPN. Understanding *why* each VPLS mechanism exists (which specific scale or resiliency problem it solves) is what makes "just use EVPN" a defensible design recommendation instead of a buzzword in an interview.

---

## 17. CLI Cheat Sheet

| Purpose | Command |
|---|---|
| Basic VFI + bridge-domain (manual mesh, XE) | `l2vpn vfi context <name>` → `vpn id <id>` → `neighbor <ip> encapsulation mpls` → `bridge-domain <id>` → `member vfi <name>` + `member <int> service-instance <n>` |
| BGP AD (LDP-signaled) under VFI (XE) | `l2vpn vfi context <name>` → `autodiscovery bgp signaling ldp` |
| BGP AD + BGP-signaled (no LDP) under VFI (XE) | `l2vpn vfi context <name>` → `autodiscovery bgp signaling bgp` → `ve id <n>` |
| Enable BGP L2VPN VPLS AF, RR client | `router bgp <asn>` → `address-family l2vpn vpls` → `neighbor <ip> activate` [+ `route-reflector-client`] |
| Use BGP for signaling instead of LDP (XE) | `neighbor <ip> suppress-signaling-protocol ldp` |
| Use BGP for signaling instead of LDP (XR) | `neighbor-group` → `address-family l2vpn vpls-vpws` → `signalling ldp disable` |
| Use LDP for signaling instead of BGP (XR) | `signalling bgp disable` |
| Split-horizon group (Hub-and-Spoke) | `l2vpn vfi context` → `member ... split-horizon group <n>` (naming varies by platform) |
| H-VPLS spoke PW from u-PE to n-PE | Standard `xconnect`/VFI-neighbor statement to the single n-PE — u-PE does NOT join the full mesh |
| QinQ on u-PE access | `service instance <n> eth` → `encapsulation dot1q <c-vlan> second-dot1q <s-vlan>` |
| Tunnel/terminate L2 protocols | `l2protocol tunnel [cdp\|lldp\|stp...]` / `l2protocol forward` under service instance |
| IRB / bridge-domain routed interface | `interface BDI<id>` bound to the matching `bridge-domain <id>` |
| MAC limiting | `mac limit maximum <n> action <shutdown\|flood\|no-flood>` under bridge-domain/service-instance |
| Storm control | `storm-control [broadcast\|multicast\|unicast] level <pct>` under the AC/service-instance |
| E-Tree root/leaf designation | `service instance ... e-tree leaf` (or `root`, root is often default) |
| Verify VPLS PW/VFI state | `show l2vpn vfi` / `show l2vpn atom vc` (XE) / `show l2vpn xconnect` (XR) |
| Verify auto-discovered peers | `show bgp l2vpn vpls` / VFI-specific autodiscovery peer show command |
| Verify MAC learning | `show bridge-domain <id> mac` / `show l2vpn forwarding mac` |

---
*Source: CCIE-SP v5.1 Labs — VPLS section (16 labs): Basic VPLS with LDP, VPLS with LDP and BGP, VPLS with BGP only, Hub and Spoke VPLS, Tunnel L2 Protocols over VPLS, Basic H-VPLS, H-VPLS with BGP, H-VPLS with QinQ, H-VPLS with Redundancy, VPLS with Routing, VPLS MAC Protection, Basic E-TREE, VPLS with LDP/BGP-AD and XRv RR, VPLS with BGP and XRv RR, VPLS with Storm Control. Some sub-topics supplemented with standard VPLS/H-VPLS design behavior (RFC 4761/4762) where specific lab page content was not directly retrievable.*
