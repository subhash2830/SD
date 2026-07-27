# Segment Routing (SR) — CCDE Notes

## 1. Subtopics

### 1.1 SR-MPLS Fundamentals — Prefix-SID and Adjacency-SID
**What:** Segment Routing encodes a source-routed path as an ordered stack of segments (labels), eliminating the need for LDP or RSVP-TE signaling. A Prefix-SID is a globally significant label representing shortest-path-to-a-node (advertised via IGP), while an Adjacency-SID is a locally significant label representing a specific link/adjacency.

**Why it matters (CCDE lens):** This is the core paradigm shift CCDE tests conceptually: with LDP, labels are locally significant and distributed hop-by-hop with no path control; with SR, the ingress node can steer traffic along an arbitrary explicit path just by stacking the right sequence of Prefix-SIDs/Adjacency-SIDs — no separate signaling protocol, no per-LSP state maintained hop-by-hop. Understanding "the label stack IS the path" is the single most important conceptual leap from MPLS-TE/LDP thinking.

**Real-world example:** A provider migrating off LDP+RSVP-TE to SR-MPLS eliminates two entire signaling protocols and their associated control-plane state/scaling limits, replacing them with IGP-only label distribution — a major operational simplification frequently cited as SR's primary business case.

**CLI (IOS-XR):**
```
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
interface Loopback0
 address-family ipv4 unicast
  prefix-sid absolute 16001
```

### 1.2 SRGB (Segment Routing Global Block)
**What:** The SRGB is the reserved label range (e.g., 16000–23999) on each node from which Prefix-SID index values are mapped to actual MPLS labels; the same SRGB range (or careful offset alignment) across all nodes in a domain simplifies operations by making a node's Prefix-SID index map to a predictable, consistent label domain-wide.

**Why it matters (CCDE lens):** Misaligned SRGBs across nodes (different starting ranges/sizes) is a classic real-world SR outage cause — if node A expects index 5 to map to label 16005 but node B's SRGB starts at 20000 so index 5 maps to 20005, cross-domain label-stack computations by a centralized controller/PCE (which assumes a uniform SRGB) silently compute wrong label stacks. CCDE expects candidates to mandate a consistent SRGB across the entire SR domain as a hard design requirement, not an optional nicety.

**Real-world example:** During a phased SR migration, half the newly upgraded routers are provisioned with the standard 16000–23999 SRGB while a few edge routers retain a vendor-default 100000+ range — traffic-engineered SR policies computed centrally (assuming uniform SRGB) silently install wrong labels on the mismatched nodes, causing traffic blackholes only on paths touching those routers.

**CLI (IOS-XR):**
```
router isis CORE
 address-family ipv4 unicast
  segment-routing global-block 16000 23999
```

### 1.3 SR-TE / Segment Routing Policy
**What:** An SR Policy is a set of one or more explicit label-stack paths (candidate paths) associated with a headend, color, and endpoint — functionally replacing RSVP-TE tunnels but without per-hop signaled state; paths can be computed locally, via PCE, or statically defined as an explicit SID list.

**Why it matters (CCDE lens):** SR Policy's color+endpoint tuple is the mechanism for intent-based steering (e.g., "low-latency" color vs "high-bandwidth" color to the same endpoint), and this is where CCDE tests understanding of the full automated-steering chain: BGP color extended-community on a service route → matches an SR Policy with that color on the headend → traffic is automatically steered onto that policy's path — a design pattern (Automated Steering) that has no RSVP-TE equivalent this seamless.

**Real-world example:** A provider defines a "gold" color SR Policy computed for minimum latency and a "bronze" color policy for minimum cost, both terminating at the same PE; customer-facing BGP routes tagged with the gold color extended-community are automatically steered onto the low-latency path without any manual tunnel-selection configuration at the ingress PE.

**CLI (IOS-XR):**
```
segment-routing
 traffic-eng
  policy GOLD-TO-PE5
   color 100 end-point ipv4 10.0.0.5
   candidate-paths
    preference 100
     explicit segment-list LOW-LATENCY-PATH
```

### 1.4 TI-LFA (Topology-Independent Loop-Free Alternate)
**What:** TI-LFA is SR's native fast-reroute mechanism — it pre-computes, via the IGP itself (no separate RSVP signaling), a backup path using a SID stack that guarantees loop-free sub-50ms protection against any single link or node failure, converging to the exact post-convergence path the IGP would have chosen anyway.

