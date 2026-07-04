# CCDE Segment Routing — Interview-Ready Answers (All Concepts)
### Just the 2-3 line interview answers for every concept from Parts 1-5
### Compiled: 2026

---

## PART 1 — SR Core: SRGB, PHP, Explicit-Null, Anycast SID

**1. SRGB** — "The SRGB is the local label range each SR node reserves for global prefix-SIDs; a node's
actual label for a given prefix is SRGB-base + index, and the SRGB itself is flooded via the IGP (ISIS
Router Capability TLV 242 / OSPF RI LSA) as a capability so every router can compute every other router's
label without any label-mapping protocol. Best practice is to keep the SRGB identical everywhere so labels
are globally predictable."

**2. SR Label Space Architecture** — "IOS-XR statically partitions the label space into reserved (0-15),
static (16-14999), SRLB (15000-15999) for local SIDs like binding-SIDs, SRGB (16000-23999 default) for
global prefix-SIDs, and dynamic (24000+) for LDP/RSVP/BGP-LU. Relocating the SRGB automatically makes the
dynamic range discontiguous around it, so SRGB sizing must be planned like address planning."

**3. Prefix-SID Index vs Label / SRGB Mismatch** — "A Prefix-SID is flooded as an index, not a label; each
router computes its local label as own SRGB + index. Forwarding still obeys downstream-allocated MPLS
semantics — a router always pushes the label its next-hop expects — so with different SRGBs you get normal
label swapping performed via computation instead of signaling."

**4. PHP** — "PHP means the penultimate router pops the top label so the final node does a single lookup
instead of pop-then-lookup. It's signaled by the P-flag (No-PHP flag) in the Prefix-SID sub-TLV, defaulting
to 'PHP allowed.' The trade-off is that PHP strips MPLS EXP bits one hop early, so if the true egress node
needs EXP-based QoS decisions, you must explicitly disable PHP for that prefix."

**5. Explicit-Null** — "SR has no signaling protocol, so instead of an LDP-style explicit-null
label-mapping message, it encodes the request as two flags in the Prefix-SID sub-TLV: the P-flag (No-PHP)
tells upstream neighbors not to pop, and the E-flag tells them to swap to the reserved Explicit-Null label
(0/2) instead — preserving EXP bits all the way to the final hop."

**6. ISIS SR Advertisement** — "SR extends IS-IS's Router Capability TLV (242) with SR-Capability and
SR-Algorithm sub-TLVs for node-level state, and extends the prefix-reachability TLVs with a Prefix-SID
sub-TLV carrying flags and an index/label. Unsupported routers just ignore the sub-TLVs and keep routing
normally, which is what makes incremental SR rollout possible."

**7. Anycast SID** — "An Anycast SID is a Prefix-SID configured identically on multiple nodes with the
N-flag cleared, representing 'any member of this redundant group' rather than one specific device.
Forgetting to clear the N-flag doesn't break normal ECMP forwarding, which is exactly why it's a
dangerous, easy-to-miss misconfiguration — it only manifests when TI-LFA tries to compute a backup path."

**8. N-flag and TI-LFA Interaction** — "TI-LFA uses the N-flag to decide whether it's safe to chain a
node-specific Adjacency-SID onto a Prefix-SID when building a repair path. If an anycast SID incorrectly
keeps the N-flag set, ECMP can send the packet to a different anycast member than TI-LFA assumed, causing
drops or mis-forwarding during the exact failure event TI-LFA was meant to protect against."

---

## PART 2 — SR-IGP/BGP Fundamentals

**1. Why SR Replaces LDP/RSVP-TE** — "SR eliminates LDP's separate signaling session and eliminates
RSVP-TE's soft-state refresh, full-mesh requirement, and non-ECMP-awareness — by carrying all label
information inside the IGP's existing flooding, so any node can compute an explicit label-stack path with
zero additional protocol state, and TI-LFA replaces RSVP-TE FRR without extra signaling."

**2. Prefix-SID Config: Index vs Absolute** — "Prefix-SIDs can be configured as an index or an absolute
label, but absolute values are silently converted to an index for advertisement — and if the SRGB is
later moved so the absolute value falls outside the new range, the SID quietly stops being advertised.
Best practice is always index-based configuration."

