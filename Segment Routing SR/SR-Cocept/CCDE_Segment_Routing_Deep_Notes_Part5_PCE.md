# CCDE-Level Segment Routing Deep Notes — Part 5 (PCE, BGP EPE, Disjoint Paths, Telemetry)
### Source labs: CCIE-SP v5.1 — SR Inter-IGP using PCE, SR-TE PCC Features, SR-TE PCE Instantiated Policy,
### SR-TE PCE Redundancy, SR-TE PCE Redundancy w/ Sync, SR-TE Basic BGP EPE, SR-TE BGP EPE for Unified MPLS,
### SR-TE Disjoint Paths, SR Converged SDN Transport Challenge, Performance-Measurement (Interface Delay)
### Compiled: 2026

> Continuation of the earlier CCDE SR deep-notes files (SRGB/PHP/ExpNull/Anycast; IGP/BGP SR fundamentals;
> LFA/RLFA/TI-LFA; SR-TE policies). This file covers the PCE (Path Computation Element) architecture, BGP
> Egress Peer Engineering, path disjointness, a full multi-domain reference design (Converged SDN
> Transport), and delay-based performance measurement.

---

## Table of Contents
1. PCE Fundamentals — The Stateful PCC/PCE Model and the TE Router-ID Requirement
2. PCEP Workflow — Reports, Updates, Delegation, and Initiation
3. SR-TED and Multi-Instance BGP-LS/IGP Distribution
4. ODN with PCE Fallback and the BGP Recursion Fix
5. Computation Design Models — Centralized, Distributed, and Hybrid
6. PCC Features — PCEP Authentication and report-all
7. PCE-Initiated Policies
8. PCE Redundancy — Precedence Election and Failover
9. PCE Redundancy with State-Sync — Master/Slave and Disjoint Path Coordination
10. BGP Egress Peer Engineering (EPE) Fundamentals
11. BGP-LS for Multi-Domain Unified MPLS with EPE
12. SR-TE Disjoint Paths
13. Anycast SID for ABRs with Conditional Advertisement (Converged SDN Transport Pattern)
14. iBGP Underlay Design for Loopback Reachability in Multi-Domain SR (Converged SDN Transport Pattern)
15. Performance Measurement — Interface Delay (TWAMP-light)

---

## 1. PCE Fundamentals — The Stateful PCC/PCE Model and the TE Router-ID Requirement

**1. Definition**
A PCE (Path Computation Element) is a centralized server that computes SR-TE paths on behalf of routers,
using a full view of the network topology. A router that asks the PCE to compute a path is called a PCC
(Path Computation Client). Cisco's implementation is always stateful — the PCE remembers and actively
manages every policy it computes.

**2. Why it exists**
A single router only knows its own local topology view. Some path types genuinely require full network
visibility no single router has — inter-domain paths spanning multiple IGP areas/instances, or
disjoint-path computation, where one path's calculation depends on another path elsewhere in the network.
A PCE centralizes the full topology view so routers can delegate these hard problems.

**3. How it works**
- The PCE builds an SR-TED (SR Traffic Engineering Database) — a consolidated topology graph, fed from
  the IGP(s) via `distribute link-state`.
- **Critical prerequisite**: every node must have a TE Router ID, or the PCE cannot even place that node
  in its topology graph.
  ```
  #ISIS
  router isis 1
   address-family ipv4 unicast
    mpls traffic-eng router-id lo1
    mpls traffic-eng level-2
  mpls traffic-eng

  #OSPF
  router ospf 1
   mpls traffic-eng router-id lo1
   area 0
    mpls traffic-eng
  mpls traffic-eng
  ```
  Global `mpls traffic-eng` must also be enabled — no need to actually run RSVP-TE.
- PCE server:
  ```
  #PCE
  pce
   address ipv4 10.10.10.1
  ```
- PCC:
  ```
  #PCC
  segment-routing
   traffic-eng
    pcc
     source-address ipv4 1.1.1.1
     pce address ipv4 10.10.10.1
  ```
- Stateful means the PCE actively tracks every delegated policy and proactively recomputes/pushes updates
  as the topology changes, without the PCC needing to ask again.

**4. Real-world use case**
Any SP network needing inter-domain SR-TE paths or path disjointness guarantees requires a PCE.

**5. Failure scenario**
An operator sets up PCEP/BGP-LS correctly but forgets TE RID + global mpls traffic-eng on even one node —
the PCE's graph has a hole there, and any path needing to transit that node silently fails or detours.

**6. Design insight**
TE-RID is a genuine hidden prerequisite — treat it as a mandatory baseline item on every node in scope, as
part of any PCE-readiness checklist.

**7. Interview-ready answer**
"A PCE is a centralized, stateful path-computation server; routers delegate as PCCs when they need
full-topology visibility for inter-domain or disjoint-path computation. A commonly-missed prerequisite is
that every node needs a TE Router ID and global mpls traffic-eng enabled, even without RSVP-TE, or the PCE
can't place that node in its graph at all."

---

## 2. PCEP Workflow — Reports, Updates, Delegation, and Initiation

**1. Definition**
PCEP is the signaling protocol between PCCs and the PCE, built around PCEP Report (PCC -> PCE, describing
policy state) and PCEP Update (PCE -> PCC, providing a computed path) — plus PCEP Initiate, used when the
PCE pushes a brand-new policy the PCC never asked for.