**Why it matters (CCDE lens):** This is a major advantage over legacy LFA (which can fail to find a loop-free alternate in many real topologies) and over MPLS-TE FRR (which requires explicit backup-tunnel provisioning) — TI-LFA works automatically wherever the topology mathematically allows a loop-free repair path, and critically it converges to the SAME path the network would settle on after full reconvergence, avoiding the "backup path is different from final path" transient behavior some legacy FRR designs exhibit. CCDE expects you to know TI-LFA's protection coverage isn't 100% in all topologies (still bounded by topology connectivity), so validate actual coverage, don't assume it.

**Real-world example:** A ring-and-spoke metro topology previously reliant on manually provisioned MPLS-TE FRR backup tunnels at every node migrates to SR with TI-LFA and gets equivalent sub-50ms protection with zero manual backup-tunnel configuration — but the design team must still validate TI-LFA coverage percentage per node, since a handful of low-connectivity edge nodes still lack a loop-free alternate and need supplemental protection.

**CLI (IOS-XR):**
```
router isis CORE
 interface Gi0/0/0
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
```

### 1.5 Flexible Algorithm (Flex-Algo)
**What:** Flex-Algo lets an operator define custom IGP path-computation algorithms (beyond plain SPF) — e.g., "min-delay SPF" or "SPF excluding links tagged X" — and advertise per-algorithm Prefix-SIDs, so a single IGP can compute and encode multiple distinct topologies/constraints simultaneously without needing a separate signaling protocol or centralized PCE for common constrained-path cases.

**Why it matters (CCDE lens):** Flex-Algo pushes a meaningful subset of what previously required SR-PCE/centralized TE computation down into distributed IGP computation — CCDE tests whether you recognize the tradeoff: Flex-Algo handles broad, static-ish constraint classes (low-latency plane, disjoint plane) very efficiently and with no controller dependency, but doesn't replace PCE/centralized SR-TE for truly dynamic, per-flow, bandwidth-aware policies that need global optimization and real-time recomputation.

**Real-world example:** A provider defines Flex-Algo 128 as "minimize IGP delay metric" across their whole core and simply advertises a second Prefix-SID per node for that algorithm — any service needing a low-latency path just uses that SID, with zero PCE/controller involvement, versus needing a full SR-PCE deployment if the same outcome were attempted via classic SR-TE.

**CLI (IOS-XR):**
```
router isis CORE
 flex-algo 128
  metric-type delay
  affinity exclude-any color HIGH-LATENCY
interface Loopback0
 address-family ipv4 unicast
  prefix-sid algorithm 128 absolute 17001
```

### 1.6 PCE / PCEP for Centralized SR-TE
**What:** A Path Computation Element (PCE, e.g., Cisco SR-PCE) can compute and delegate SR Policy candidate paths on behalf of headends via PCEP, providing centralized, network-wide-optimized path computation (bandwidth-aware, disjointness-aware across multiple policies) that distributed per-headend CSPF cannot achieve alone.

**Why it matters (CCDE lens):** CCDE tests knowing exactly when centralization is required vs unnecessary complexity — a single headend computing its own SR Policy locally has no visibility into what OTHER headends are simultaneously reserving, so true global bandwidth optimization or multi-policy disjointness (e.g., "these two policies must never share a link, even under failure") mathematically requires a centralized view; local/static SID-list policies are fine for simpler steering intent that doesn't need cross-policy coordination.

**Real-world example:** Two independent, unrelated headends each locally compute a "shortest available path" SR Policy for two customers who contractually require physically disjoint paths for resiliency — without a PCE providing global visibility, both headends can independently and unknowingly select paths that share a common fiber segment, violating the disjointness SLA; only a PCE with full-topology, multi-policy visibility can guarantee true disjointness.

**CLI (IOS-XR, headend delegating to PCE):**
```
segment-routing
 traffic-eng
  pcc
   pce address ipv4 10.0.0.99
   source-address ipv4 10.0.0.1
  policy DISJOINT-A
   candidate-paths
    preference 100
     dynamic
      pcep
```