**3. ISIS Prefix-SID Sub-TLV Flags** — "The Prefix-SID sub-TLV flags octet has R (re-advertised), N (true
node SID), P (No-PHP), E (Explicit-Null, requires P), V and L (should never appear on a genuine
Prefix-SID). P and E are separate bits because No-PHP and Explicit-Null answer different questions — you
frequently need No-PHP without Explicit-Null."

**4. ISIS Router Capabilities TLV 242** — "The Router Capabilities TLV (242) carries node-scoped SR state:
SRGB via SR-Capability, supported SPF algorithms via SR-Algorithm, and Max SID Depth (MSD) — the deepest
label stack that node's hardware can forward. Any PCE or SR-TE headend must respect the minimum MSD along
a path."

**5. SR Algorithms — Algo 0 vs Algo 1** — "Algo 0 follows the IGP shortest path but allows local policy to
override it at any transit hop. Algo 1, strict SPF, guarantees the literal IGP path with zero possibility
of local diversion. A prefix can advertise both simultaneously as two separate Prefix-SIDs."

**6. Adjacency-SID Fundamentals** — "Every IGP adjacency gets two dynamically-allocated Adj-SIDs: a
non-FRR one, always advertised, and an FRR-eligible one, only advertised once TI-LFA is enabled on that
interface — because the label itself signals whether that forwarding entry is backed by a pre-computed
local repair path."

**7. Adjacency-SID Flags, Persistence, SRLB** — "Adj-SID flags include F, B (backup/FRR), V and L (always
set, since Adj-SIDs are always locally-significant explicit values), S (unsupported), and P (persistent).
Dynamic Adj-SIDs are only guaranteed stable for 30 minutes or until reboot, so a design needing a
guaranteed-stable value must statically allocate it from the SRLB."

**8. LAN Adjacency-SID** — "On a LAN segment, ISIS decomposes the shared pseudonode adjacency into one
LAN-Adj-SID per neighbor (DIS advertises none), while OSPF advertises a normal Adj-SID toward the DR but a
LAN-Adj-SID toward every BDR/DROTHER. In ISIS, LAN-Adj-SIDs are FRR-eligible but not actually
TI-LFA-protected."

**9. OSPF SR Control Plane** — "OSPF carries SR state in Opaque-Area LSAs: Type 1 (TE info, auto-generated
with SR), Type 4 (router-level SR capabilities), Type 7 (per-prefix Prefix-SID), and Type 8 (per-link
Adj-SID) — deliberately separate LSA types from classic MPLS-TE, so SR can run fully independent of
RSVP-TE."

**10. OSPF vs ISIS Structural Differences** — "sr-prefer is separate in OSPF vs. a keyword in ISIS; OSPF
allows TI-LFA at any level with inheritance while ISIS only allows interface-level; OSPF prefers higher
tiebreaker index while ISIS prefers lower; and only ISIS currently supports IPv6 for both SR-MPLS and
SRv6."

**11. OSPF Prefix-SID Flags** — "OSPF splits Prefix-SID flags across two levels: the Extended Prefix TLV
has A (attached-to-ABR) and N (node SID), while the nested SID sub-TLV has NP, M (mapping-server, no ISIS
equivalent), E, V, and L. The M flag specifically enables brownfield SR rollout without requiring every
router to be individually SR-capable."

**12. OSPF Adjacency-SID Flags** — "OSPF Adj-SID flags are B (backup/FRR), V, L (always set by
construction), and S (unsupported). In practice you only ever see 0xE0 (protected) or 0x60 (unprotected),
simplifying OSPF Adj-SID auditing versus ISIS."

**13. SR/RSVP-TE Interaction (Algo 0 Hijacking)** — "An RSVP-TE tunnel with autoroute-announce will
silently capture Algo-0 SR traffic at any transit node, because Algo 0 explicitly permits local-policy
overrides. The fix doesn't require touching the transit node — you advertise a second, Algo-1 Prefix-SID
at the destination, giving traffic a label that guarantees the literal IGP path."

**14. Inter-Area SR Propagation (ISIS)** — "Inter-area/inter-level SR propagation works automatically once
you configure normal route propagation. The ABR/L1L2 router sets No-PHP and R, but deliberately does not
set Explicit-Null, since there's no QoS-preservation need — exactly why No-PHP and Explicit-Null are
separate flags."