**2. Why it exists**
The protocol needs a clean way for a PCC to request computation and for the PCE to respond, while also
supporting the PCE proactively pushing policies, and keeping both sides synchronized as the network
changes.

**3. How it works**
1. PCC sends a Report with an empty SID list and the Delegate (D) flag set — "please compute this and
   manage it."
2. PCE computes the path, signals it via an Update.
3. PCC installs the policy, sends a Report echoing back the SID list as an ACK.
- If the PCE can't compute the policy, it replies with an empty Update and clears the delegate flag.
- Topology changes trigger automatic PCE recomputation and a new Update if anything changed.
- **PCEP Initiate**: the PCE defines a full policy under its own config and pushes it to a PCC with zero
  local configuration:
  ```
  #PCE
  pce
   segment-routing
    traffic-eng
     peer ipv4 1.1.1.1
      policy POL1
       color 10 end-point ipv4 7.7.7.1
       candidate-paths
        preference 100
         explicit segment-list R1_TO_R7
  ```
- **Tie-breaking**: if a PCE-initiated policy collides with a locally-configured one on the same
  preference/color/endpoint, the headend prefers its own local path.
- PCE can also dictate the BSID for an initiated policy:
  ```
  pce
   segment-routing
    traffic-eng
     peer ipv4 7.7.7.1
      policy POL1
       binding-sid mpls label <num>
  ```

**4. Real-world use case**
Report/Update cycle is standard for ODN and explicit PCE-computed policies. PCEP Initiate is used by
higher-level SDN orchestration systems pushing centrally-decided paths.

**5. Failure scenario**
A PCE-initiated policy colliding with local config silently loses — the operator might expect the PCE's
answer to win, but it doesn't.

**6. Design insight**
This dual-direction message model lets one PCE architecture support both hybrid and fully centralized
models on the same protocol.