### 1.7 SRv6 (Segment Routing over IPv6) — Awareness Level
**What:** SRv6 encodes segments as IPv6 addresses (SIDs) directly in the IPv6 header (via the Segment Routing Header), eliminating the MPLS label plane entirely — a node processes a SID as a native IPv6 destination with an associated function (End, End.X, End.DT4, etc.).

**Why it matters (CCDE lens):** SRv6 is the strategic direction for networks wanting to unify the transport and service layers under a single IPv6-native data plane (no separate MPLS forwarding plane to operate/troubleshoot), but CCDE expects awareness of real deployment friction: larger header overhead (128-bit SIDs vs 20-bit MPLS labels) impacting MTU/throughput at scale, and hardware/ASIC support for SRv6 SID processing at line rate being less universally mature than long-established SR-MPLS forwarding — so a "should we deploy SRv6 today" design answer must weigh strategic alignment against current hardware/ecosystem maturity, not just feature availability.

**Real-world example:** A greenfield provider building a new core with modern SRv6-capable ASICs chooses SRv6 to unify IPv6 transport and VPN services under one data plane from day one, while an established SR-MPLS provider with a large installed base of older linecards defers SRv6 migration until hardware refresh cycles naturally replace the non-SRv6-capable equipment.

**CLI (IOS-XR, conceptual):**
```
segment-routing
 srv6
  locator MAIN
   prefix 2001:db8:1::/48
router isis CORE
 address-family ipv6 unicast
  segment-routing srv6
   locator MAIN
```

### 1.8 Anycast-SID
**What:** An Anycast-SID is a Prefix-SID advertised identically by two or more nodes (typically sharing an anycast loopback address) — the IGP naturally routes traffic to the topologically nearest node advertising that SID, giving built-in load-balancing/redundancy without any explicit protocol beyond normal IGP ECMP behavior.

**Why it matters (CCDE lens):** Anycast-SID is the SR-native replacement for a lot of what used to require manual pseudo-node or VRRP-style tricks purely for path-selection redundancy at a logical "any-of-these-N-nodes" hop in a SID list — e.g., "exit via any of these three border routers" as a single segment. CCDE tests whether you recognize when a design needs a specific node (regular Prefix-SID) versus when it needs "any equally-valid node in this set" (Anycast-SID) — using the wrong one either over-constrains the path unnecessarily or removes desired determinism.

**Real-world example:** A design wants an SR Policy's SID list to force egress via "any of our three peering routers to Provider X" rather than one specific router — an Anycast-SID shared by all three lets the ingress simply include that one anycast segment, and the IGP/ECMP naturally load-balances and reroutes across the surviving peering routers if one fails, with no SID-list recomputation needed.

**CLI (IOS-XR):**
```
interface Loopback10
 ipv4 address 10.255.255.10 255.255.255.255
router isis CORE
 interface Loopback10
  address-family ipv4 unicast
   prefix-sid absolute 16100 n-flag-clear
```

### 1.9 Binding-SID
**What:** A Binding-SID is a locally significant SID that maps to an entire pre-defined SID list (i.e., "this one label represents that whole other explicit path") — used primarily to compress long SID stacks at domain/area boundaries and to enable SR Policy stitching across separate IGP domains or between an SR-TE domain and a legacy MPLS-TE/LDP domain.

**Why it matters (CCDE lens):** Binding-SID is the mechanism that solves SR's "how deep can the SID stack realistically get" hardware-imposed limit (most ASICs only push a bounded number of MPLS labels efficiently) — CCDE tests whether you know to insert a Binding-SID at domain boundaries in a multi-area or multi-AS SR design so the ingress-node SID stack stays shallow (referencing the Binding-SID rather than every hop of a long inter-domain path explicitly), and that this is also the standard technique for interconnecting an SR island with a still-MPLS-TE-only remote domain during a phased migration.

**Real-world example:** A path spanning three IGP areas would require an explicit SID list of 12+ labels end-to-end, exceeding the ingress ASIC's efficient push depth — instead, each area boundary node advertises a Binding-SID representing "the rest of the path through this area," collapsing the ingress SID stack to just 3 labels (one Binding-SID per area) regardless of the underlying path's true hop count.

**CLI (IOS-XR):**
```
segment-routing
 traffic-eng
  policy AREA2-TRANSIT
   binding-sid mpls 900001
   color 50 end-point ipv4 10.0.2.1
```

