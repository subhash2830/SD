# CCDE Segment Routing — Quick Reference (All Concepts, Short Form)
### Condensed summaries (4-5 lines each) of every concept from Parts 1-5
### Compiled: 2026

---

## PART 1 — SR Core: SRGB, PHP, Explicit-Null, Anycast SID

**1. SRGB (Segment Routing Global Block)**
The local label range (default 16000-23999) a node reserves for global Prefix-SIDs. A node's label for a
prefix = SRGB base + index, and the SRGB itself is flooded via the IGP as a capability. Best practice is
keeping SRGB identical across all nodes so labels are globally predictable; non-uniform SRGBs still work
via normal downstream label-swapping but complicate SR-TE and troubleshooting.

**2. SR Label Space Architecture**
IOS-XR splits the label space into Reserved (0-15), Static (16-14999), SRLB (15000-15999, for local SIDs
like binding-SIDs), SRGB (16000-23999, global Prefix-SIDs), and Dynamic (24000+, for LDP/RSVP/BGP-LU).
Moving the SRGB fragments the dynamic range around it. Must be sized upfront like address planning.

**3. Prefix-SID Index vs Label & SRGB Mismatch**
Prefix-SIDs are flooded as an index, not a label; each router computes its own label as SRGB+index. MPLS
forwarding still obeys downstream allocation — a router pushes the label its next-hop expects, so
differing SRGBs just become normal label swapping performed via computation instead of signaling.

**4. PHP (Penultimate Hop Popping)**
The second-to-last router pops the top label before the final hop, so the destination only does one
lookup instead of pop-then-lookup. Signaled by the P-flag (No-PHP) in the Prefix-SID sub-TLV, default
clear (PHP allowed). Trade-off: PHP strips MPLS EXP bits one hop early, breaking EXP-based QoS at the edge.

**5. Explicit-Null**
An alternative to PHP where the penultimate router swaps the top label for the reserved Explicit-Null
label (0/2) instead of popping, preserving EXP bits to the true final hop. Requires both the P-flag
(No-PHP) and E-flag set together in the Prefix-SID sub-TLV — SR has no signaling protocol, so this whole
behavior is encoded purely as flags on flooded IGP state.

**6. ISIS SR Advertisement (Router Capability TLV, Prefix-SID Sub-TLV)**
SR extends existing ISIS TLVs rather than inventing a new protocol: Router Capability TLV (242) carries
SR-Capability (SRGB) and SR-Algorithm sub-TLVs; Prefix-SID sub-TLV (Type 3) carries flags (R/N/P/E/V/L) and
rides inside the normal prefix-reachability TLVs. Unsupported routers simply ignore unknown sub-TLVs.

**7. Anycast SID**
A Prefix-SID configured identically on 2+ nodes with the N-flag cleared, representing "any member of this
group" rather than one specific device — used to steer SR-TE toward a redundant pair as one segment.
Forgetting to clear the N-flag doesn't break normal ECMP forwarding, making it a dangerous, easy-to-miss
misconfiguration.

**8. N-flag and TI-LFA Interaction**
TI-LFA uses the N-flag to decide if it's safe to chain a node-specific Adjacency-SID onto a Prefix-SID
during repair-path computation. If an anycast SID incorrectly keeps N-flag set, TI-LFA may build a repair
path assuming one specific member, then ECMP sends traffic to the other member instead — causing drops or
mis-forwarding during the exact failure event TI-LFA was meant to protect against.

---

## PART 2 — SR-IGP/BGP Fundamentals: ISIS, OSPF, Adjacency-SID, Inter-domain

**1. Why SR Replaces LDP/RSVP-TE**
SR eliminates LDP's separate signaling session and IGP-sync problem, and eliminates RSVP-TE's soft-state
refresh, full-mesh requirement, and lack of ECMP-awareness, by carrying all label info inside existing IGP
flooding. Any node computes an explicit label-stack path with zero extra protocol state; TI-LFA replaces
RSVP-TE FRR without signaling.

**2. Prefix-SID Config: Index vs Absolute**
Prefix-SIDs can be configured as an index or an absolute label (silently converted to an index). If the
SRGB later moves so the absolute value falls outside the new range, the SID silently stops being
advertised with no error — best practice is always index-based configuration.

**3. ISIS Prefix-SID Sub-TLV Flags (R/N/P/E/V/L)**
R = re-advertised across levels, N = true node SID, P = No-PHP, E = Explicit-Null (needs P), V/L = value
/local (should never appear on a genuine Prefix-SID). P and E are separate bits because No-PHP and
Explicit-Null answer genuinely different questions — you often need No-PHP without Explicit-Null.