**7. Interview-ready answer**
"PCEP works through Reports (PCC requests, empty SID list + delegate flag = please compute) and Updates
(PCE's computed path). PCEP Initiate is the reverse — the PCE defines an entire policy and pushes it to a
PCC with no local config. If local and PCE-initiated policies collide on preference/color/endpoint, local
config wins."

---

## 3. SR-TED and Multi-Instance BGP-LS/IGP Distribution

**1. Definition**
The SR-TED is the PCE's consolidated topology graph, merging information from potentially multiple IGP
instances (and BGP-LS) — each source tagged with a distinct instance-id to keep them logically separate
before merging.

**2. Why it exists**
A PCE often needs to compute paths spanning more than one IGP domain. It needs a way to ingest raw
topology data while tracking which pieces came from which source, to correctly stitch or keep them apart.

**3. How it works**
```
#PCE, belonging to both an ISIS and OSPF domain
router isis 1
 distribute link-state instance-id 100
!
router ospf 1
 distribute link-state instance-id 200
```
- No instance-id specified defaults to 0 — only safe with a genuinely single IGP instance.
- Instance ID separates entire IGP **instances**, not areas/levels — all routers in the same instance use
  the same ID.
- The PCE treats all metrics/attributes from different instances as one unified global graph — a real
  issue if different teams manage different instances with non-comparable metrics.
- For nodes in multiple IGPs (ABR/ASBR), the **TE RID must match** across both IGPs so the PCE recognizes
  it as one physical node — but the regular IGP router ID should **differ** to prevent an accidental
  adjacency if the IGPs ever leak into each other.
- `distribute link-state` has an optional throttle (default 50ms ISIS, 5ms OSPF).

**4. Real-world use case**
Any multi-domain SR deployment (large core split into multiple instances, or access/agg/core domains)
needs correctly-scoped instance IDs feeding a central PCE.

**5. Failure scenario**
Reusing the same instance ID across two genuinely separate instances feeding the same PCE causes the merge
logic to incorrectly conflate topology data, producing invalid computed paths.

**6. Design insight**
Instance-ID planning should be a first-class piece of network numbering, documented upfront like IP
addressing or AS numbering.

**7. Interview-ready answer**
"The SR-TED is the PCE's merged topology graph, fed by distribute link-state from each IGP instance,
tagged with a unique instance-id. The instance ID separates entire instances, not areas or levels. For
inter-domain boundary nodes, the TE RID must match across both IGPs so the PCE recognizes one physical
node, while the regular router ID should differ to prevent accidental cross-IGP adjacencies."

---

## 4. ODN with PCE Fallback and the BGP Recursion Fix

**1. Definition**
An ODN policy with the `pcep` keyword creates two candidate paths automatically: preference 200 for local
computation, preference 100 for PCE computation — local is tried first, PCE is the fallback. Separately,
PCE-computed ODN reachability can trigger a BGP "next-hop unreachable" problem needing an explicit fix.

**2. Why it exists**
Not every ODN policy needs inter-domain visibility, so preferring local computation avoids unnecessary
PCEP round-trips. The BGP issue exists because BGP's next-hop validation checks the RIB, which may have
nothing for a remote endpoint reachable only via SR-TE/PCE.

**3. How it works**
```
segment-routing
 traffic-eng
  on-demand color 10
   dynamic
    pcep
    metric
     type igp
```
- **The BGP recursion problem**: with no redistribution between two IGPs (e.g., R1 on ISIS, R7 on OSPF),
  R1 has no RIB route to R7's loopback. Even though the ODN policy comes up successfully with a BSID, BGP
  still flags a "RIB failure" for next-hop reachability.
- **Fix 1 — null0 static route**:
  ```
  #R1
  router static add ipv4 uni 7.7.7.1/32 null0
  #R7
  router static add ipv4 uni 1.1.1.1/32 null0
  ```
  Gives the RIB something to point at, satisfying BGP's check so it recurses via the BSID instead.
- **Fix 2 — newer IOS-XR 7.x knobs**:
  ```
  router bgp 100
   bgp bestpath igp-metric sr-policy
   nexthop validation color-extcomm sr-policy
  ```
  Lets BGP directly recognize a matching colored SR-TE policy as valid reachability, no RIB entry needed.

**4. Real-world use case**
Common in pure "PCE-only reachability" designs, where RIB-based reachability between PEs is deliberately
avoided (scale/isolation reasons), relying entirely on colored ODN policies.

**5. Failure scenario**
An operator sees the ODN policy "up" with a valid BSID and assumes VPN service works, but BGP still shows
"RIB failure" — the policy being up isn't sufficient for BGP to actually recurse through it.

**6. Design insight**
This is a critical layer interaction any architect must account for in a design where SR-TE/PCE is the
only reachability mechanism (no underlying RIB route at all) — an expected, first-class part of such a
design pattern, not an edge-case workaround.

**7. Interview-ready answer**
"ODN with pcep creates two candidate paths — local at preference 200, PCE fallback at preference 100. If a
PE has no RIB-level reachability to a remote endpoint by design, BGP still flags a RIB failure even with
the SR-TE policy up, since BGP's next-hop check looks at the RIB. Fix with a null0 static route, or the
newer bgp bestpath igp-metric sr-policy / nexthop validation color-extcomm sr-policy knobs."

---

## 5. Computation Design Models — Centralized, Distributed, and Hybrid

**1. Definition**
Three architectural models for SR-TE path computation: Centralized (only a controller computes/pushes
every policy), Distributed (every router computes locally, no PCE), Hybrid (routers compute locally when
possible, delegating to a PCE only when necessary).

**2. Why it exists**
Different networks have different scale, control, and operational-philosophy requirements — these three
models represent the spectrum of choices.

**3. How it works**
- Centralized ("vertical"): controller pushes all policies via PCEP Initiate; no local computation at all.
- Distributed: every router computes every path itself, no PCE involvement.
- Hybrid ("horizontal"): routers build/manage policies locally, delegating to a PCE for inter-domain or
  disjoint-path problems. Hybrid is explicitly stated as "generally what is recommended."

**4. Real-world use case**
Most mature production SR-TE deployments land on hybrid — local computation for the bulk of traffic, PCE
reserved for genuinely hard problems.

**5. Failure scenario**
A pure centralized design creates a single point of control for every path — if the controller becomes
unavailable without proper redundancy, the network can't create new paths or react to topology changes.

**6. Design insight**
Hybrid distributes computational load appropriately, avoiding an overloaded PCE bottleneck for trivial
decisions while reserving centralized intelligence for what structurally requires it.

**7. Interview-ready answer**
"Centralized has a controller push every policy with no local computation; distributed has every router
compute locally with no PCE; hybrid computes locally by default, delegating to a PCE only for inter-domain
or disjoint paths. Hybrid is generally recommended, avoiding an unnecessary PCE bottleneck for trivial
decisions."

---

## 6. PCC Features — PCEP Authentication and report-all

**1. Definition**
PCEP sessions can be authenticated with TCP MD5 (option 19). Separately, `report-all` controls whether a
PCC reports policies it computed entirely on its own to the PCE.

**2. Why it exists**
Authentication prevents unauthorized PCEP sessions. report-all exists because, by default, a PCC only
reports policies the PCE was actually involved in — an operator might want the PCE to have full visibility
into every policy for topology awareness or northbound-interface completeness.

**3. How it works**
```
#PCC
segment-routing
 traffic-eng
  pcc
   pce address ipv4 10.10.10.1
    password clear PASSWORD
#PCE
pce
 address ipv4 10.10.10.1
 password clear PASSWORD
```
- By default, a purely local policy is not reported at all.
- `report-all` forces the PCC to report every LSP, including self-computed ones:
  ```
  segment-routing
   traffic-eng
    pcc
     report-all
  ```
- On the PCE, a purely-reported (not PCE-computed) policy shows "Computed path: None."

**4. Real-world use case**
Valuable when the PCE's northbound interface is consumed by external applications needing a complete
picture of all TE activity, not just what the PCE itself computed.

**5. Failure scenario**
Monitoring tooling built against a PCE's northbound interface misses purely-local policies on PCCs where
report-all wasn't enabled — a silent blind spot.

**6. Design insight**
Any design using the PCE as a single source of truth for network-wide visibility must explicitly enable
report-all on every PCC — not automatic just because a session exists.

**7. Interview-ready answer**
"PCEP sessions can use TCP MD5 authentication with a shared password. By default a PCC only reports
policies the PCE was involved in computing — purely local policies are invisible unless you enable
report-all, visible on the PCE as an empty 'Computed path' since the PCE never calculated it."

---

## 7. PCE-Initiated Policies

**1. Definition**
The PCE defines a complete SR-TE policy under its own configuration and pushes it onto a PCC via PCEP
Initiate, without the PCC having any local configuration for that policy.

**2. Why it exists**
In centralized or controller-driven designs, an operator wants to define policy centrally and have it
automatically deployed — the foundational building block for true SDN-style traffic engineering in SR-TE.

**3. How it works**
```
#PCE
pce
 address ipv4 10.10.10.1
 segment-routing
  traffic-eng
   segment-list name R1_TO_R7
    index 1 address ipv4 10.1.4.4
    index 2 address ipv4 6.6.6.1
    index 3 address ipv4 7.7.7.1
   !
   peer ipv4 1.1.1.1
    policy POL1
     color 10 end-point ipv4 7.7.7.1
     candidate-paths
      preference 100
       explicit segment-list R1_TO_R7
```
- Syntax is essentially identical to local policy config, just located under the PCE's peer-specific
  config.
- R10 pushes via PCEP Initiate; R1 replies with a Report to confirm installation.
- R1's own SR-TE config shows zero locally-configured policies — it exists purely via PCEP push.
- No visible difference in show output between PCE-initiated and PCC-requested policies.
- **Tie-break**: local configuration always wins over PCE-initiated on a collision.

**4. Real-world use case**
Standard building block for controller-driven TE — an orchestration platform computing optimal paths and
pushing directly with zero manual router configuration.

**5. Failure scenario**
An operator unaware a controller manages a color+endpoint manually configures the same locally for
testing — the local config silently wins, and the controller's path is ignored.

**6. Design insight**
Treat "no local SR-TE config for controller-managed colors" as a hard operational rule to avoid this
ambiguous, hard-to-debug conflict.

**7. Interview-ready answer**
"A PCE can define a full policy under its own peer-specific config and push it via PCEP Initiate with zero
local router configuration — the foundation for centralized, SDN-controller-driven TE. If a local policy
collides with a PCE-initiated one on the same color/endpoint/preference, local always wins."

---

## 8. PCE Redundancy — Precedence Election and Failover

**1. Definition**
A PCC can maintain sessions with multiple PCEs, electing one primary based on precedence (lowest wins),
with automatic re-delegation to the next-best PCE on primary loss.

**2. Why it exists**
If headends depend on a PCE for inter-domain or disjoint paths, losing that single PCE could mean losing
the ability to maintain/recompute those LSPs — redundancy is essential wherever PCE is mandatory.

**3. How it works**
```
#PCC
segment-routing
 traffic-eng
  pcc
   pce address ipv4 9.9.9.1
    precedence 200
   pce address ipv4 10.10.10.1
    precedence 100
```
- Tie: lowest PCE IP wins. Default precedence is 255 (worst).
- Keepalive/dead timers are NOT negotiated (unlike BGP), can be asymmetric:
  ```
  #PCC
  segment-routing
   traffic-eng
    pcc
     timers keepalive 15
     timers deadtimer 45
  ```
  - PCE's dead interval is always fixed at 120 seconds.
  - PCE has a minimum-peer-keepalive (default 20s) — if the PCC's configured keepalive is below this, the
    session never comes up:
    ```
    #PCE
    pce
     timers
      minimum-peer-keepalive 15
    ```
- PCC uses next-hop tracking on the PCE address — session drops immediately if the PCE's address leaves
  the RIB.
- Failover: PCC sends Reports to every connected PCE but only sets the D flag toward the primary. Only the
  primary responds with an Update; standbys learn of the policy but don't own its computation.
- On primary loss, the PCC immediately re-delegates to the next-best PCE.
- Orphan paths (can't be re-delegated) are invalidated after delegation-timeout (default 60s; 0 = retain
  indefinitely):
  ```
  #PCC
  segment-routing
   traffic-eng
    pcc
     timers delegation-timeout <0-3600>
  ```
- **PCE-initiated policies fail over less gracefully**: they live purely as PCE-side config; a standby has
  no way to re-push a policy it never itself created. Uses much longer timers: re-delegation after 3
  minutes, full timeout after 10 minutes:
  ```
  #PCC
  segment-routing
   traffic-eng
    pcc
     timers initiated orphan <10-180 sec>
     timers initiated state <15-14400 sec>
  ```

**4. Real-world use case**
Any production network depending on a PCE for inter-domain/disjoint paths should deploy at least two PCEs
as a near-mandatory recommendation.

**5. Failure scenario**
An operator assumes PCE redundancy protects all policy types equally, not realizing PCE-initiated policies
are structurally more fragile on primary loss with much longer recovery timers.

**6. Design insight**
PCC-delegated policies get clean, fast failover for free; PCE-initiated policies need either a redundant
orchestration layer above the PCEs or acceptance of longer recovery timers — weigh this explicitly when
choosing hybrid vs. centralized.

**7. Interview-ready answer**
"A PCC elects a primary PCE by precedence (lowest wins), delegating there while reporting to all connected
PCEs so standbys stay aware. On primary loss (via next-hop tracking), the PCC immediately re-delegates.
PCE-initiated policies fail over much less gracefully, since a standby has no independent way to re-push a
policy it never created — they use much longer recovery timers."

---

## 9. PCE Redundancy with State-Sync — Master/Slave and Disjoint Path Coordination

**1. Definition**
PCEs can form direct PCEP sessions with each other (state-sync), forwarding PCEP Reports to each other —
adding state consistency beyond PCC-driven reporting, and enabling correct disjoint-path computation when
different PCCs use different primary PCEs.

**2. Why it exists**
PCC-driven reporting has gaps: (1) if a PCC loses just one PCE session while that PCE stays healthy, it
never gets reports; (2) cross-PCE disjoint-path coordination can require modifying a path delegated to a
different PCE than the one handling the new request — impossible without direct PCE sync.

**3. How it works**
```
#PCE1
pce
 address ipv4 9.9.9.1
 state-sync ipv4 10.10.10.1
#PCE2
pce
 address ipv4 10.10.10.1
 state-sync ipv4 9.9.9.1
```
- PCEs cannot update/instantiate policies on each other — only forward Reports.
- A PCE clears the delegate flag, tags the originating PCC, and forwards received reports to synced
  peers — but never re-forwards a report received from another PCE (iBGP split-horizon behavior), so a
  full mesh of sync sessions is required.
- **Master/slave**: the PCE with the lowest session IP becomes master. A slave receiving a disjoint-path
  request it can't fully resolve (needing to modify a policy delegated elsewhere) sub-delegates the whole
  computation to the master, which computes both paths and reports back.
- Verified: nulling a PCC's route to one PCE still lets that PCE learn the PCC's new policy via the sync
  session with the other PCE.

**4. Real-world use case**
Any network with different PCCs using different primary PCEs, combined with disjoint-path requirements
across those PCCs, requires this sync mechanism.

**5. Failure scenario**
An operator spreads PCC-to-PCE primary assignments for load-balancing but forgets the sync session —
cross-PCC disjoint-path requests silently fail to compute correctly.

**6. Design insight**
Having "PCE redundancy" configured is not sufficient for correct disjoint-path behavior across different
primaries — the sync session is a distinct, additional requirement.

**7. Interview-ready answer**
"PCEs can sync directly, forwarding — but never re-forwarding — PCC reports, mimicking iBGP split-horizon
and requiring a full mesh. This closes the single-session-loss gap and solves cross-PCC disjoint-path
computation via a master/slave relationship, where the slave sub-delegates to the master when it lacks
delegated control over both PCCs' policies needed to satisfy the constraint."

---

## 10. BGP Egress Peer Engineering (EPE) Fundamentals

**1. Definition**
BGP EPE lets an operator traffic-engineer past their own egress PE, controlling exactly which eBGP peer
(and even which specific link) is used to exit the AS — impossible with normal BGP attributes, since the
egress PE always makes its own independent bestpath decision.

**2. Why it exists**
Normal BGP TE can influence which egress PE your AS uses, but once traffic arrives there, that PE's own
eBGP bestpath process takes over with no standard mechanism for upstream influence. Real problem for
performance-based engineering, since BGP has no concept of latency at all.

**3. How it works**
- Egress PE allocates a peering SID representing the eBGP adjacency — like an IGP Adj-SID but for a BGP
  peering relationship.
- Three types: **Peer Node SID** (pop and forward to the peer via ECMP if applicable), **Peer Adj SID**
  (one per link, automatically allocated for multihop sessions even with a single link), **Peer Set SID**
  (a group of peers — not supported on IOS-XR).
  ```
  #Egress PE
  router bgp 100
   neighbor 10.6.7.7
    egress-engineering
  ```
- The remote eBGP peer has zero awareness of any of this.
- Peer SIDs are local, dynamically-allocated, typically distributed via BGP-LS to a PCE.
- Example segment-list:
  ```
  segment-list EPE
   index 1 mpls label 16006
   index 2 mpls label 24008
  ```
  Forces arrival at the egress PE, then forwarding to the specific eBGP peer via the peer SID —
  bypassing the egress PE's own bestpath logic.

**4. Real-world use case**
(1) Traffic-engineering egress peer/link selection for performance reasons BGP can't express; (2) an
elegant alternative to BGP-LU for inter-domain unified MPLS with full TE capability.

**5. Failure scenario**
Attempting to influence the egress PE's own peer selection via upstream weight (as in the lab's initial
setup) simply fails, since weight only affects the local router's own bestpath decision.