**15. OSPF Inter-Area SR** — "OSPF inter-area SR propagation marks Prefix-SIDs with route-type 3, sets
No-PHP without Explicit-Null, and inherits OSPF's existing Type-3 loop-prevention rule automatically —
meaning SR requires zero additional loop-prevention logic of its own."

**16. Inter-IGP SR Redistribution** — "SR-aware redistribution is RIB-driven — the ASBR pulls the
already-resolved label out of the RIB and converts it back to an index, but only if both IGPs share the
same SRGB; if they don't, the prefix still redistributes at the IP layer but silently loses its label."

**17. SR for BGP** — "SR for BGP is BGP-LU plus an optional-transitive Prefix-SID attribute carrying a
label-index — every hop with a globally-defined SRGB honors this as a hint to allocate SRGB+index. The
dangerous failure mode is running IGP-SR and classic BGP-LU together on the same box — they compete for
the same LFIB slot and BGP's allocation attempt fails outright."

**18. BGP-SR Operational Pitfalls** — "PEs need allocate-label all even for routes they don't originate;
direct eBGP peering between two PEs on a labeled address-family always triggers local Pop-label
allocation, fixed via Route Reflector peering or ebgp-multihop mpls; and stale duplicate dynamic-label
bugs have been observed requiring a full BGP process removal-and-readd to clear."

---

## PART 3 — Fast Reroute: LFA, RLFA, TI-LFA

**1. LFA Fundamentals** — "LFA pre-computes a backup next-hop using the inequality Dist(N,D) <
Dist(N,PLR)+Dist(PLR,D) to guarantee the backup won't loop traffic back. It gives sub-50ms protection with
zero extra protocol signaling, but coverage is entirely topology-dependent."

**2. LFA Verification Flags** — "The FRR flags — P, NP, TM, LC, D, SRLG — tell you exactly what kind of
protection a backup path provides. Seeing NP=Yes with plain LFA doesn't mean you configured node
protection — you need explicit tiebreakers to guarantee it stays true."

**3. Excluding Interfaces from LFA** — "Interface exclusion for LFA is an absolute rule, not a preference —
unlike SRLG-disjoint tiebreakers, excluding an interface means it will never be used, even if that means
losing protection completely for that prefix."

**4. LFA Tiebreakers** — "ISIS LFA tiebreakers decide which backup path wins when multiple valid LFAs
exist, evaluated in index order. The default order prefers cheapest cost first, which can silently pick a
backup that isn't node-protecting — so any network caring about surviving a full router failure must
explicitly reorder tiebreakers."

**5. Remote LFA** — "Remote LFA tunnels traffic via LDP to a PQ node — a router in both your P-space and
the destination's Q-space. If the PQ node doesn't accept targeted LDP sessions, you still get a working
backup path but with only one label instead of two — silently breaking end-to-end MPLS service LSPs."

**6. Remote LFA Limitations** — "RLFA can only ever provide link protection, never node or SRLG
protection, no matter how you configure tiebreakers. It also doesn't use the true post-convergence path
and depends on targeted LDP sessions with every PQ node — exactly the gaps TI-LFA was built to close."

**7. Extended P-Space** — "Extended P-space lets an RLFA-protecting router borrow its own neighbor's
P-space, finding a valid PQ node even when the router's own P-space doesn't overlap the destination's
Q-space. But feasibility depends on a metric threshold that can silently be crossed elsewhere."

**8. TI-LFA Fundamentals** — "TI-LFA removes the protected link, node, or SRLG from its topology view, runs
CSPF to find the genuine post-convergence path, and builds the shortest SID list using Adjacency-SIDs.
This gives 100% topology coverage, unlike LFA or RLFA, and can even protect plain LDP/IP traffic."

**9. TI-LFA Prefix-SID Dependency** — "TI-LFA needs a Prefix-SID for every intermediate node it might route
through. ISIS uses the TE Router ID TLV (134), falling back to TLV 132; OSPF checks the RID, then the
highest reachable host prefix with the N-flag set — exactly why an Anycast SID with the N-flag left set can
silently break TI-LFA."

**10. TI-LFA Adj-SID Selection** — "When a Q-node has both a protected and unprotected Adjacency-SID for
the same link, TI-LFA deliberately uses the unprotected one, specifically to avoid microloops during a
second, overlapping failure."

**11. TI-LFA Node Protection** — "TI-LFA by default only protects against link failures. To get real node
protection, you add the node-protecting tiebreaker. Enabling TI-LFA alone does not guarantee you survive a
full router crash."