**4. ISIS Router Capabilities TLV 242 (RID, SR-Algo, MSD)**
Carries node-scoped SR state: RID (priority: MPLS-TE RID > ISIS RID > lowest loopback > lowest interface),
SRGB via SR-Capability, supported algorithms via SR-Algorithm, and Max SID Depth (MSD) — the deepest label
stack a node's hardware can forward, which any PCE/SR-TE headend must respect.

**5. SR Algorithms — Algo 0 vs Algo 1 (Strict SPF)**
Algo 0 follows the IGP shortest path but allows local policy (like an RSVP-TE tunnel) to override it at any
hop. Algo 1 (strict SPF) guarantees the literal IGP path with zero possibility of diversion. A prefix can
advertise both simultaneously as two separate Prefix-SIDs for flexibility vs. guarantee.

**6. Adjacency-SID Fundamentals**
A locally-significant label meaning "pop and forward out this specific link." Every adjacency gets two
dynamically-allocated Adj-SIDs — non-FRR (always advertised) and FRR-eligible (only advertised once TI-LFA
is enabled on that interface) — because the label itself signals whether backup protection exists.

**7. Adjacency-SID Flags, Persistence, SRLB**
Flags: F (address-family), B (backup/FRR), V/L (always set — Adj-SIDs are always local explicit values), S
(unsupported), P (persistent). Dynamic Adj-SIDs are only stable for 30 minutes/until reboot; anything
needing a guaranteed value must be statically allocated from the SRLB.

**8. LAN Adjacency-SID (ISIS vs OSPF)**
ISIS decomposes the shared pseudonode adjacency into one LAN-Adj-SID per neighbor (DIS advertises none);
these are FRR-eligible but NOT actually TI-LFA-protected. OSPF advertises a normal Adj-SID toward the DR
but LAN-Adj-SIDs toward every BDR/DROTHER, with the DR carrying the most state.

**9. OSPF SR Control Plane (Opaque LSAs 1/4/7/8)**
Type 1 (TE info, auto-generated with SR), Type 4 (router SR capabilities — SRGB/SRLB/MSD/algos/hostname),
Type 7 (per-prefix Prefix-SID), Type 8 (per-link Adj-SID). Deliberately separate from classic MPLS-TE LSAs
so SR can run fully independent of RSVP-TE.