**6. Design insight**
EPE is a fundamentally different category of TE — dictating a downstream router's forwarding decision, one
hop past where BGP's normal influence ends.

**7. Interview-ready answer**
"BGP EPE lets you dictate exactly which eBGP peer, even which link, an egress PE uses — something normal
BGP attributes can't do since those only influence your own bestpath decision. EPE works via peering SIDs
(Peer Node SID for the peer as a whole, Peer Adj SID per link for multihop) that an SR-TE explicit path can
push as the final label, bypassing the egress PE's own bestpath logic entirely."

---

## 11. BGP-LS for Multi-Domain Unified MPLS with EPE

**1. Definition**
BGP-LS (AFI/SAFI link-state/link-state) carries IGP topology and BGP EPE peering information as ordinary
BGP updates — letting a PCE learn the entire multi-domain topology as one unified graph.

**2. Why it exists**
A PCE needs full visibility for inter-domain paths, but IGP link-state is normally confined to a single
instance/area. BGP-LS re-encodes topology into a common format, transported over BGP's naturally
domain-spanning reach.

**3. How it works**
```
#IGP edge router
router isis 1
 distribute link-state instance-id 101
!
router bgp 65001
 address-family link-state link-state
 neighbor 10.10.10.1
  remote-as 65010
  update-source lo1
  ebgp-multihop
  address-family link-state link-state
```
- **Three NLRI types**: Type 1 Node [V] (hostname, area, RID, SRLB, SRGB, algos, MSD); Type 2 Link [E]
  (local/remote RID, IGP/TE metric, admin group, Adj-SID); Type 3 IPv4 Prefix [T] (prefix, metric, flags,
  SID index).