### 1.10 BGP Egress Peer Engineering (EPE) with Peer-SIDs
**What:** EPE extends SR to the internet-peering edge — a border router advertises a Peer-SID (Peer-Node-SID or Peer-Adjacency-SID) for each specific BGP peering session/interface, letting an ingress SR Policy explicitly steer traffic out a chosen egress peer/link rather than accepting whatever BGP best-path would normally select.

**Why it matters (CCDE lens):** This is SR's answer to a problem operators previously solved clumsily with per-peer route-maps/communities and manual next-hop manipulation — CCDE tests whether you understand EPE lets a centralized controller (via BGP-LS reporting Peer-SIDs back to a PCE) make network-wide egress-peer traffic-engineering decisions (e.g., steer specific high-value traffic away from a congested transit link to a settlement-free peer) at a granularity normal BGP best-path selection can't express per-flow.

**Real-world example:** A content provider under BGP best-path would send all traffic to a particular eyeball network out Transit Provider A, but Transit Provider A's link is congested — with EPE, a centralized controller computes an SR Policy steering that specific traffic out a Peer-SID for a settlement-free peering session instead, without touching global BGP policy for all other traffic.

**CLI (IOS-XR):**
```
router bgp 65000
 neighbor 203.0.113.10
  remote-as 65999
  address-family ipv4 unicast
   next-hop-self
  egress-engineering
```

### 1.11 SR Mapping Server (SRMS) — LDP/SR Interworking
**What:** During a phased SR migration, nodes not yet running SR still need Prefix-SID mapping for prefixes that ARE SR-enabled elsewhere in the domain — the SR Mapping Server advertises a Prefix-to-SID mapping via IGP that non-SR nodes can use purely for label-stitching purposes (LDP-to-SR and SR-to-LDP boundary translation) without those nodes running SR themselves.

**Why it matters (CCDE lens):** Greenfield SR designs rarely need this, but almost every real brownfield migration does — CCDE tests whether candidates plan the LDP/SR coexistence and boundary-stitching strategy explicitly (mapping server placement, redundancy, which nodes are the LDP/SR boundary) rather than assuming a "flag day" cutover, which is operationally unrealistic for any network of meaningful size.

**Real-world example:** A carrier migrates SR core-by-core over 18 months; SRMS lets already-migrated SR nodes and still-on-LDP nodes exchange labeled traffic correctly at the boundary throughout the transition, avoiding a disruptive single flag-day cutover across the entire footprint.

**CLI (IOS-XR):**
```
segment-routing
 mapping-server
  prefix-sid-map
   address-family ipv4
    10.0.0.0/24 range 256 16000
```

### 1.12 Micro-loop Avoidance
**What:** During IGP convergence after a topology change, different routers in the domain update their forwarding tables at slightly different times, which can create transient micro-loops (traffic looping for milliseconds to a few seconds until all routers finish converging) — SR-based micro-loop avoidance uses temporary SID-stack-based detours during the convergence window itself to guarantee loop-free forwarding even mid-convergence, not just post-convergence (TI-LFA) or pre-convergence (steady state).

**Why it matters (CCDE lens):** This closes a gap CCDE candidates often overlook: TI-LFA protects against the FIRST failure instant and the network eventually reconverges to a correct final state, but the convergence process itself can transiently loop — for latency/loss-sensitive services (voice, financial trading), that transient window matters. CCDE expects awareness that "we have TI-LFA" is not automatically "we have zero loss during any topology change," and that micro-loop avoidance is the mechanism specifically closing that remaining gap.

**Real-world example:** A financial-trading network with strict packet-loss SLAs discovers that even with TI-LFA deployed, brief microloops during large-scale IGP reconvergence events (e.g., a whole node reload) still cause measurable loss — enabling SR micro-loop avoidance closes this residual gap that TI-LFA alone didn't cover.

**CLI (IOS-XR):**
```
router isis CORE
 address-family ipv4 unicast
  microloop avoidance protected
  microloop avoidance segment-routing
```

### 1.13 Multi-Domain / Inter-AS SR-TE via Binding-SID Stitching
**What:** For an SR Policy that must cross multiple IGP domains or AS boundaries where no single controller/CSPF has full end-to-end visibility, each domain independently computes its own local segment (often expressed as a Binding-SID) and the end-to-end path is built by stitching these domain-local segments together — the ingress only needs to know the sequence of domain-boundary Binding-SIDs, not every hop in every remote domain.