**10. OSPF vs ISIS — Structural Differences**
sr-prefer is separate in OSPF, a keyword in ISIS. OSPF allows TI-LFA at any level with inheritance; ISIS
only interface-level. OSPF prefers higher tiebreaker index, ISIS prefers lower. Only ISIS currently
supports IPv6 for SR-MPLS and SRv6 (SR isn't supported on OSPFv3).

**11. OSPF Prefix-SID Flags (A/N + NP/M/E/V/L)**
Extended Prefix TLV: A (attached to ABR), N (node SID). Nested SID sub-TLV: NP (No-PHP), M
(Mapping-Server — SID assigned centrally, no ISIS equivalent, enables brownfield rollout), E, V, L.

**12. OSPF Adjacency-SID Flags (B/V/L/S)**
B = backup/FRR, V/L always set by construction (Adj-SIDs are inherently local/explicit), S unsupported. In
practice only two flag-byte values are ever seen: 0xE0 (protected) or 0x60 (unprotected).

**13. SR/RSVP-TE Interaction (Algo 0 Hijacking)**
An RSVP-TE tunnel with autoroute-announce silently captures Algo-0 SR traffic at any transit node it
exists on. Fix: advertise a second, Algo-1 (strict-SPF) Prefix-SID at the destination — no changes needed
on the misbehaving transit node at all.

**14. Inter-Area SR Propagation (ISIS)**
Works automatically once normal route propagation is configured. The ABR/L1L2 router sets No-PHP (it
isn't the true destination) and R (re-advertisement) but NOT Explicit-Null (no QoS need) — this is exactly
why No-PHP and Explicit-Null are separate flags.

**15. OSPF Inter-Area SR**
Marks Prefix-SIDs with route-type 3 (inter-area), sets No-PHP without Explicit-Null, and inherits OSPF's
existing Type-3 loop-prevention rule (an ABR never re-injects a non-backbone route back into another area)
automatically — zero extra loop-prevention logic needed.

**16. Inter-IGP SR Redistribution**
RIB-driven: the ASBR pulls the resolved label from the RIB and converts it back to an index for the other
protocol — but only if both IGPs share the same SRGB. If not, the prefix redistributes at the IP layer but
silently loses its label. Also needs the ASBR's own loopback natively advertised with a Prefix-SID in both
protocols independently.

**17. SR for BGP (BGP Prefix-SID Attribute)**
BGP-LU plus an optional-transitive Prefix-SID attribute carrying a label-index — nodes with a globally
defined SRGB honor it as a hint to allocate SRGB+index instead of a dynamic label, fully interworking with
classic BGP-LU. Danger: mixing IGP-SR with classic (non-SR) BGP-LU on the same box causes an LFIB
collision and BGP's label allocation fails outright.

**18. BGP-SR Operational Pitfalls**
PEs need allocate-label all even for routes they don't originate (enables VPN-route recursive lookup);
direct eBGP peering between PEs on a labeled AFI always triggers local Pop-label allocation regardless of
any existing SR label (fix: peer via RR or use ebgp-multihop mpls); stale duplicate dynamic-label bugs
require a full BGP process removal-and-readd to clear.

---

## PART 3 — Fast Reroute: LFA, RLFA, TI-LFA

**1. LFA Fundamentals**
A pre-computed backup next-hop, validated by the inequality Dist(N,D) < Dist(N,PLR) + Dist(PLR,D) to
guarantee the backup won't loop traffic back. Gives sub-50ms protection with zero signaling, but coverage
is entirely topology-dependent — many prefixes have no valid LFA neighbor at all.

**2. LFA Verification Flags (P/NP/TM/LC/D/SRLG)**
Show output flags describing exactly what protection a backup gives (node-protecting, line-card-disjoint,
downstream, SRLG-disjoint). Seeing NP=Yes with plain LFA doesn't mean you configured node protection — it
may just be topology coincidence; only tiebreakers guarantee it.

**3. Excluding Interfaces from LFA**
A hard, absolute exclusion (stronger than SRLG-disjoint preference) — an excluded interface is never used
as backup, even if it means losing protection entirely. Reserved for genuine hard constraints like
bandwidth or regulatory segregation.

**4. LFA Tiebreakers (ISIS)**
Rules deciding which backup wins when multiple exist: primary-path, lowest-backup-metric, line-card-
disjoint, node-protecting, SRLG-disjoint, secondary-path, downstream — evaluated in index order. Default
prefers cheapest cost, which can silently pick a non-node-protecting backup.

**5. Remote LFA (RLFA)**
Tunnels traffic via LDP to a PQ node (in both the PLR's P-space and the destination's Q-space), forcing an
MPLS decision instead of an IP one at the intermediate neighbor. Dangerous gap: if the PQ node doesn't
accept targeted LDP, the backup path still comes up but with only one label — silently breaking end-to-end
MPLS service LSPs while plain IP looks fine.

**6. Remote LFA Limitations**
RLFA structurally cannot provide node or SRLG protection (proven — tiebreakers have zero effect), doesn't
use the true post-convergence path, requires targeted LDP sessions everywhere, and doesn't guarantee full
coverage. These are hard ceilings, not tuning gaps.

**7. Extended P-Space**
Lets an RLFA-protecting router borrow its own neighbor's P-space (since it's tunneling via MPLS, not
making an IP decision), finding a valid PQ node even when the router's own P-space doesn't overlap the
Q-space. Feasibility depends on a metric threshold that can be silently crossed by unrelated metric
changes elsewhere.

**8. TI-LFA Fundamentals**
Removes the protected link/node/SRLG from the topology, runs CSPF to find the genuine post-convergence
path, then builds the shortest SID list using Adjacency-SIDs to force traffic along it — giving 100%
topology coverage, unlike LFA or RLFA, and can even protect plain LDP/IP traffic if SR is enabled purely
for backup.

**9. TI-LFA's Prefix-SID Dependency for Intermediate Nodes**
TI-LFA needs a Prefix-SID for every intermediate node it might route through: ISIS uses TE Router ID TLV
134 (falling back to 132), OSPF checks the RID then the highest N-flagged host prefix. This is exactly why
an anycast SID with N-flag left set can silently break TI-LFA.

**10. TI-LFA's Adjacency-SID Selection (Unprotected by Design)**
When a Q-node has both protected and unprotected Adj-SIDs for a link, TI-LFA deliberately uses the
unprotected one — intentional, to avoid microloops if a second, overlapping failure hits the same link
during an active repair.

**11. TI-LFA Node Protection**
By default TI-LFA only protects against link failure. Real node protection requires the node-protecting
tiebreaker, which tries computing a backup with the entire next-hop node excluded, falling back to
link-only protection if none exists. "TI-LFA enabled" != "node protection guaranteed."

**12. TI-LFA SRLG Protection**
Prunes same-SRLG links before running CSPF, falling back to link protection if none found. SRLG checking
is local-only (no visibility into remote routers' tags), and lowest-backup-metric has zero effect on
TI-LFA, since TI-LFA's whole point is the genuine post-convergence path.

**13. TI-LFA Protection Priority — ISIS vs OSPF**
Both fall back the same way (both types → higher priority → link-only), but ISIS prefers lower tiebreaker
index while OSPF prefers higher; ISIS interface tiebreakers replace AFI-level ones while OSPF merges them;
lowest-backup-metric has zero effect on ISIS TI-LFA but does affect OSPF's, which also has extra
lc-disjoint and a hidden always-on "Post Convergence Path" tiebreaker.

---

## PART 4 — SR-TE Policies

**1. SR-TE Policy Fundamentals**
Uniquely identified by color + endpoint, with candidate paths ranked by preference (highest tried first,
opposite of RSVP-TE). BGP routes get a color community, and Automated Steering matches color+nexthop to a
local policy, recursing onto its BSID instead of the default IGP path.

**2. BGP Automated Steering with Multiple Colors**
A route can carry multiple colors; the router tries the highest first, falling back automatically to the
next valid policy, and finally plain IGP. Gives primary/backup path selection at the service layer,
entirely independent of BGP reconvergence.

**3. Dynamic Metric Types and ECMP**
Only IGP metric type gets automatic ECMP (it resolves to the Algo-0 Prefix-SID). TE metric, hopcount, and
latency always collapse to a single explicit path, since there's no equivalent "optimal path with ECMP"
Prefix-SID concept for those metrics.

**4. Affinity (Link Coloring) Constraints**
Tags links with named attributes flooded as an admin group (up to 256 bit positions via 8 extended admin
group entries). Policies constrain with include-all (AND), include-any (OR), exclude-any (inverted OR).
An over-restrictive requirement can produce zero valid paths.

**5. Metric Margin**
Lets a TE/latency-metric policy ECMP across paths within a configured tolerance of the best path. Hidden
catch: the underlying IGP cost of the candidate paths must already be equal, or margin ECMP silently has
no effect even for paths well within the margin.

**6. Candidate Path Fallback and BSID Stability**
A lower-preference candidate is only used when the higher-preference one completely fails — never on a
tie. The BSID stays exactly the same across a fallback event, which is what lets external systems
reference the policy without reconfiguration during a real failover.

**7. Explicit Paths — Addresses vs Labels**
Address-based lists validate every hop; label-based lists only validate the first label (enabling
inter-domain, PCE-computed paths). An invalid extra label past the first index still shows the policy as
fully "up" while traffic is silently broken.

**8. Explicit Paths Are "Loose"**
The router resolves each listed entry to a SID in order with no CSPF and no label-stack optimization — an
over-specified list is never simplified, and if the destination isn't the final entry, the policy still
shows "up" but isn't actually a path to that destination.

**9. Disjoint Planes Using Anycast SIDs**
Uses an Anycast SID (N-flag cleared) as the first hop of a two-segment explicit path to steer traffic onto
one of several physically-separate network planes, with IGP forwarding handling the rest. Doesn't scale
well — incompatible with ODN, so affinity + ODN is the more scalable production alternative.

**10. Binding SID (BSID) Allocation**
A local label representing an entire policy as one segment. Default dynamic; for predictability, assign
from the SRLB (which only exists once segment-routing is enabled globally). Conflicting explicit BSIDs
fail hard by default (fallback-dynamic can soften this); incompatible with ODN templates.

**11. SR/RSVP-TE Stitching via BSID**
Allocating a BSID directly on an RSVP-TE tunnel interface turns the whole tunnel into one opaque label an
SR-TE explicit path can push — letting SR-TE span a legacy RSVP-TE-only core segment with zero autoroute
or static-route configuration on the RSVP-TE side.

**12. Autoroute Include**
Makes a policy appear as an IGP-graph link for included prefixes, but by default only affects IP-to-Label
forwarding, not MPLS-to-MPLS, due to Algo-0 loop risk. Forcing this with force-sr-include can cause the
policy itself to flap from a GRE-style recursion loop — autoroute-include is an anti-pattern; color BGP
routes and use Automated Steering/ODN instead.

---

## PART 5 — PCE, BGP EPE, Disjoint Paths, Telemetry

**1. PCE Fundamentals**
A centralized, stateful path-computation server; routers delegate as PCCs for problems needing full
topology visibility (inter-domain, disjoint paths). Every node needs a TE Router ID and global mpls
traffic-eng enabled, or the PCE simply can't place it in its topology graph.

**2. PCEP Workflow (Report/Update/Initiate)**
PCC sends a Report with an empty SID list and the delegate flag set to request computation; PCE replies
with an Update. PCEP Initiate lets the PCE push an entirely new policy with zero local config on the PCC.
Local config always wins a tie against a PCE-initiated policy.

**3. SR-TED and Multi-Instance Distribution**
The PCE's merged topology graph, fed by distribute link-state tagged with a unique instance-id per IGP
instance (not per area/level). Inter-domain boundary nodes need a matching TE RID across both IGPs but a
different regular router ID to avoid accidental adjacencies.

**4. ODN with PCE Fallback and BGP Recursion Fix**
ODN with pcep creates two candidate paths — local (pref 200) tried first, PCE (pref 100) as fallback. If a
PE has no RIB-level reachability to a remote endpoint by design, BGP flags a "RIB failure" even with the
policy up — fixed with a null0 static route or the newer bgp bestpath/nexthop-validation sr-policy knobs.

**5. Computation Design Models**
Centralized (controller pushes everything, no local computation), Distributed (every router computes
locally, no PCE), Hybrid (local by default, PCE only for problems that need it) — hybrid is generally
recommended to avoid making the PCE an unnecessary bottleneck.

**6. PCC Features (Authentication, report-all)**
PCEP sessions support TCP MD5 authentication. By default a PCC only reports PCE-involved policies;
report-all forces it to report every locally-computed LSP too, needed if the PCE is meant to be a
complete source of network-wide visibility.

**7. PCE-Initiated Policies**
The PCE defines a full policy under its own peer-specific config and pushes it via PCEP Initiate with zero
local router configuration — the foundation for centralized SDN-driven TE. Local config always wins if it
collides with a PCE-initiated policy.

**8. PCE Redundancy**
A PCC elects a primary PCE by precedence (lowest wins), reporting to all connected PCEs but delegating
only to the primary; on primary loss it re-delegates immediately. PCE-initiated (controller-pushed)
policies fail over far less gracefully, using much longer recovery timers.

**9. PCE Redundancy with State-Sync**
PCEs sync directly, forwarding — never re-forwarding — PCC reports (iBGP split-horizon style, needing a
full mesh). Solves cross-PCC disjoint-path computation via a master/slave relationship where the slave
sub-delegates to the master when it lacks delegated control over both PCCs' policies.

**10. BGP Egress Peer Engineering (EPE)**
Lets you dictate exactly which eBGP peer/link an egress PE uses via peering SIDs (Peer Node SID, Peer Adj
SID per link) — something normal BGP attributes can't do since those only affect your own bestpath
decision. The remote peer needs zero SR awareness.

**11. BGP-LS for Unified MPLS with EPE**
Carries IGP topology (3 NLRI types: Node/Link/Prefix) plus EPE peering SIDs over BGP, letting a PCE build
one unified multi-domain graph. EPE links are always metric-0, which can cause bizarre figure-8 paths —
fixed by optimizing on hopcount instead of IGP/TE metric for policies crossing EPE segments.

**12. SR-TE Disjoint Paths**
A shared group-id makes the PCE compute two policies together (link/node/srlg/srlg-node disjoint) instead
of each independently finding the same shortest path. Only two LSPs supported per group-id; silently falls
back to a weaker type unless strict mode is enabled on the PCE.

**13. Anycast SID for ABRs with Conditional Advertisement**
ABR pairs share an Anycast SID on a dedicated loopback; conditional, event-driven (async) advertisement
withdraws it from the access domain if the ABR loses its own upstream connectivity — fixing the "healthy
downstream, dead uplink" black-hole scenario basic Anycast redundancy misses.

**14. iBGP Underlay Design for Multi-Domain Loopback Reachability**
Uses iBGP (not direct IGP redistribution) on ABRs to carry loopbacks across domain boundaries, with route
tagging to stop an ABR from preferring a stale, redistributed IGP route (leaked from its peer) over its
own fresher, directly-learned BGP route.

**15. Performance Measurement (Interface Delay / TWAMP-light)**
Actively measures real link delay — two-way mode needs no clock sync, one-way needs PTP — feeding
latency-optimized SR-TE policies via a two-tier IGP advertisement system: periodic checks (percentage +
absolute threshold) and accelerated checks for large jumps needing immediate flooding.