- ISIS and OSPF both transcode into the same common NLRI format — the PCE's graph doesn't care which
  protocol originated a piece of topology.
- BGP-EPE peer SIDs appear as link-type NLRI between ASBRs — **EPE links carry no metric** (treated as 0).
- **Critical requirement**: the BGP router ID must exactly match the IGP TE Router ID, or the PCE can't
  recognize the BGP node and the IGP node as the same physical router.
- **MPLS must be manually enabled**: egress-engineering does not turn on MPLS forwarding on the
  peer-facing interface. Use `mpls static` or `mpls traffic-eng`; `mpls activate` under BGP does not work
  since no labeled AFI is exchanged.
- Topology changes propagate immediately as NLRI withdrawals (verifiable via `show pce verification`).
- BGP-LS troubleshooting follows normal BGP bestpath rules; an unreachable NLRI next-hop is simply invalid.

**4. Real-world use case**
The modern replacement for classic BGP-LU-based unified MPLS, adding full constraint-based TE capability
that BGP-LU alone cannot offer.

**5. Failure scenario**
Because EPE links are metric-0, a topology change elsewhere can cause the PCE to compute a bizarre "figure
-8" path repeatedly crossing zero-cost EPE links — demonstrated directly, producing a 5-deep label stack.

**6. Design insight**
Use hopcount (not IGP/TE metric) as the optimization type for any policy that might traverse EPE segments,
since hopcount naturally penalizes unnecessary extra transits the way a metric calculation won't.