**Why it matters (CCDE lens):** This is the direct evolution of the Inter-AS L3VPN Option A/B/C trust-model discussion applied to the SR-TE control plane — CCDE expects you to map the same trust-boundary reasoning (how much visibility/coordination are two administrative domains willing to share) onto SR multi-domain design: a single global PCE spanning two independent companies' domains is rarely realistic (same trust argument as Option C), so Binding-SID-based per-domain stitching, coordinated only at the boundary, is usually the practical inter-provider answer.

**Real-world example:** Two merged-but-still-operationally-separate regional networks (same overall company, different historical SR-PCE domains) stitch end-to-end low-latency SR Policies across their boundary using a Binding-SID advertised by the boundary router, avoiding the need for a single unified PCE with full visibility into both domains' internal topology during the transition period.

**CLI (IOS-XR, boundary node advertising Binding-SID for a remote-domain segment):**
```
segment-routing
 traffic-eng
  policy REMOTE-DOMAIN-SEGMENT
   binding-sid mpls 900050
   color 60 end-point ipv4 10.9.0.1
   candidate-paths
    preference 100
     dynamic
      pcep
```

---

## 2. Interview Q&A

**Q1: What is the fundamental architectural difference between SR-MPLS and LDP/RSVP-TE that CCDE cares about most?**
A: SR-MPLS encodes the entire forwarding path as a stack of segments computed at the ingress node — no per-hop signaling protocol or per-LSP state maintained hop-by-hop in the network, unlike LDP (locally significant labels, no path control) or RSVP-TE (explicit signaling with hop-by-hop soft state). "The label stack IS the path" is the core paradigm shift.

**Q2: Why must SRGB be consistent across an SR domain, and what specifically breaks if it isn't?**
A: A Prefix-SID is advertised as an index, and each node maps that index into an actual label using its own SRGB range — if SRGBs differ across nodes, the same index maps to different labels on different nodes. A centralized PCE/controller computing a label stack under the assumption of a uniform SRGB will compute wrong labels for mismatched nodes, causing silent traffic blackholing on paths touching them.