**12. TI-LFA SRLG Protection** — "SRLG protection tells TI-LFA to avoid backup paths sharing physical fate
with the protected link. Key limitations: SRLG checking is local-only, and the lowest-backup-metric
tiebreaker has zero effect on TI-LFA, since TI-LFA's entire purpose is the genuine post-convergence path."

**13. TI-LFA Priority — ISIS vs OSPF** — "TI-LFA priority ordering works conceptually the same in both, but
mechanics diverge: ISIS prefers lower index, OSPF prefers higher; ISIS interface tiebreakers replace
AFI-level ones, OSPF merges them; and lowest-backup-metric has zero effect on ISIS's TI-LFA but does affect
OSPF's, alongside OSPF's hidden always-on Post Convergence Path tiebreaker."

---

## PART 4 — SR-TE Policies

**1. SR-TE Policy Fundamentals** — "An SR-TE policy is uniquely identified by a color and an endpoint, with
candidate paths ranked by preference — highest tried first. BGP routes are tagged with a color community,
and Automated Steering matches color+nexthop to a local policy, recursing onto the policy's BSID."

**2. Multi-Color Automated Steering** — "A BGP route can carry multiple color communities, set individually
with additive, and the router tries the highest color first, falling back automatically to the next valid
policy, and finally to plain IGP — giving primary/backup selection entirely independent of BGP
reconvergence."

**3. Dynamic Metric Types and ECMP** — "Only the IGP metric type gets automatic ECMP in dynamic SR-TE
policies, because it resolves to the Algo-0 Prefix-SID. TE metric, hopcount, and latency policies always
collapse to a single path, since there's no equivalent Prefix-SID representing 'the optimal path with ECMP
built in.'"

**4. Affinity Constraints** — "Affinity tags links with named attributes flooded as an admin group — up to
256 bit positions via 8 extended admin group entries. Policies constrain with include-all, include-any, or
exclude-any. An over-restrictive requirement can produce zero valid paths."

**5. Metric Margin** — "Margin lets a TE or latency-metric policy ECMP across paths within a configured
tolerance of the best path. But the underlying IGP cost of those paths must already be equal for margin
ECMP to take effect — a genuinely counter-intuitive design trap."

**6. Candidate Path Fallback / BSID Stability** — "A lower-preference candidate path is only used when the
higher-preference one completely fails — never due to a tie. The policy's BSID stays exactly the same
across a fallback, which is what lets other systems reference it without reconfiguration."

**7. Explicit Paths — Addresses vs Labels** — "Address-based segment-lists get every hop validated;
label-based lists only validate the first label, enabling inter-domain PCE-computed paths. An invalid
label past the first index shows the policy as fully up while traffic is silently broken."

**8. Explicit Paths Are Loose** — "SR-TE explicit segment-lists are loose in that intermediate hops don't
need to be adjacent, but the router just resolves each entry to a SID in order, without CSPF or label-
stack optimization. If the destination isn't the final entry, the policy still shows up but isn't a path to
that destination."

**9. Disjoint Planes Using Anycast SIDs** — "Disjoint-plane steering uses an Anycast SID (N-flag cleared)
as the first hop of a two-segment explicit path. The real lesson is scalability — this explicit approach
doesn't work with ODN, so production designs would use affinity-based link coloring combined with ODN
instead."

**10. BSID Allocation** — "A BSID is a local label representing an entire SR-TE policy as one segment. By
default it's dynamically allocated; for predictability you assign it from the SRLB. Conflicting explicit
BSIDs fail hard by default, and explicit BSIDs are incompatible with ODN templates."

**11. SR/RSVP-TE Stitching via BSID** — "You can allocate a BSID directly on an RSVP-TE tunnel interface,
turning the whole tunnel into a single opaque label an SR-TE explicit path can push — letting an SR-TE
policy span a legacy RSVP-TE-only core segment with zero autoroute or static-route configuration needed."

**12. Autoroute Include** — "Autoroute include makes a policy appear as an IGP-graph link, but by default
only affects IP-to-Label forwarding, not MPLS-to-MPLS, due to Algo-0 loop risk. Forcing this with
force-sr-include can cause the policy itself to flap from a GRE-style recursion loop — the correct approach
is coloring BGP routes and using Automated Steering or ODN."

---