**7. Interview-ready answer**
"BGP-LS re-encodes IGP topology into three NLRI types carried over BGP, letting a PCE build one unified
graph spanning multiple IGP domains and inter-AS EPE peering points. It automatically carries EPE peering
SID info too, but EPE links always show as metric-0, which can cause bizarre figure-8 paths after a
topology change — fixed by optimizing on hopcount instead of IGP/TE metric."

---

## 12. SR-TE Disjoint Paths

**1. Definition**
Path disjointness is a PCE-computed constraint ensuring two policies sharing a disjoint group-id never
share links, nodes, or SRLG groups, depending on the chosen type.

**2. Why it exists**
Two independently-configured policies can easily compute to the exact same shortest path. True redundancy
requires computation aware of the other path, which structurally requires a PCE.

**3. How it works**
```
segment-routing
 traffic-eng
  policy POL_1_AS
   candidate-paths
    preference 100
     constraints
      disjoint-path group-id 10 type node
```
- **Four types**: link (different links, may share nodes), node (different nodes, implies link-disjoint),
  srlg (different SRLG groups, may share nodes), srlg-node (both combined).
- **Computation flow**: first request for a group-id gets the best path normally. Second request with the
  same group-id triggers joint recomputation of both — potentially modifying the first, though the lab
  shows the PCE preferring to leave the first-configured policy untouched and giving the second the worse
  path.
- **Hard limit**: only two LSPs supported per disjoint group-id, even if the topology could support more —
  a third policy with the same group-id simply fails to come up.
- **Automatic fallback**: if the requested type can't be satisfied, silently falls back to a lower type
  (node/SRLG -> link -> no disjointness at all), visible only via PCE syslog.
- **Strict mode** prevents silent fallback:
  ```
  #PCE
  pce
   disjoint-path
    group-id 10 type node
     strict
  ```
  With strict mode, an unsatisfiable constraint makes the policy fail outright instead of degrading
  silently. Note: if both policies terminate on the same endpoint node, true node-disjointness can never
  be satisfied — a fundamental topological reality, not a bug.
- **Controlling which policy gets the best path**: by default, first-configured gets the better path.
  Override directly on the PCE:
  ```
  #PCE
  pce
   disjoint-path
    group-id 10 type link
     strict
     lsp 1 pcc ipv4 2.2.2.1 lsp-name cfg_R2_TO_R7_discr_100 shortest-path
     lsp 2 pcc ipv4 1.1.1.1 lsp-name cfg_R1_TO_R7_discr_100
  ```
  Must use the full internal symbolic name, not the shorter configured name; only two LSP entries accepted.
- **Sub-ID**: subdivides a group-id into further disjoint groups, reusing an IP-address-formatted PCEP
  field — noted as not especially useful given ample available group-id values.

**4. Real-world use case**
Essential for dual-homed/dual-path redundant services needing genuine physical/logical path independence,
especially premium services with strict availability SLAs.

**5. Failure scenario**
A node-disjointness request between policies terminating on the same node, without strict mode, silently
falls back to link-disjointness (or nothing) — the operator believes they have node-level redundancy when
they may have none at all.

**6. Design insight**
The two-LSP-per-group-id limit is a real scaling constraint requiring an entirely different design approach
for 3+ mutually disjoint paths. Strict mode should be the default for any real disjointness requirement.

