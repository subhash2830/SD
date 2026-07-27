# CCIE/CCDE — ISIS
*Simple explanations, CCDE-level design depth, interview answers, CLI, and a concept memory map — covering all 26 ISIS labs.*

---

## 1. Adjacency Fundamentals

### 1.1 Hello/Hold Timers
**What:** The hello interval controls how often Hellos are sent; the hold time (hello × multiplier) is how long a neighbor is considered alive without a Hello.
**Why it matters (CCDE lens):** Unlike OSPF, ISIS hello/hold timers do **not** need to match between neighbors — each router independently reports its own hold time in the Hello, and the receiver uses whatever the neighbor tells it. This is a deliberate protocol design choice: it lets you tune convergence speed asymmetrically (e.g., a core P router can hold a fast timer while a slower CE doesn't need to match it) — something OSPF cannot do natively.
**Real-world example:** On a LAN segment, the DIS often uses a lower hold time (e.g., 3x faster) than non-DIS routers, since losing the DIS causes a full re-election and LSP resync — faster detection there minimizes flooding disruption.
```
int Gi2.12
 isis network point-to-point
 isis hello-interval 2
 isis hello-multiplier 3
```

### 1.2 Hello Padding
**What:** By default ISIS pads every Hello to the full interface MTU, to proactively catch MTU mismatches before real traffic hits the problem.
**Why it matters (CCDE lens):** This is a bandwidth-vs-safety trade-off. Padding on every Hello (forever) wastes bandwidth on stable, already-verified links; padding only during adjacency formation (`hello padding sometimes`) keeps the MTU-mismatch protection while cutting steady-state overhead — useful at scale on many point-to-point core links.
```
no isis hello padding                 ! never pad (IOS-XE)
hello-padding sometimes [level 2]      ! pad only first N (IOS-XR)
```

### 1.3 Mesh Groups
**What:** By default, ISIS floods a received LSP out every other interface. Mesh groups let you restrict flooding so an LSP received on a group member is not re-flooded to other members of the same group.
**Why it matters (CCDE lens):** This is a **flooding-scale control** for highly meshed cores (think large IXP or DC-core-like ISIS topologies) — without it, flooding redundancy grows combinatorially with mesh density, wasting bandwidth and CPU on duplicate LSPs. But it's a dangerous knob: mesh groups can silently break convergence, since a router may only learn of a changed LSP via the slow periodic CSNP instead of immediate flooding — and on point-to-point mesh-group members, it may **never** learn of the LSP at all outside periodic resync.
**Real-world example:** A full-mesh set of core P routers over a LAN/hub design (e.g., DWDM ring emulated as LAN) uses mesh groups to avoid re-flooding every LSP change N times across N mesh members — critical at 50+ node core scale.
> **Design trap:** Don't apply mesh groups to point-to-point links casually — the periodic-CSNP-only fallback that saves you on a LAN doesn't exist on p2p circuits.

---

## 2. Metrics & Compatibility

### 2.1 Default Metric & Wide Metrics
**What:** Narrow metrics are 6-bit (max 63 per link, max path metric 1024) — a legacy constraint. Wide metrics extend this to 24-bit, needed for any modern TE/SR use.
**Why it matters (CCDE lens):** Any design that touches SR, TE, or large multi-area topologies with meaningful metric differentiation **requires** wide metrics — narrow metrics cap out fast in large networks. This is one of the first compatibility checks in any ISIS migration/greenfield design.
```
router isis
 metric-style wide
 metric 100
```

### 2.2 Metric-Style Transition (narrow/wide interop)
**What:** Five valid combinations exist: narrow-only (default), wide-only, `transition` (generate+accept both), `narrow transition` (generate narrow, accept both), `wide transition` (generate wide, accept both).
**Why it matters (CCDE lens):** This is the **exact mechanism for a live, non-disruptive narrow→wide migration** in a running SP network — you can't just flip every router to wide simultaneously without breaking routes on routers still narrow-only. `transition` is the safe rolling-upgrade state.
**Real-world example / Troubleshooting pattern:** A network with mixed narrow/wide routers shows `**` (unreachable metric) in `show isis topology` — the classic symptom of a metric-style mismatch. Fix: put the mismatched routers into `metric-style transition` (not `narrow transition`, which stops generating wide and looks like "disabling a feature" in an exam context).

### 2.3 LSP Size / Fragmentation
**What:** `lsp-mtu` controls when a router's own LSP is split into fragments (`SysID.CircuitID-Fragment`, e.g., `R1.00-01`).
**Why it matters (CCDE lens):** Relevant at scale when redistributing large numbers of external routes or running many TE/SR TLVs — a single LSP can outgrow a conservative MTU on a path, causing fragmentation issues or (historically) even flooding failures on paths with inconsistent MTU. CCDE-level awareness: LSP fragmentation ≠ IP fragmentation; it's an ISIS-internal concept, and each fragment is flooded/aged independently.

### 2.4 ISIS Timers (SPF/PRC/LSP generation/flooding pacing)
**What:** A layered set of throttle timers: LSP generation backoff, SPF backoff, PRC (partial route calc, for non-topology prefix-only changes) backoff, and interface-level flooding/retransmission pacing.
**Why it matters (CCDE lens):** This is the classic **convergence-speed vs. stability/CPU-load trade-off**. Aggressive (low) timers converge faster but risk CPU spikes and SPF "storms" during flapping events; conservative timers protect the control plane at the cost of slower reaction. CCDE candidates should be able to reason about exponential backoff design (initial wait → secondary wait → max wait) as the standard pattern for absorbing bursts of related changes without either being too slow on the first event or overwhelming the CPU on repeated events.
```
router isis
 spf-interval 3 250 500        ! max 3s, initial 250ms, 2nd-run min 500ms
 lsp-gen-interval 3 500 750
 prc-interval 3 250 500
int Gi2.12
 isis lsp-interval 20                    ! flooding pacing
 isis retransmit-throttle-interval 40    ! retransmit pacing
```

---

## 3. Overload Bit — Graceful Insertion/Removal from Transit

**What:** A router sets the OL bit in its own LSP to say "don't use me for transit" — but it can still be reached for its own directly-attached prefixes.
**Why it matters (CCDE lens):** This is ISIS's version of the "make-before-break" / graceful-drain pattern used everywhere in network operations (BGP graceful shutdown, OSPF max-metric, LDP session protection). Two production use cases: (1) **on-startup, wait-for-bgp** — hold the router out of the forwarding path until BGP has fully converged, preventing it from blackholing traffic during a reload before its BGP table is populated; (2) **manual OL set** — a clean way to drain a router for maintenance without physically pulling links.
**Key CCDE distinction vs. OSPF max-metric:** ISIS OL-bit router is **never** used for transit — even as a last resort with no other path. OSPF max-metric routers can still be used if no alternative exists. This materially changes failure-mode reasoning in mixed OSPF/ISIS designs.
```
router isis
 set-overload-bit on-startup wait-for-bgp suppress external   ! IOS-XE
 set-overload-bit                                             ! immediate
```
**Interaction with TE:** An OL-bit node is excluded from CSPF for TE tunnel path computation by default — a tunnel head-end will report "no path" if the only route crosses an overloaded node. Override with `mpls traffic-eng path-selection overload allow middle` (XE) / `ignore overload mid` (XR) — a real troubleshooting pattern, not just a lab curiosity.

---

## 4. Prefix Advertisement Control

### 4.1 Prefix Suppression
**What:** By default ISIS advertises both the loopback AND the transit link's own subnet as prefixes. `advertise passive-only` (or per-interface `no isis advertise prefix` / XR `suppressed`) stops advertising the link subnet, leaving only passive (loopback) prefixes.
**Why it matters (CCDE lens):** In large SP cores, advertising every /30 or /31 point-to-point transit link as a routable prefix bloats the RIB/LSDB for zero operational benefit — nothing needs to route TO a P2P transit subnet. This is a standard **RIB/LSDB hygiene** practice at scale. Reachability to the nexthop is preserved because ISIS learns the nexthop's interface address directly from the Hello, not from the suppressed prefix.

### 4.2 Prefix Summarization
**What:** L1/L2 boundary routers summarize L1→L2 (like an OSPF ABR); any router can summarize redistributed/external routes at the point of redistribution.
**Why it matters (CCDE lens):** Summarization is your primary lever for **LSDB/RIB scale control** in large multi-area SP designs — fewer specific routes flooded domain-wide means smaller LSDBs, faster SPF, and blast-radius containment (a flapping /32 inside an area doesn't need to ripple L2-wide if it's covered by a stable summary). Unlike EIGRP, the ISIS summary route keeps the same AD (115) as regular routes — no "trusted less" signaling via AD.
```
router isis
 summary-address 1.1.1.0 255.255.255.0 level-2   ! L1->L2 summarization
 summary-address 5.5.5.0 255.255.255.0 level-1   ! L2->L1 (after leaking!)
```

### 4.3 Route Filtering with Tags
**What:** Wide-metric TLVs carry a tag field; you can match on tag in route-maps/route-policies to selectively filter redistribution (e.g., L2→L1 leaking).
**Why it matters (CCDE lens):** Tags are the standard mechanism for **policy-based selective route leaking** — e.g., "leak only routes belonging to this business unit/customer into L1" without hand-maintaining prefix-lists that go stale. Narrow metrics have no tag field — another reason wide metrics are a prerequisite for any policy-rich design.

---

## 5. Multi-Area & Inter-Area Design

### 5.1 Non-Optimal Intra-Area Routing
**What:** ISIS always prefers an L1 (intra-area) route over an L2 (inter-area) route, **regardless of metric**. If two routers share an area but only have an L2-only adjacency between them, traffic can take a longer L1 path through a third router instead of the shorter direct L2 link.
**Why it matters (CCDE lens):** This is a **classic ISIS design trap** with no OSPF equivalent behaving quite the same way — in OSPF, a link belongs to exactly one area (intra vs inter is a hard boundary), forcing an explicit trade-off. ISIS lets the same link carry both L1 and L2 simultaneously, which is more flexible, but only if you remember to configure it that way. Forgetting this is a top real-world path-selection troubleshooting scenario.
**Fix:** Allow the link to run both L1 and L2 (remove `isis circuit-type level-2-only`) so an L1 path exists directly.

### 5.2 Multi-Area (multiple NET addresses)
**What:** A router can belong to more than one L1 area simultaneously by configuring multiple NET addresses (same System ID, different area IDs) — up to a negotiated max (default 3, adjustable on IOS-XE, fixed at 3 on IOS-XR).
**Why it matters (CCDE lens):** This is how you **merge/bridge two L1 areas into one L1 domain** through a single router, without touching circuit types — useful in mergers/acquisitions or phased area-renumbering projects where you can't do a flag-day area cutover. The merged areas act as one L1 topology on that router even though multiple area IDs are declared.
> **Platform limit gotcha:** IOS-XR caps max-area-addresses at 3 (not configurable beyond that) — a real platform constraint that can force a different design (e.g., you may not be able to make one router straddle 4 areas on XR, requiring an extra hop or router in the design).

### 5.3 Default Route Preference (L1 area exit selection)
**What:** An L1-only router normally picks its L2 exit via the ATT bit (any L1/L2 router with ATT set is an equally-valid default). To force preference of one exit over another **without using metrics**, originate an explicit `0/0` route into L1 from the preferred exit.
**Why it matters (CCDE lens):** An explicit default route always wins over an ATT-bit-derived default — this is the standard technique for **deterministic default-path engineering** in stub/L1 areas (e.g., preferring a primary SP peering exit over a backup) when you can't or don't want to manipulate the IGP metric itself.
```
route-map SET_L1
 set level level-1
router isis
 default-information originate route-map SET_L1
```

### 5.4 Conditional ATT Bit
**What:** `set-attached-bit route-map <name>` makes an L1/L2 router only set the ATT bit (i.e., only advertise itself as an L2 exit) when a specific condition (e.g., a specific prefix present in the RIB) is true.
**Why it matters (CCDE lens):** This prevents an L1/L2 router from being falsely advertised as a viable Internet/L2 exit when it has actually lost its own upstream reachability (e.g., its own external BGP/L2 path is down) — directly analogous to conditional route advertisement patterns used elsewhere (BGP conditional advertisement, IP SLA-tracked static defaults). Prevents blackholing traffic that trusts a "false" exit.

### 5.5 Backdoor Link
**What:** A direct L1 adjacency between routers in two *different* areas, used for a private shortcut without transiting the L2 backbone — combined with selective L2→L1 leaking so only intended traffic uses it.
**Why it matters (CCDE lens):** Demonstrates that ISIS area boundaries are a **policy construct, not a hard physical constraint** — you can build a disjoint, area-crossing L1/L2 adjacency for controlled traffic engineering (e.g., a direct low-latency link between two data centers in different areas) while keeping the backbone as the default path for everything else.

---

## 6. IPv6 in ISIS — Single-Topology vs Multi-Topology

This is one of the highest-value CCDE topics in ISIS — the single vs. multi-topology decision has real operational and scaling consequences.

| | Single-Topology (ST) | Multi-Topology (MT) |
|---|---|---|
| Model | IPv6 rides on the IPv4 topology/adjacencies | IPv6 has its own separate topology, own metrics, can take a different path |
| Requirement | **Every** link must run both IPv4 AND IPv6, or the adjacency breaks entirely | IPv4-only and IPv6-only neighbors can coexist; incremental rollout |
| Metric-style | Narrow or wide, either works | **Requires wide metrics** (IOS-XE) |
| Default on | IOS-XE | IOS-XR |
| Cost | Cheaper — one topology, one SPF run | More CPU/memory — two independent SPF-maintained topologies |

**Why it matters (CCDE lens):** This is a direct **cost-vs-flexibility trade-off** you must be able to argue in an interview. ST is cheap but "all or nothing" — enabling IPv6 on one link with no IPv6 on the neighbor kills the adjacency for BOTH IPv4 and IPv6 (a genuinely dangerous default-behavior gotcha in a live network). MT is safe for incremental/brownfield rollout and lets IPv6 take a genuinely different path than IPv4 (relevant if you need IPv6-specific TE/latency policy), at real control-plane cost from maintaining two topologies.

**The two critical gotchas:**
- **ST migration risk:** Turning on IPv6/ST on one router before its neighbor is IPv6-ready breaks the existing IPv4 adjacency too — `no adjacency-check` (XE) / `adjacency-check disable` (XR) is the safe-rollout escape valve, letting the adjacency stay IPv4-only up until every router has IPv6 enabled.
- **ST→MT migration:** Cannot be a flag-day cutover on IOS-XR (no "transition" keyword there) — expect a brief IPv6 reachability blip on XR nodes; IOS-XE's `multi-topology transition` keyword allows a genuinely non-disruptive rolling migration by generating both TLV types simultaneously.

**Real-world example:** A greenfield SP core standardizes on IOS-XR (MT by default) specifically because it allows incremental, per-router IPv6 rollout without a synchronized flag-day cutover across the whole domain — a meaningful operational-risk reduction on a live production network.

---

## 7. Authentication

**What:** Three independent authentication scopes exist: Hello (per-link), L1 area password (all L1 LSPs/SNPs), and L2 domain password (all L2 LSPs/SNPs) — each can use plaintext or MD5, and each is configured separately.
**Why it matters (CCDE lens):** Unlike a single "turn on auth" toggle, ISIS's layered authentication model lets you scope trust precisely — e.g., authenticate the L2 backbone strictly (domain-wide integrity) while being more permissive at L1 area edges during a migration. The "accept unauthenticated" transition knobs (`authenticate snp send-only`, accepting missing/wrong auth) exist specifically to allow a **non-disruptive rollout** of authentication onto a live network, the same pattern seen in the metric-style and IPv6 migrations above — a recurring CCDE theme: *how do you change a live control-plane setting without an outage?*
```
router isis
 area-password AREA1234 authenticate snp send-only
int Gi2.12
 isis password HELLO123
 isis authentication mode md5
 isis authentication key-chain ISIS_HELLO
```

---

## 8. Operational Visibility

### 8.1 Log Neighbor Changes
**What:** `log-adjacency-changes` (default on) logs adjacencies lost via Hello timeout. `log-adjacency-changes all` additionally logs adjacencies lost due to other causes (e.g., interface shutdown).
**Why it matters (CCDE lens):** A small but real NOC-visibility design decision — default logging can miss the most common real-world failure cause (a physical/interface event) if you don't add `all`. Trivial to configure, easy to overlook, and a genuine gap in incident timelines if missed.

---

## 9. Troubleshooting Patterns (Interview Gold)

### 9.1 "No Routes" — Metric-Style Mismatch
**Symptom:** `show isis topology` shows `**` (unreachable) for the metric on some routers; RIB is missing routes.
**Root cause:** Some routers generate/accept only narrow metrics, others only wide — they can't parse each other's metric TLVs.
**Fix:** `metric-style transition` on the mismatched routers (generate AND accept both) — NOT `narrow transition` (stops generating wide) or `wide transition` (stops generating narrow), either of which would silently regress capability elsewhere in the network.

### 9.2 Broken P2P Adjacency — The Full Checklist
For any two ISIS neighbors to form an adjacency, ALL of the following must align:
- Same IP subnet (IPv4)
- Matching circuit type / level combination (L1 talks to L1, L2 talks to L2, L1/L2 is compatible with either)
- Matching Hello authentication
- Matching MTU (padded Hellos expose mismatches)
- Matching protocol-supported set — **unless** using multi-topology, or `no adjacency-check` for single-topology IPv6 rollout

**Why it matters (CCDE lens):** This checklist IS the mental model for any ISIS adjacency troubleshooting question — walk it in order rather than guessing. Notice IPv6/MT is the one legitimate exception to the "protocols must match" rule, which ties directly back to Section 6.

### 9.3 iBGP Session Won't Establish Despite "Having a Route"
**Symptom:** `show ip bgp neighbor` claims a route to the peer exists, but the TCP session never opens; `debug ip bgp` shows the real story.
**Root cause:** The RIB entry is a `0/0` default route (from ATT bit or default-origination) — but BGP session establishment and BGP nexthop resolution **cannot use a default route**, even though other show commands make it look "reachable."
**Fix:** Leak an explicit /32 for the peer's loopback from L2 into L1 (route-map/route-policy matching that specific prefix) so BGP has a real, non-default route to key off of. Needed on **both** the TCP-initiating side and — separately — wherever nexthop resolution for the advertised prefix happens (IOS-XR is stricter than IOS-XE about resolving nexthops through 0/0).
**Why it matters (CCDE lens):** A textbook "the show command lied to me" scenario — reinforces that CCDE-level troubleshooting requires understanding what a protocol is actually allowed to use (default route eligibility rules), not just whether a route "exists" in the table.

### 9.4 TE Tunnel Won't Come Up — Overloaded Mid-Node
**Symptom:** Tunnel head-end reports "no path"; TED shows a transit node marked as excluded/"no SPF."
**Root cause:** A mid-path node has the ISIS overload bit set — CSPF automatically excludes OL-bit nodes from any computed path, even if it's the only path available.
**Fix:** `mpls traffic-eng path-selection overload allow middle` (XE) / `ignore overload mid` (XR) if you deliberately want to permit transit through an overloaded node for TE purposes.
**Why it matters (CCDE lens):** Ties Section 3 (Overload Bit) directly into MPLS-TE path computation — a great example of one IGP-level design decision (draining a router) having a downstream effect on a completely different feature (TE tunnel admission) that's easy to forget when troubleshooting "tunnel down" in isolation.

---

## 10. CCDE-Style Interview Q&A

**Q1. Why doesn't ISIS require hello/hold timers to match between neighbors, unlike OSPF?**
Each router independently advertises its own hold time in its Hello; the receiving neighbor just uses whatever value it's told. This lets you tune convergence asymmetrically per-router (e.g., a DIS using a faster hold time) without needing network-wide timer coordination.

**Q2. A network has some routers on narrow metrics and some on wide. What breaks, and how do you fix it live without an outage?**
Routers can't parse each other's metric TLVs, so `show isis topology` shows unreachable (`**`) metrics and routes go missing on the mismatched side. Fix by setting `metric-style transition` on the narrow-only routers — this generates AND accepts both metric types, letting you migrate the rest of the network to wide-only later without a flag-day cutover.

**Q3. Why would two routers in the same L1 area take a longer path through a third router instead of a direct, shorter link between them?**
Because that direct link is configured L2-only. ISIS always prefers an L1 route over an L2 route regardless of metric — so the "shorter" L2 path loses to a longer L1 path via a third router. Fix: allow the direct link to run both L1 and L2.

**Q4. What's the practical difference between the ISIS overload bit and OSPF's max-metric?**
An OL-bit ISIS node is NEVER used for transit, even as a last resort with no alternative path. OSPF's max-metric node CAN still be used for transit if no other path exists. This changes your failure-mode assumptions in a mixed or migrating network.

**Q5. When would you choose IPv6 multi-topology over single-topology in an ISIS design?**
When you need incremental/brownfield IPv6 rollout without a synchronized flag-day cutover (ST requires every router on a link to support both v4 and v6, or the whole adjacency breaks), or when IPv6 needs genuinely independent path selection (different metrics/topology) from IPv4. The cost is running two SPF-maintained topologies instead of one.

**Q6. A BGP session between two ISIS-connected routers won't establish, even though the routing table appears to show a route to the peer. What's your troubleshooting angle?**
Check whether that "route" is actually a default (0/0) route — BGP session establishment and nexthop resolution cannot use a default route even if `show ip route` output makes it look reachable. The real fix is leaking an explicit /32 for the peer into the level where it's needed.

**Q7. Why can an ISIS router belong to multiple Level-1 areas simultaneously, and what's a real use case?**
By configuring multiple NET addresses (same System ID, different area IDs), a router merges those L1 areas into a single L1 topology on itself — useful for merging domains during an M&A integration or a phased area-renumbering project without a disruptive cutover. Watch platform limits: IOS-XR cannot exceed 3 area addresses.

**Q8. Why does an overloaded (OL-bit) node block an MPLS-TE tunnel from finding a path, and how do you override it?**
CSPF automatically excludes OL-bit nodes from any computed TE path, since the OL bit means "never use me for transit" — the same rule that applies to normal IGP forwarding. Override per-tunnel with `path-selection overload allow middle` (XE) or `ignore overload mid` (XR) if you deliberately need to transit an overloaded node.

---

## 11. Memory Map — How These Concepts Connect

```
ISIS Core
│
├── Adjacency Formation (Section 1, 9.2)
│     Hello/Hold Timers ── independent per router (unlike OSPF)
│     Hello Padding ────── MTU-mismatch protection, tunable for bandwidth
│     Mesh Groups ──────── flooding scale control, dangerous on p2p
│     └─ feeds → "Adjacency Checklist" (9.2) which itself has an
│                explicit IPv6/MT exception (→ Section 6)
│
├── Metrics (Section 2)
│     Narrow → Wide migration ── uses `transition` style (same migration
│     │                          PATTERN reused in Section 6 IPv6 ST→MT
│     │                          and Section 7 Authentication rollout)
│     Wide Metrics ── PREREQUISITE for:
│        ├─ Route tags (4.3)
│        ├─ IPv6 Multi-Topology (6)
│        └─ Any SR/TE design (not covered here, but same dependency)
│
├── Overload Bit (Section 3)
│     "Never used for transit" ── stricter than OSPF max-metric
│     ├─ used for: graceful startup (wait-for-bgp), manual drain
│     └─ directly BREAKS → MPLS-TE CSPF path computation (9.4)
│           unless explicitly overridden
│
├── Prefix Control (Section 4)
│     Suppression → LSDB hygiene (don't advertise transit /30s)
│     Summarization → scale + blast-radius containment at L1/L2 boundary
│     Tags → policy-based selective leaking (needs Wide Metrics)
│     └─ feeds → Multi-Area design (Section 5) which relies on
│                controlled L2→L1 leaking for default-route and
│                iBGP-nexthop-resolution scenarios (9.3)
│
├── Multi-Area Design (Section 5)
│     Non-Optimal Intra-Area Routing ── L1-always-preferred-over-L2 rule
│     Multi-Area (multiple NETs) ── merges L1 areas via one router
│     Default Route Preference ── explicit 0/0 beats ATT-bit default
│     Conditional ATT bit ── don't falsely advertise as an exit
│     Backdoor Link ── policy-based shortcut across area boundaries
│     └─ ALL of these interact with iBGP nexthop/session
│          troubleshooting (9.3) — "the RIB has *a* route,
│          but not the RIGHT KIND of route" is the recurring theme
│
├── IPv6 (Section 6)
│     ST vs MT ── cost/flexibility trade-off
│     └─ REQUIRES Wide Metrics (→ Section 2) for MT on IOS-XE
│     └─ Migration risk pattern mirrors metric-style transition (2.2)
│          and authentication rollout (7) — "don't break the live
│          adjacency while adding a new capability" is a repeating
│          CCDE theme across all three
│
├── Authentication (Section 7)
│     Three independent scopes: Hello / L1 domain / L2 domain
│     Rollout uses same "accept both, migrate gradually" pattern
│
└── Operational Visibility (Section 8) + Troubleshooting (Section 9)
      log-adjacency-changes all ── catches interface-down events
      the 4 troubleshooting patterns (9.1–9.4) are direct applications
      of Sections 2, 3, 5, and 9.2 respectively — troubleshooting IS
      just "which section's assumption got violated"
```

**The single biggest recurring CCDE theme across ISIS:** *how do you change a live control-plane behavior (metric style, IPv6 topology mode, authentication) without breaking the network during the transition?* Nearly every "advanced" ISIS feature here exists to answer that question in one specific dimension.

---

## 12. CLI Cheat Sheet

| Purpose | Command |
|---|---|
| P2P network type + custom hello/hold | `isis network point-to-point` / `isis hello-interval <n>` / `isis hello-multiplier <n>` |
| Reduce/disable hello padding | `no isis hello padding` (XE) / `hello-padding sometimes` (XR) |
| Mesh group (restrict flooding) | `isis mesh-group <n>` / `isis mesh-group blocked` |
| Enable wide metrics | `metric-style wide` |
| Safe narrow↔wide migration | `metric-style transition` |
| Set default metric | `metric <n>` (under `add ipv4` on XR) |
| LSP fragmentation size | `lsp-mtu <bytes>` |
| SPF/PRC/LSP-gen backoff timers | `spf-interval <max> <init> <2nd>` / `prc-interval ...` / `lsp-gen-interval ...` |
| Flooding pacing | `isis lsp-interval <ms>` / `isis retransmit-throttle-interval <ms>` |
| Set overload bit | `set-overload-bit [on-startup <sec>|wait-for-bgp] [suppress ...]` |
| Suppress transit-link prefixes | `advertise passive-only` / `no isis advertise prefix` (XE) / `suppressed` (XR) |
| L1→L2 summarization | `summary-address <net> <mask> level-2` |
| L2→L1 leak + summarize | `redistribute isis ip level-2 into level-1 route-map <rm>` + `summary-address ... level-1` |
| Explicit default route into L1 | `default-information originate route-map <rm-set-level-1>` |
| Conditional ATT bit | `set-attached-bit route-map <rm>` |
| Multiple area membership | `net <area1>...`  `net <area2>...` (same System ID) |
| Max area addresses (XE only) | `max-area-addresses <n>` |
| IPv6 single-topology (XR) | `address-family ipv6 unicast` → `single-topology` |
| IPv6 no-adjacency-check (safe ST rollout) | `no adjacency-check` (XE) / `adjacency-check disable` (XR) |
| IPv6 multi-topology | `add ipv6` → `multi-topology` (requires wide metrics on XE) |
| Non-disruptive XE ST→MT migration | `multi-topology transition` |
| Hello/area/domain authentication | `isis password <pwd>` / `area-password <pwd>` / `authentication mode md5` |
| Log all adjacency changes | `log-adjacency-changes all` |
| Allow TE tunnel through OL node | `mpls traffic-eng path-selection overload allow middle` (XE) / `ignore overload mid` (XR) |
| Verify negotiated metric-style | `show isis protocol` |
| Verify topology / metric issues | `show isis topology` |
| Verify LSP contents / OL bit | `show isis database` |

---
*Source: CCIE-SP v5.1 Labs — ISIS section (26 labs): Start, Topology, Prefix Suppression, Hello Padding, Overload Bit, LSP Size, Default Metric, Hello/Hold Timer, Mesh Groups, Prefix Summarization, Default Route Preference, ISIS Timers, Log Neighbor Changes, Troubleshooting 1–2, IPv6 ST/MT (4 labs), Wide Metrics Explained, Route Filtering, Backdoor Link, Non-Optimal Intra-Area Routing, Multi Area, Authentication, Conditional ATT Bit, Troubleshooting iBGP, Troubleshooting TE Tunnel.*