## PART 5 — PCE, BGP EPE, Disjoint Paths, Telemetry

**1. PCE Fundamentals** — "A PCE is a centralized, stateful path-computation server; routers delegate as
PCCs when they need full-topology visibility for inter-domain or disjoint-path computation. A commonly-
missed prerequisite is that every node needs a TE Router ID and global mpls traffic-eng enabled."

**2. PCEP Workflow** — "PCEP works through Reports (empty SID list + delegate flag = please compute) and
Updates (PCE's computed path). PCEP Initiate is the reverse — the PCE pushes an entirely new policy with no
local config. Local config always wins a tie against a PCE-initiated policy."

**3. SR-TED / Multi-Instance Distribution** — "The SR-TED is the PCE's merged topology graph, fed by
distribute link-state tagged with a unique instance-id per IGP instance, not per area or level. Inter-
domain boundary nodes need a matching TE RID across IGPs but a different regular router ID."

**4. ODN + PCE Fallback / BGP Recursion Fix** — "ODN with pcep creates two candidate paths — local at
preference 200, PCE fallback at preference 100. If a PE has no RIB-level reachability by design, BGP still
flags a RIB failure even with the SR-TE policy up — fixed with a null0 static route or the newer bgp
bestpath/nexthop-validation sr-policy knobs."

**5. Computation Design Models** — "Centralized has a controller push every policy with no local
computation; distributed has every router compute locally with no PCE; hybrid computes locally by default,
delegating to a PCE only when needed. Hybrid is generally recommended."

**6. PCC Features** — "PCEP sessions can use TCP MD5 authentication. By default a PCC only reports policies
the PCE was involved in — report-all forces it to report every LSP, visible on the PCE as an empty
'Computed path' since the PCE never calculated it."

**7. PCE-Initiated Policies** — "A PCE can define a full policy under its own peer-specific config and push
it via PCEP Initiate with zero local router configuration — the foundation for centralized SDN-driven TE.
Local config always wins if it collides with a PCE-initiated one."

**8. PCE Redundancy** — "A PCC elects a primary PCE by precedence (lowest wins), delegating there while
reporting to all connected PCEs. On primary loss, the PCC immediately re-delegates. PCE-initiated policies
fail over much less gracefully, using much longer recovery timers."

**9. PCE Redundancy with Sync** — "PCEs can sync directly, forwarding — but never re-forwarding — PCC
reports, mimicking iBGP split-horizon. This solves cross-PCC disjoint-path computation via a master/slave
relationship where the slave sub-delegates to the master when it lacks delegated control."

**10. BGP EPE** — "BGP EPE lets you dictate exactly which eBGP peer, even which link, an egress PE uses via
peering SIDs — something normal BGP attributes can't do since those only affect your own bestpath
decision, while the remote peer needs zero SR awareness."

**11. BGP-LS for Unified MPLS** — "BGP-LS re-encodes IGP topology into three NLRI types carried over BGP,
letting a PCE build one unified graph spanning multiple domains. EPE links always show as metric-0, which
can cause bizarre figure-8 paths — fixed by optimizing on hopcount instead of IGP/TE metric."

**12. SR-TE Disjoint Paths** — "Disjoint-path uses a shared group-id to make the PCE compute two policies
together. Only two LSPs are supported per group-id, and the PCE silently falls back to a weaker disjoint
type unless you explicitly enable strict mode, which should be the default for any real requirement."

**13. Anycast SID + Conditional Advertisement (ABRs)** — "ABR pairs share an Anycast SID on a dedicated
loopback. Conditional advertisement solves the 'healthy downstream, dead uplink' black-hole by tying the
Anycast loopback's advertisement to the presence of specific upstream routes in the RIB via an
event-driven async check."

**14. iBGP Underlay Design** — "Instead of redistributing loopbacks directly between IGPs, this pattern
uses iBGP on the ABRs. The key gotcha is an ABR preferring a stale, IGP-redistributed route leaked from its
peer over its own fresher BGP route — solved by tagging routes on redistribution and denying re-acceptance
of that tag back into the RIB."

**15. Performance Measurement (TWAMP-light)** — "Performance-Measurement uses TWAMP-light to measure real
link delay — two-way by default needing no clock sync, or one-way requiring PTP. Measured values feed
latency-optimized SR-TE via a two-tier advertisement system: periodic checks and accelerated checks for
large jumps needing immediate flooding."