**7. Interview-ready answer**
"Disjoint-path uses a shared group-id to make the PCE compute two policies together — link, node, SRLG, or
srlg-node disjoint. Only two LSPs are supported per group-id, and by default the PCE silently falls back
to a weaker type if the requested one can't be satisfied, unless you enable strict mode on the PCE, which
fails the policy outright instead — strict mode should be the default for any real requirement."

---

## 13. Anycast SID for ABRs with Conditional Advertisement (Converged SDN Transport Pattern)

**1. Definition**
A pair of ABRs connecting an access domain to an aggregation/core domain share a common Anycast SID on a
dedicated loopback, combined with conditional prefix advertisement — the ABR withdraws its Anycast SID
advertisement into the access domain if it loses its own upstream connectivity.

**2. Why it exists**
Basic Anycast SIDs handle a full ABR failure via IGP reconvergence, but not a "half-open" failure: an ABR
healthy on the access side but with a dead uplink would still attract traffic via its Anycast SID, only to
black-hole it. Conditional advertisement ties the downstream advertisement to upstream health.

**3. How it works**
- Dedicated loopback, since a single loopback can only carry one SID per algorithm:
  ```
  #ABR pair
  int lo56
   ip add 10.0.0.56/32
  router isis ACCESS-1
   interface Loopback56
    address-family ipv4 unicast
     prefix-sid index 1056 n-flag-clear
  router isis AGG-1
   interface Loopback56
    address-family ipv4 unicast
     prefix-sid index 1056 n-flag-clear
  ```
- Conditional advertisement:
  ```
  route-policy AGG-1-PREFIXES
   if rib-has-route async (10.0.0.1/32, 10.0.0.3/32) then pass endif
  end-policy
  !
  router isis ACCESS-1
   int lo56
    add ipv4 unicast
     advertise prefix route-policy AGG-1-PREFIXES
  ```
- The `async` keyword means event-driven RIB notification — if the referenced upstream route disappears,
  the ABR immediately stops advertising the Anycast loopback downstream.
- This technique is architecturally general-purpose (could apply to regular loopbacks too), though the
  official design guide only documents it for the Anycast loopback.
- **Hidden constraint**: when an Anycast SID is used in PCE path computation, only the IGP metric type is
  supported.

**4. Real-world use case**
The standard Cisco reference design for resilient access-to-aggregation boundaries in multi-domain SP
cores.

**5. Failure scenario**
Without conditional advertisement, a healthy-looking ABR with a dead uplink continues attracting
access-domain traffic via its Anycast SID, black-holing it on arrival.

**6. Design insight**
Redundancy mechanisms must be tied to the health of the thing actually being relied upon, not just the
health of the node advertising it — a general CCDE principle.

**7. Interview-ready answer**
"ABR pairs share an Anycast SID on a dedicated loopback, since one loopback can only carry one SID per
algorithm. Basic Anycast handles a full ABR failure, but not a half-open failure where the ABR is healthy
downstream but has lost its upstream uplink — conditional advertisement solves this by tying the Anycast
loopback's advertisement to the presence of specific upstream routes in the RIB via an event-driven async
check."

---

## 14. iBGP Underlay Design for Loopback Reachability in Multi-Domain SR (Converged SDN Transport Pattern)

**1. Definition**
Rather than direct IGP-to-IGP redistribution, ABRs run iBGP to carry access-domain and PCE/RR loopbacks
between domains, with route-tagging to ensure each ABR prefers its own directly-learned BGP route over a
redistributed IGP route leaked from its peer ABR.

**2. Why it exists**
Direct IGP redistribution is explicitly not recommended — iBGP gives finer control and avoids classic
redistribution risks (loops, unpredictable metric translation). No label is needed since this is purely
for BGP/PCEP session reachability, not service traffic.

**3. How it works**
```
#ABR
router bgp 100
 ibgp policy out enforce-modifications
 bgp redistribute-internal
 address-family ipv4 unicast
  network 10.0.0.9/32
 !
 neighbor-group AGG-CORE-ABR
  remote-as 100
  update-so lo0
  address-family ipv4 unicast
   next-hop-self
```
- `ibgp policy out enforce-modifications` and `bgp redistribute-internal` relax iBGP's normal
  re-propagation restrictions, needed since this iBGP session feeds subsequent IGP redistribution.
- **Route-tagging fix**: without it, an ABR might prefer a redistributed IGP version of a route (leaked
  from its peer ABR) over its own fresher, directly-learned BGP version, due to admin distance:
  ```
  route-policy SET_TAG($TAG)
   set tag $TAG
  end-policy
  !
  route-policy DENY_TAG($TAG)
   if not tag is $TAG then pass endif
  end-policy
  !
  router isis ACCESS-1
   add ipv4 uni
    redistribute bgp 100 route-policy SET_TAG(101)
    distribute-list route-policy DENY_TAG(101) in
  ```
- The PCE also joins this iBGP mesh to learn aggregation-domain ABR loopbacks for BGP-LS session
  establishment, again via iBGP rather than direct IGP-to-IGP redistribution.

**4. Real-world use case**
Standard reference-design pattern for the underlay reachability layer connecting PCE/RR, access, and
aggregation/core loopbacks in a large multi-domain SR deployment.

**5. Failure scenario**
Without tagging, an ABR could prefer a stale redistributed IGP route over its own fresher BGP route purely
due to admin distance, causing subtly incorrect PCEP/BGP session behavior.