**Q3: How does Automated Steering via SR Policy color work end-to-end?**
A: A BGP service route is tagged with a color extended-community; the headend matches that color (plus the route's next-hop as endpoint) against a locally configured SR Policy with the same color+endpoint tuple, and automatically steers matching traffic onto that policy's path — no manual per-flow tunnel selection is needed at the ingress.

**Q4: How does TI-LFA improve on legacy LFA, and what's the key convergence-behavior advantage over MPLS-TE FRR?**
A: Legacy LFA can fail to find a loop-free alternate in many real topologies; TI-LFA computes a SID-stack-based backup that provides sub-50ms protection wherever topology mathematically allows it, AND converges to the exact same path the IGP would settle on after full reconvergence — avoiding a "different backup path than final path" transient that some legacy FRR/MPLS-TE FRR designs exhibit.

**Q5: Does TI-LFA guarantee 100% protection coverage across any topology? Why does this matter for design validation?**
A: No — coverage is bounded by actual topology connectivity; low-connectivity nodes/links may lack a mathematically valid loop-free alternate. Designers must validate actual per-node/per-prefix TI-LFA coverage rather than assuming universal protection, and provision supplemental protection where coverage gaps exist.

**Q6: When is Flex-Algo sufficient, and when do you actually need a PCE for SR-TE?**
A: Flex-Algo handles broad, relatively static constraint classes (e.g., a domain-wide low-latency plane) via pure distributed IGP computation with no controller dependency. A PCE is required when you need cross-policy global optimization — e.g., true path disjointness between two independently-computed policies, or bandwidth-aware admission control across the whole network — because no single headend has visibility into what every other headend is simultaneously reserving.

**Q7: Give a concrete example of a design that appears correct locally but fails without a PCE.**
A: Two headends independently compute "shortest available path" SR Policies for two customers requiring contractually disjoint paths; each headend, with only local visibility, can select paths that unknowingly share a common fiber segment. Only a PCE with full-topology, multi-policy awareness can actually guarantee disjointness across both policies simultaneously.

**Q8: What's the honest tradeoff analysis for deploying SRv6 today versus staying on SR-MPLS?**
A: SRv6 unifies transport and service layers under one IPv6-native data plane, eliminating the separate MPLS forwarding plane — strategically attractive. But 128-bit IPv6 SIDs versus 20-bit MPLS labels increase header overhead (MTU/throughput impact at scale), and not all existing ASICs/linecards support SRv6 SID processing at line rate as maturely as long-established SR-MPLS forwarding — so the decision should weigh current hardware/ecosystem maturity against strategic direction, not adopt SRv6 purely because it's newer.

**Q9: When should you use an Anycast-SID instead of a regular Prefix-SID in an SR Policy's SID list?**
A: Use Anycast-SID when the design intent is "reach any equally-valid node in this redundant set" (e.g., any of three peering routers) rather than one specific node — the IGP naturally load-balances/reroutes across surviving anycast members without SID-list recomputation. Using a regular Prefix-SID here over-constrains the path to one specific node and loses that built-in redundancy.

**Q10: Why is Binding-SID necessary in multi-area or multi-domain SR designs, beyond just being a convenience?**
A: Ingress hardware has a bounded efficient label-push depth; a long inter-area/inter-domain explicit path can exceed that depth if expressed as one flat SID list. Binding-SID collapses an entire remote segment/domain's path into a single locally significant SID, keeping the ingress SID stack shallow regardless of the true underlying hop count — it's also the standard mechanism for stitching an SR domain to a still-MPLS-TE/LDP-only domain during migration.

**Q11: How does BGP EPE differ from normal BGP best-path selection for controlling egress traffic, and why would a network need it?**
A: Normal BGP best-path applies one policy uniformly to all traffic toward a given prefix/peer. EPE advertises a distinct Peer-SID per peering session/link, letting a centralized controller steer specific flows out a chosen egress peer at a per-flow/per-policy granularity — e.g., moving high-value traffic off a congested transit link onto a settlement-free peer — without altering global BGP policy for all other traffic to that destination.

**Q12: What residual gap does micro-loop avoidance close that TI-LFA does not, and why does it matter for latency-sensitive services?**
A: TI-LFA protects the instant of failure and guarantees the network eventually reconverges to a correct final state, but the convergence process itself — different routers updating forwarding state at slightly different times — can still cause transient microloops lasting milliseconds to seconds. For loss-sensitive services like voice or financial trading, that transient window is unacceptable; micro-loop avoidance uses temporary SID-stack detours specifically during the convergence window to guarantee loop-free forwarding throughout, not just before and after.

---

## 3. Memory Map

```
Segment Routing
├── SR-MPLS Data Plane
│    ├── Prefix-SID (global, shortest-path-to-node)
│    └── Adjacency-SID (local, specific link)
│         └── "label stack = the path" — no per-hop signaling protocol needed
├── Label Space Foundation
│    └── SRGB (Segment Routing Global Block)
│         └── MUST be consistent domain-wide — mismatch breaks centralized label-stack computation silently
├── Traffic Engineering
│    └── SR Policy (headend + color + endpoint)
│         ├── Candidate paths: explicit SID-list / dynamic CSPF / PCE-delegated
│         └── Automated Steering — BGP color-community → auto-match to SR Policy (no manual tunnel select)
├── Fast Reroute (native, no backup-tunnel provisioning)
│    └── TI-LFA
│         ├── converges to SAME path as full IGP reconvergence (unlike legacy FRR backup ≠ final path)
│         └── coverage bounded by topology — must validate, not assume 100%
├── Distributed Constrained Computation
│    └── Flex-Algo
│         ├── custom SPF (e.g., min-delay, exclude-color) via IGP, per-algorithm Prefix-SID
│         └── sufficient for broad/static constraint classes — NOT a PCE replacement
├── Centralized Computation
│    └── PCE / PCEP
│         └── required for cross-policy global optimization (true disjointness, bandwidth-aware, multi-headend visibility)
├── Redundancy / Compression Primitives
│    ├── Anycast-SID — "any node in this set" (built-in load-balance/failover via IGP)
│    └── Binding-SID — collapses a whole remote path/domain into one local SID
│         ├── solves ingress push-depth hardware limits
│         └── enables SR-island ↔ legacy MPLS-TE/LDP domain stitching
├── Peering-Edge Traffic Engineering
│    └── BGP EPE (Peer-SID)
│         └── per-flow egress-peer steering beyond what BGP best-path alone can express
├── Brownfield Migration
│    └── SR Mapping Server (SRMS)
│         └── LDP ↔ SR label interworking during phased (non-flag-day) migration
├── Convergence-Window Protection
│    └── Micro-loop Avoidance
│         └── closes the gap TI-LFA leaves: transient loops DURING reconvergence itself
├── Multi-Domain / Inter-AS SR-TE
│    └── Binding-SID stitching across domains/ASes
│         └── mirrors Inter-AS L3VPN trust-boundary reasoning (per-domain visibility vs single global PCE)
└── Data-Plane Evolution
     └── SRv6
          ├── unifies transport+service under IPv6 (no MPLS plane)
          └── tradeoff: SID header overhead + hardware/ASIC maturity vs strategic alignment
```

---

## 4. CLI Cheat Sheet

| Task | Platform | Command |
|---|---|---|
| Enable SR-MPLS in IS-IS | IOS-XR | `router isis NAME` / `address-family ipv4 unicast` / `segment-routing mpls` |
| Set static Prefix-SID | IOS-XR | `interface Loopback0` / `address-family ipv4 unicast` / `prefix-sid absolute N` |
| Configure SRGB | IOS-XR | `router isis NAME` / `address-family ipv4 unicast` / `segment-routing global-block MIN MAX` |
| Enable TI-LFA per interface | IOS-XR | `interface X` / `address-family ipv4 unicast` / `fast-reroute per-prefix ti-lfa` |
| Define Flex-Algo | IOS-XR | `router isis NAME` / `flex-algo N` / `metric-type delay` |
| Assign algorithm-specific Prefix-SID | IOS-XR | `interface Loopback0` / `prefix-sid algorithm N absolute LABEL` |
| Define SR Policy | IOS-XR | `segment-routing` / `traffic-eng` / `policy NAME` / `color N end-point ipv4 x.x.x.x` |
| Define explicit segment-list | IOS-XR | `segment-routing traffic-eng` / `segment-list NAME` / `index N mpls label L` |
| Configure PCE client (PCC) | IOS-XR | `segment-routing traffic-eng` / `pcc` / `pce address ipv4 x.x.x.x` |
| Delegate SR Policy path to PCE | IOS-XR | `policy NAME` / `candidate-paths` / `preference N dynamic pcep` |
| Enable SRv6 locator | IOS-XR | `segment-routing srv6` / `locator NAME` / `prefix X::/N` |
| Bind IS-IS to SRv6 locator | IOS-XR | `router isis NAME` / `address-family ipv6 unicast` / `segment-routing srv6` / `locator NAME` |
| Verify SR-MPLS label table | IOS-XR | `show mpls forwarding` |
| Verify SID/label mapping | IOS-XR | `show isis segment-routing label table` |
| Verify SR Policy status | IOS-XR | `show segment-routing traffic-eng policy` |
| Verify TI-LFA coverage | IOS-XR | `show isis fast-reroute summary` |
| Verify PCEP session state | IOS-XR | `show segment-routing traffic-eng pcc ipv4 peer` |
| Configure Anycast-SID (n-flag-clear) | IOS-XR | `interface LoopbackN` / `prefix-sid absolute N n-flag-clear` |
| Define Binding-SID on an SR Policy | IOS-XR | `policy NAME` / `binding-sid mpls N` |
| Enable BGP EPE on a peer session | IOS-XR | `router bgp ASN` / `neighbor x.x.x.x` / `egress-engineering` |
| Configure SR Mapping Server prefix range | IOS-XR | `segment-routing` / `mapping-server prefix-sid-map` / `address-family ipv4` / `prefix/len range N start-label` |
| Enable micro-loop avoidance | IOS-XR | `router isis NAME` / `address-family ipv4 unicast` / `microloop avoidance segment-routing` |
| Verify Anycast-SID advertisement | IOS-XR | `show isis segment-routing prefix-sid detail` |
| Verify Binding-SID forwarding entry | IOS-XR | `show segment-routing traffic-eng policy name NAME detail` |
| Verify EPE Peer-SID allocation | IOS-XR | `show bgp egress-engineering` |
| Verify SRMS mapping table | IOS-XR | `show isis segment-routing prefix-sid-map` |
| Verify micro-loop avoidance status | IOS-XR | `show isis fast-reroute microloop-avoidance` |