**6. Design insight**
"Just redistribute between the IGPs" is discouraged precisely because it hides this class of subtle
route-preference issues — BGP's richer route-control toolset gives far more precise, predictable control.

**7. Interview-ready answer**
"Instead of redistributing loopbacks directly between IGPs, ABRs run iBGP to carry access and PCE/RR
loopbacks across domain boundaries — no label needed since it's purely for session reachability. The key
gotcha is an ABR preferring a stale IGP-redistributed route leaked from its peer over its own fresher BGP
route, purely due to admin distance — solved by tagging on redistribution and denying re-acceptance of
that tag back into the RIB."

---

## 15. Performance Measurement — Interface Delay (TWAMP-light)

**1. Definition**
Performance-Measurement uses TWAMP-light to actively measure real, live delay on a link, which can be
flooded into the IGP (for SR-TE latency-optimized policies) and streamed to a telemetry collector via MDT.

**2. Why it exists**
Latency-optimized SR-TE policies need actual measured delay values, not a static operator-configured
metric. This feature provides that real data.

**3. How it works**
- **Two-way mode (default)**: no clock sync needed, since it only compares timestamps of the same clock:
  `(T4-T1) - (T3-T2)`, divided by two for one-way estimate.
- **One-way mode**: requires PTP-synchronized clocks, comparing `T2-T1` across two different clocks.
- **Defaults**: probe every 3s; compute every 30s (10 probes); IGP advertisement evaluated every 120s,
  needing both 10% and 1ms change; accelerated advertisement disabled by default.
- **Custom config** (10 probes/20s; periodic flood every 4 computations if >=5%/500us; accelerated flood
  immediately if >=15%/1ms):
  ```
  performance-measurement
   interface GigabitEthernet0/0/0/0
    delay-measurement
     delay-profile name DELAY_PROFILE1
   !
   delay-profile interfaces name DELAY_PROFILE1
    advertisement
     accelerated
      minimum-change 1000
      threshold 15
     !
     periodic
      minimum-change 500
      interval 80
      threshold 5
     !
    !
    probe
     tx-interval 2000000
     computation-interval 20
  ```
  - `tx-interval` in microseconds (2,000,000 = 2s); `computation-interval` in seconds.
  - `periodic interval 80` = 4x the 20s computation-interval.
  - `minimum-change` under periodic/accelerated is in microseconds — a "1ms" tolerance requires entering
    1000, not 1 (an easy silent misconfiguration).
  - Accelerated evaluates after every single computation and floods immediately if triggered, bypassing
    the periodic wait entirely.

**4. Real-world use case**
Required to feed any SR-TE latency-optimized policy with real data; also independently valuable for
telemetry/observability via MDT streaming.

**5. Failure scenario**
Overly-aggressive thresholds/intervals cause every minor delay fluctuation to flood the IGP repeatedly,
generating unnecessary reconvergence overhead network-wide.

**6. Design insight**
The two-tier threshold system (periodic + accelerated) is a deliberate trade-off dial between
responsiveness and control-plane churn — tighter thresholds give more accurate SR-TE latency data at the
cost of more frequent IGP updates.

**7. Interview-ready answer**
"Performance-Measurement uses TWAMP-light to measure real link delay — two-way by default needing no clock
sync, or one-way requiring PTP. Measured values feed latency-optimized SR-TE via a two-tier IGP
advertisement system: periodic checks every N computations requiring both a percentage and absolute
change, and accelerated checks after every single computation for large jumps needing immediate flooding."

---

## Quick-Reference Summary Table

| # | Concept | Key Mechanism | Hidden Detail / Risk |
|---|---|---|---|
| 1 | PCE fundamentals | Stateful PCC/PCE, SR-TED | Missing TE RID = node invisible to PCE |
| 2 | PCEP workflow | Report/Update/Initiate | Local config wins tie vs. PCE-initiated |
| 3 | SR-TED/multi-instance | distribute link-state, instance-id | TE RID must match across IGPs; RID must differ |
| 4 | ODN + PCE fallback | pref 200 local / pref 100 PCE | BGP RIB-failure needs null0 or sr-policy knobs |
| 5 | Computation models | Centralized/Distributed/Hybrid | Hybrid recommended; pure centralized = SPOF |
| 6 | PCC features | MD5 auth, report-all | Local-only policies invisible without report-all |
| 7 | PCE-initiated policies | PCEP Initiate, peer config | Local config always wins a tie |
| 8 | PCE redundancy | Precedence election | PCE-initiated fails over far less gracefully |
| 9 | PCE sync | state-sync, master/slave | Full mesh required; solves cross-PCC disjointness |
| 10 | BGP EPE | Peer Node/Adj/Set SID | Remote peer needs zero SR awareness |
| 11 | BGP-LS unified MPLS | NLRI types 1/2/3 | EPE links = metric 0, causes figure-8 paths |
| 12 | Disjoint paths | Shared group-id | Only 2 LSPs/group; silent fallback without strict |
| 13 | Anycast ABR + cond. adv | async RIB-triggered withdrawal | Fixes "healthy but uplink-dead" black hole |
| 14 | iBGP underlay | Route tagging vs redistribution | Prevents stale IGP route from beating fresh BGP |
| 15 | Performance measurement | TWAMP-light, 2-tier thresholds | minimum-change is in microseconds, not ms |

