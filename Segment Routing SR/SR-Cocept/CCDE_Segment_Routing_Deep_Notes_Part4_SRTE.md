# CCDE-Level Segment Routing Deep Notes — Part 4 (SR-TE Policies)
### Source labs: CCIE-SP v5.1 — SR-TE with AS Primary/Secondary Paths, SR-TE Dynamic Policies,
### SR-TE Dynamic Policy with Margin, SR-TE Explicit Paths, SR-TE Disjoint Planes using Anycast SIDs,
### SR-TE BSIDs, SR-TE RSVP-TE Stitching, SR-TE Autoroute Include
### Compiled: 2026

> Continuation of the earlier CCDE SR deep-notes files (SRGB/PHP/ExpNull/Anycast; IGP/BGP SR fundamentals;
> LFA/RLFA/TI-LFA). This file covers SR-TE (Segment Routing Traffic Engineering) policies: how they're
> built, steered into, constrained, and stitched across legacy cores.

---

## Table of Contents
1. SR-TE Policy Fundamentals — Color, Endpoint, Candidate Paths
2. BGP Automated Steering (AS) with Multiple Colors — Primary/Backup via Fallback
3. SR-TE Dynamic Metric Types and Why Only IGP Metric Gets ECMP
4. SR-TE Constraints — Link Affinity (Color) and the Extended Admin Group TLV
5. SR-TE Metric Margin — Controlled ECMP Tolerance for TE/Latency Metrics
6. SR-TE Candidate Path Selection — Fallback-on-Failure (Not Tie), and BSID Stability
7. SR-TE Explicit Paths — Addresses vs. Labels, and the "Only First Label Validated" Trap
8. SR-TE Explicit Paths Are "Loose" — No CSPF, No Label Reduction, Destination Must Be Explicit
9. SR-TE Disjoint Planes Using Anycast SIDs
10. SR-TE Binding SID (BSID) — Allocation, Conflicts, and SRLB Enforcement
11. SR-TE / RSVP-TE Stitching via BSID
12. SR-TE Autoroute Include — Per-Prefix IGP Injection, and the MPLS-to-MPLS FIB Gotcha

---

## 1. SR-TE Policy Fundamentals — Color, Endpoint, Candidate Paths

**1. Definition**
An SR-TE policy is a named, explicit traffic path from a headend router to a destination, uniquely
identified by a color (a number representing an intent, like "low-latency" or "backup") and an endpoint
(the destination router's address). Inside a policy, one or more candidate paths describe how to actually
build that path.

**2. Why it exists**
A network often needs more than one way to reach the same destination. Prefix-SIDs alone only give you the
single default IGP path. SR-TE policies let you define multiple, purpose-built paths to the same
destination and let BGP tell each service route which one it actually wants, using the color as the
selector.

**3. How it works**
- A policy is defined by `color X end-point ipv4 Y` — this pair is the unique key.
- Inside, one or more candidate-paths blocks are defined, each with a preference number.
- Unlike RSVP-TE (lower path-option number tried first), SR-TE works the opposite way: the highest
  preference number is tried first.
- Each candidate path can be dynamic (router computes via CSPF) or explicit (a specific segment-list).
  ```
  segment-routing
   traffic-eng
    policy R7_COLOR_10
     color 10 end-point ipv4 7.7.7.1
     candidate-paths preference 100 dynamic
  ```
- BGP service routes get a color extended community attached; the router automatically matches
  color+nexthop against a local SR-TE policy — Automated Steering (AS).

**4. Real-world use case**
Any SP offering differentiated SLA tiers over the same physical core (premium low-latency vs. standard
best-effort VPN service) uses this color+endpoint model.

**5. Failure scenario**
If a BGP route arrives with a color that doesn't match any local policy, AS has nothing to steer into — the
route falls back to normal IGP/BGP resolution, silently losing the SLA guarantee unless specifically
monitored.

**6. Design insight**
The color+endpoint model scales because you don't need per-customer configuration everywhere — just
consistent color semantics agreed upon network-wide, and BGP naturally carries the steering intent.

**7. Interview-ready answer**
"An SR-TE policy is uniquely identified by a color and an endpoint, with candidate paths ranked by
preference — highest tried first, opposite of RSVP-TE's numbering. BGP routes are tagged with a color
community, and Automated Steering matches color+nexthop to a local policy, recursing onto the policy's
BSID instead of the default IGP path."

---

## 2. BGP Automated Steering (AS) with Multiple Colors — Primary/Backup via Fallback

**1. Definition**
A BGP route can carry multiple color extended communities. The router tries the highest-numbered color
first; if no valid policy exists for that color, it tries the next-highest, falling all the way back to
plain IGP routing if none are valid.

**2. Why it exists**
Real networks need primary/backup path selection at the service level. Rather than a separate mechanism,
SR-TE reuses BGP coloring: attach both a primary and backup color to the route, and let color-preference
logic do the failover automatically.

**3. How it works**
- R1 has two policies to R7: color 20 (explicit path via R4), color 10 (dynamic TE-metric path).
- R7 attaches both colors:
  ```
  extcommunity-set opaque COLOR10
    10
  end-set
  !
  extcommunity-set opaque COLOR20
    20
  end-set
  !
  route-policy CE107_IN
    set extcommunity color COLOR10 additive
    set extcommunity color COLOR20 additive
  end-policy
  ```
  The `additive` keyword is required on each `set` line individually — you cannot set multiple colors in a
  single line.
- R1 uses color 20 (higher) by default. If R4's Prefix-SID is removed, breaking the color-20 explicit
  path, R1 automatically falls back to color 10 — no BGP reconvergence needed.

**4. Real-world use case**
Standard mechanism for primary/backup SR-TE paths for premium services — a latency-engineered primary
(higher color) with automatic fallback to a TE-metric-optimized backup (lower color).

**5. Failure scenario**
A critical service route with only a single color and no fallback falls all the way back to plain IGP
forwarding if that color's policy becomes invalid — silently losing the SLA guarantee.

**6. Design insight**
This shifts "failover" to the service-selection layer, not the transport layer — an independent, differently
-computed backup path, switched to automatically. Architects should always plan at least a two-color
(primary + fallback) scheme for anything beyond best-effort traffic.

**7. Interview-ready answer**
"A BGP route can carry multiple color communities, set individually with additive, and the router tries
the highest color first, falling back automatically to the next valid policy, and finally to plain IGP.
This gives primary/backup SR-TE path selection entirely independent of BGP reconvergence."

---

## 3. SR-TE Dynamic Metric Types and Why Only IGP Metric Gets ECMP

**1. Definition**
A dynamic SR-TE candidate path can optimize for IGP, TE (default), hopcount, or latency — but only IGP
metric produces ECMP; the other three always collapse to a single path.

**2. Why it exists**
Different services care about different things — hop count, engineered TE cost, or measured latency. But
the reason only IGP gets ECMP is a deep protocol consequence of how Prefix-SIDs work.

**3. How it works**
- IGP metric policy resolves to the destination's normal Algo-0 Prefix-SID: "follow IGP shortest path" —
  which inherently means ECMP if multiple equal-cost paths exist.
  ```
  policy R7_IGP
   color 10 end-point ipv4 7.7.7.1
   candidate-paths preference 100 dynamic metric type igp
  ```
  Local label (24016) shows two outgoing interfaces — genuine ECMP via R3 and R4.
- TE metric policy (the default type) never ECMPs, even with equal TE cost — because 16007 (a Prefix-SID)
  inherently means "follow the IGP path," not "follow the TE path." There's no equivalent "TE-metric
  Prefix-SID" concept, so the router must build a single explicit SID list.
- Hopcount policy collapses to a single path for the identical reason.
- Latency follows the same underlying logic (explored in a separate lab).
- This label-reduction/ECMP-optimization logic is called the "SR-native" algorithm — maximizing ECMP where
  the metric type allows, minimizing SID list length.

**4. Real-world use case**
A "TE-optimized, lowest cost" service tier will not automatically load-balance the way a plain IGP-based
service would — real capacity-planning implications.

**5. Failure scenario**
An operator configures a premium TE-metric service expecting natural load-balancing and is surprised to
find all traffic concentrating on one path, causing unexpected congestion while a parallel path sits idle.

**6. Design insight**
Choosing a richer metric type trades away automatic ECMP for precise single-path optimization. A design
needing both must accept the IGP metric type, or manually engineer multiple policies with weighted (UCMP)
load-sharing.

**7. Interview-ready answer**
"Only the IGP metric type gets automatic ECMP in dynamic SR-TE policies, because it resolves down to the
destination's Algo-0 Prefix-SID. TE metric, hopcount, and latency policies always collapse to a single
path, since there's no equivalent Prefix-SID representing 'the TE-optimal path with ECMP built in.'"

---

## 4. SR-TE Constraints — Link Affinity (Color) and the Extended Admin Group TLV

**1. Definition**
Affinity lets you tag interfaces with named attributes (BLUE, YELLOW, GREEN), and write SR-TE policy
constraints requiring a path to include, avoid, or logically combine links with those tags.

**2. Why it exists**
Some constraints aren't about cost at all — "only use certified low-jitter links," or "avoid links
crossing a regulatory boundary." Affinity gives a flexible tagging system decoupled from IGP metric.

**3. How it works**
```
segment-routing
 traffic-eng
  affinity-map
   name YELLOW bit-position 1
   name BLUE bit-position 0
   name GREEN bit-position 2
!
segment-routing
 traffic-eng
  interface GigabitEthernet0/0/0/3
   affinity
    name YELLOW
    name BLUE
    name GREEN
```
- Flooded into ISIS as an admin group value (0x7 with all three bits set).
- Extended range works via 8 Extended Admin Group entries of 32 bits each — 256 total bit positions. Bit
  position 32 triggers a second extended-admin-group TLV entry (0x1), while the legacy admin-group value
  stays unchanged.
- Three logical operators: include-all (AND — every link must have all names), include-any (OR — at least
  one), exclude-any (inverted OR — none may have any).
- Color 40 (YELLOW AND GREEN) resolves to a two-label path (16009, 16007).
- Color 50 (BLUE only) fails entirely — no path exists where every link is BLUE.
- Color 60 (YELLOW OR BLUE) succeeds.

**4. Real-world use case**
Avoiding specific submarine cable segments, regulatory/data-sovereignty jurisdictions, or requiring
SLA-certified fiber routes.

**5. Failure scenario**
An over-constrained affinity requirement (color 50) produces a policy with no valid path at all — if it's
the only candidate path, the service loses SR-TE connectivity entirely.

**6. Design insight**
Affinity constraints should almost always pair with a less-constrained fallback candidate path, unless "no
path" is a deliberately acceptable failure mode (e.g., strict regulatory requirements).

**7. Interview-ready answer**
"Affinity tags links with named attributes flooded as an admin group — up to 256 bit positions via 8
extended admin group entries. Policies constrain with include-all, include-any, or exclude-any. Because
it's a hard constraint, an over-restrictive requirement can produce zero valid paths, so affinity-
constrained policies should have a fallback candidate path."

---

## 5. SR-TE Metric Margin — Controlled ECMP Tolerance for TE/Latency Metrics

**1. Definition**
Margin lets a TE-metric or latency-metric dynamic policy accept multiple paths as equally valid for ECMP,
as long as their metric values fall within a specified tolerance of the best path.

**2. Why it exists**
TE and latency metrics normally never ECMP (Concept #3). But two paths close enough in value (1ms vs
1.1ms latency) shouldn't waste capacity by using only the "best" one. Margin reclaims ECMP within a
tolerance.

**3. How it works**
- `metric margin absolute <value>` or `metric margin relative <percentage>` — only one at a time.
- Example: R1-R4 TE metric 15, IGP metric 10. Without margin, only one path used. With `margin absolute
  20`, both R3 and R4 are used.
- **Critical hidden requirement**: the underlying IGP cost of all candidate paths must already be equal —
  the TE/latency metric can vary within margin, but IGP cost cannot. Proven directly: setting R1's IGP
  metric to R4 to 11 drops the policy back to using only R3, even though the TE metric was still within
  margin.
- Latency is measured in nanoseconds internally — a "1ms tolerance" margin must be configured as 1000.

**4. Real-world use case**
Latency-optimized services (financial trading, real-time media) use margin to avoid concentrating all
traffic on a marginally-better path while a near-identical alternate sits unused.

**5. Failure scenario**
An operator configures a generous margin expecting ECMP, but the IGP topology doesn't have equal-cost
paths to that destination — margin silently has zero effect, with no direct error.

**6. Design insight**
Margin-based ECMP requires co-designing the IGP topology and the TE/latency engineering together — "just
add a margin" is an oversimplification.

**7. Interview-ready answer**
"Margin lets a TE or latency-metric policy ECMP across paths within a configured tolerance of the best
path. But the underlying IGP cost of those paths must already be equal for margin ECMP to take effect — if
IGP costs differ, even paths well within the TE-metric margin won't be used together."

---

## 6. SR-TE Candidate Path Selection — Fallback-on-Failure (Not Tie), and BSID Stability

**1. Definition**
A lower-preference candidate path is only used when the higher-preference one completely fails to
resolve — never due to a tie. The policy's BSID stays the same single value across all its candidate
paths.

**2. Why it exists**
Predictable behavior: preference 100 as "the path I want," preference 90 as "last resort" only. BSID
stability matters because external systems reference a policy by BSID — if it changed on every fallback,
downstream references would break during normal, non-outage events.

**3. How it works**
- Color 50 (BLUE-only, fails) with a fallback:
  ```
  policy R7_USE_BLUE
   color 50 end-point ipv4 7.7.7.1
   candidate-paths
    preference 100 dynamic constraints affinity include-all name BLUE
    preference 90 dynamic
  ```
  Preference 100 fails, preference 90 is used.
- Fallback does NOT trigger on tie or non-failure: adding a lower-preference path to an already-working
  policy has no effect.
- BSID stability: a policy with preference 90 (unconstrained) and preference 100 (GREEN-constrained) comes
  up on preference 100 with BSID 24030. Removing GREEN affinity causes fallback to preference 90 — but the
  BSID remains 24030 unchanged.
- Any IGP topology change automatically triggers reevaluation of all affected candidate/explicit paths.

**4. Real-world use case**
BSID stability is essential for inter-domain stitched paths or PCE-referenced policies — those references
remain valid through a fallback event with no reconfiguration.

**5. Failure scenario**
An operator expects a lower-preference path to sometimes be used for load-distribution — it never will be,
as long as the higher-preference path remains valid.

**6. Design insight**
Candidate-path preference is a strict, ordered fallback chain, never a load-distribution mechanism. Real
load-distribution needs weighted/UCMP explicit paths instead.

**7. Interview-ready answer**
"A lower-preference candidate path is only used when the higher-preference one completely fails — never
due to a tie — and this reevaluation happens automatically on topology change. The policy's BSID stays
exactly the same across a fallback, which is what lets other systems reference it without reconfiguration."

---

## 7. SR-TE Explicit Paths — Addresses vs. Labels, and the "Only First Label Validated" Trap

**1. Definition**
An explicit segment-list can use IP addresses (fully resolved and validated) or raw MPLS labels (only the
first one is ever validated) — a distinction with serious operational consequences.

**2. Why it exists**
Address-based lists are for operator-friendly, fully-validated paths within a visible IGP domain.
Label-based lists exist for inter-domain paths where a headend (typically told by a PCE) uses labels from
a remote domain it has no visibility into — it structurally cannot validate them, so it only validates
enough to forward the first packet.

**3. How it works**
- Address-based: every address must resolve to a valid SID for the policy to be up.
  ```
  segment-list R6_R9_R7_ADDRESSES
   index 1 address ipv4 6.6.6.1
   index 2 address ipv4 9.9.9.1
   index 3 address ipv4 7.7.7.1
  ```
- Label-based: only the first label is resolved.
  ```
  segment-list R6_R9_R7_LABELS
   index 1 mpls label 16006
   index 2 mpls label 16009
   index 3 mpls label 16007
  ```
- **The trap**: adding an invalid extra label (e.g., index 4 = 16077) still shows the policy as fully up —
  only index 1 was checked. Traffic is silently broken while the control plane shows healthy.
- Mixed address/label lists are allowed; only the first index (regardless of type) is validated.
- In practice, hand-configuring long label-based paths on a headend is rare — a PCE computes the full path
  and signals it as a pre-validated label list.
- Algorithm selection: if an address has multiple Prefix-SIDs, algo 1 (strict SPF) is preferred by
  default, falling back to algo 0 — overridable with a `sid-algorithm` constraint.
- Adjacency-SID resolution via link IP addresses: the protected Adj-SID is automatically chosen over the
  unprotected one when both exist. Manually hard-coding a specific Adj-SID label is discouraged since it's
  dynamic by default.

**4. Real-world use case**
Label-based explicit segment-lists are standard for PCE-driven, inter-domain SR-TE.

**5. Failure scenario**
An operator (or script) appends an extra invalid label — the policy stays "up," monitoring shows green,
but real traffic is silently dropped/misdirected.

**6. Design insight**
Explicit label-based paths should be machine-generated, PCE-validated artifacts, never hand-typed in
production. Runbooks should mandate end-to-end traffic verification, not just policy-state checks.

**7. Interview-ready answer**
"Address-based segment-lists get every hop validated; label-based lists only validate the first label,
enabling inter-domain PCE-computed paths where the headend has no visibility into remote labels. But an
invalid label past the first index shows the policy as fully up while traffic is silently broken — which
is why hand-typed label paths in production are dangerous."

---

## 8. SR-TE Explicit Paths Are "Loose" — No CSPF, No Label Reduction, Destination Must Be Explicit

**1. Definition**
Unlike RSVP-TE's loose-hop CSPF trigger, an SR-TE segment-list is resolved by walking each entry one at a
time, resolving each to a SID, with no path computation and no optimization.

**2. Why it exists**
A segment-list is a literal, ordered list of segments — "looseness" only means intermediate hops don't
need to be adjacent (a router will take its normal IGP path between listed, non-adjacent hops), not that
the router will intelligently shorten an over-specified list.

**3. How it works**
- An over-specified list (every node listed, when two SIDs would suffice) is resolved exactly as given,
  with zero label-reduction optimization:
  ```
  segment-list VERY_LONG
   index 1 address ipv4 3.3.3.1
   index 2 address ipv4 9.9.9.1
   index 3 address ipv4 5.5.5.1
   index 4 address ipv4 7.7.7.1
  ```
- The destination must be explicitly the last entry. Removing it leaves the policy showing "up" but no
  longer actually a valid path to the endpoint — same "looks fine, isn't" trap as Concept #7.
- The router appears to simply resolve each entry one by one, rather than running genuine CSPF through the
  waypoints — which also explains the lack of SID-list-reduction optimization.

**4. Real-world use case**
Operators/automation building segment-lists must understand the router will not "clean up" a verbose
list — optimization must happen at the PCE/planning stage.

**5. Failure scenario**
A template off-by-one error or a stale list (after a topology change) omits the final destination entry —
the policy shows fully operational, but traffic never reaches the destination.

**6. Design insight**
The headend trusts the segment-list far more than it validates it. Automation must confirm the final entry
matches the endpoint, and ideally run automated end-to-end reachability tests after any change.

**7. Interview-ready answer**
"SR-TE explicit segment-lists are loose in that intermediate hops don't need to be adjacent, but the
router just resolves each entry to a SID in order, without CSPF or label-stack optimization. An
over-specified list is never simplified, and if the destination isn't the final entry, the policy still
shows up — it's just no longer a path to that destination."

---

## 9. SR-TE Disjoint Planes Using Anycast SIDs

**1. Definition**
A technique for steering traffic onto one of two physically-separate network "planes," using an Anycast
SID to represent "any router in this plane" as the first hop of an explicit path, with normal IGP
forwarding handling the rest once traffic lands in the correct plane.

**2. Why it exists**
Some designs deliberately build physically-independent core planes, engineered so traffic never crosses
between them under normal conditions. Operators need to steer certain traffic onto one plane without a
full mesh of per-router explicit paths — Anycast SIDs represent "reach any node in this group" as one
segment.

**3. How it works**
- Two planes — "odd" (R3, R5, R9) and "even" (R4, R6, R10) — with high-metric inter-plane core links, but
  equal-cost (10) PE-to-each-plane links.
  ```
  #R3, R5, R9 (odd)
  int lo101
   ip add 101.101.101.1/32
  router isis 1
   int lo101
    add ipv4 uni
     prefix-sid index 101 n-flag-clear

  #R4, R6, R10 (even)
  int lo100
   ip add 100.100.100.1/32
  router isis 1
   int lo100
    add ipv4 uni
     prefix-sid index 100 n-flag-clear
  ```
- Two-hop explicit path: plane's Anycast SID first, destination's Prefix-SID second.
  ```
  segment-list R7_VIA_ODD
   index 1 address ipv4 101.101.101.1
   index 2 address ipv4 7.7.7.1
  !
  policy R7_VIA_ODD
   color 10 end-point ipv4 7.7.7.1
   candidate-paths pref 100 explicit segment-list R7_VIA_ODD
  ```
- Traffic label-switches to whichever plane member is IGP-closest, then ordinary IGP forwarding carries it
  the rest of the way. R1's forwarding table shows next-hop R3 for ODD, R4 for EVEN.

**4. Real-world use case**
Large-scale providers wanting strict physical/logical separation between two infrastructure halves for
risk-diversity, with a simple mechanism to direct traffic classes onto one plane.

**5. Failure scenario**
If the N-flag were left set on either Anycast SID, this design would be exposed to the TI-LFA/Adj-SID-
chaining hazard from earlier notes — a real risk since disjoint-plane designs are exactly where TI-LFA
matters most.

**6. Design insight**
The lab's own conclusion: this method doesn't scale well — explicit, per-headend, per-destination policies
are incompatible with ODN. Affinity/link-coloring combined with ODN would achieve the same outcome without
a full mesh of manually pre-configured policies.

**7. Interview-ready answer**
"Disjoint-plane steering uses an Anycast SID (N-flag cleared) representing 'any router in this plane' as
the first hop of a two-segment explicit path, letting IGP forwarding carry traffic the rest of the way. The
real lesson is scalability — this explicit approach doesn't work with ODN, so production designs would use
affinity-based link coloring combined with ODN instead."

---

## 10. SR-TE Binding SID (BSID) — Allocation, Conflicts, and SRLB Enforcement

**1. Definition**
A BSID is a local MPLS label representing an entire SR-TE policy as a single segment — other routers and
local BGP recursion can use this one label to steer into the policy without knowing the full SID list.

**2. Why it exists**
Without a BSID, anyone steering into a policy would need to know its entire label stack, which breaks down
for dynamic policies whose paths can change. A BSID is a stable indirection layer — reference one small
label, and the headend translates it to the current SID list internally.

**3. How it works**
- Default: dynamically allocated from the dynamic range (24000+).
- Explicit, predictable allocation from the SRLB (15000-15999 default):
  ```
  policy POL1
   binding-sid mpls 15000
   color 10 end-point ipv4 7.7.7.1
   candidate-paths preference 100 dynamic
  ```
- **Hidden prerequisite**: the SRLB only exists once `segment-routing` is configured globally — simply
  configuring an SR-TE policy alone doesn't create it (the first policy configuration triggers global SR
  activation).
- Conflicting explicit BSIDs fail hard by default with a syslog error — each must be unique.
- `fallback-dynamic`: lets a policy degrade to a dynamic label instead of failing outright on conflict.
- `enforce-srlb`: mutually exclusive with fallback-dynamic; requires explicit BSIDs to fall within the
  SRLB range. Without it, an explicit BSID from the dynamic range is also accepted (but not guaranteed to
  survive reboot).
- `binding-sid dynamic disable`: requires every policy to have an explicit BSID; a policy without one
  fails to come up. This does NOT retroactively break a policy already using fallback-dynamic, since that
  policy did explicitly request a BSID.
- PCE-awareness: a PCE learns the headend's SRLB via the IGP, and currently-used static BSIDs via PCEP
  reports, avoiding collisions.
- Incompatibility with ODN: explicit BSIDs cannot be used on ODN templates, since each dynamically-spawned
  policy needs its own unique BSID.

**4. Real-world use case**
Predictable BSIDs are used when multiple Anycast nodes share a common BSID representing the same logical
policy, or when external systems need to reference a policy reliably across reboots/reoptimizations.

**5. Failure scenario**
An operator explicitly configures an already-in-use BSID without fallback-dynamic — the second policy
fails outright with a syslog error that might go unnoticed.

**6. Design insight**
The knobs represent a trade-off between predictability and resilience: dynamic disable + enforce-srlb
favors strict, predictable BSIDs (fail loudly on conflict); fallback-dynamic favors "the policy should
come up no matter what."

**7. Interview-ready answer**
"A BSID is a local label representing an entire SR-TE policy as one segment. By default it's dynamically
allocated; for predictability you assign it from the SRLB, which only exists once segment-routing is
enabled globally. Conflicting explicit BSIDs fail hard by default, though fallback-dynamic lets a policy
degrade instead — and explicit BSIDs are incompatible with ODN templates entirely."

---

## 11. SR-TE / RSVP-TE Stitching via BSID

**1. Definition**
Lets an SR-TE headend steer traffic through a legacy RSVP-TE-only network segment, by treating the RSVP-TE
tunnel's own BSID as just another label in an SR-TE explicit segment-list — no autoroute announce or
static routes needed.

**2. Why it exists**
Real RSVP-TE-to-SR migrations rarely happen all at once. This technique lets SR-TE policies span a
non-SR-capable core segment by treating the RSVP-TE tunnel as one opaque segment — a bridge during a long
migration.

**3. How it works**
- On the RSVP-TE headend (R3):
  ```
  int tunnel-te5
   binding-sid mpls label 4005
  ```
  Creates an LFIB entry: pop and forward out the TE tunnel.
- On the SR-TE headend (R1):
  ```
  segment-list R7_VIA_RSVP
   index 1 mpls label 16003
   index 2 mpls label 4005
   index 3 mpls label 16007
  !
  policy R7_VIA_RSVP
   color 10 end-point ipv4 7.7.7.1
   candidate-paths preference 100 explicit segment-list R7_VIA_RSVP
  ```
- Only the first label (16003, R3's Prefix-SID) is validated by R1 — label 4005 is trusted (Concept #7's
  "only first label validated" behavior).
- Flow: R1 pushes 3 labels toward R3. At R3, top label pops, revealing 4005 — R3's own tunnel BSID — which
  is popped and forwarded natively into the RSVP-TE tunnel (R3-R4-R6-R5). At tunnel egress, remaining
  label (16007) continues via ordinary SR/IGP forwarding.
- Verified via traceroute and RSVP-TE tunnel traffic counters.

**4. Real-world use case**
Standard technique for phased SR migrations — deploying new SR-TE services end-to-end immediately, even
while parts of the core remain RSVP-TE-only, without touching the legacy infrastructure's steering.

**5. Failure scenario**
If the RSVP-TE tunnel goes down without the BSID mapping being torn down cleanly, the SR-TE policy shows
up (only the first label is validated) while being completely broken past that point.

**6. Design insight**
BSID doesn't care what's behind it — a full SR-TE policy or a completely different technology's tunnel —
it's just an opaque label. This stitching technique should be a first-class transition tool for long
migrations, not a workaround.

**7. Interview-ready answer**
"You can allocate a BSID directly on an RSVP-TE tunnel interface, turning the whole tunnel into a single
opaque label an SR-TE explicit path can push. This lets an SR-TE policy span a legacy RSVP-TE-only segment
cleanly with zero autoroute-announce or static routes — the SR-TE headend only validates the first label
to reach the RSVP-TE headend, then trusts the tunnel's BSID for the rest."

---

## 12. SR-TE Autoroute Include — Per-Prefix IGP Injection, and the MPLS-to-MPLS FIB Gotcha

**1. Definition**
Makes an SR-TE policy appear as a link in the IGP graph for specific destination prefixes (or all), letting
uncolored traffic automatically use the policy — SR-TE's version of RSVP-TE autoroute-announce, but with
finer per-prefix granularity.

**2. Why it exists**
Sometimes an operator wants a policy to simply become "the path" for certain destinations without coloring
every relevant BGP route. Autoroute include supports this coarser steering style.

**3. How it works**
```
policy POL5
 color 10 end-point ipv4 5.5.5.1
 autoroute
  include ipv4 5.5.5.1/32
  include ipv4 9.9.9.1/32
 candidate-paths preference 100 dynamic
```
- `include all` mimics classic RSVP-TE autoroute-announce for every downstream prefix.
- **MPLS-to-MPLS FIB gotcha**: by default, even with autoroute configured, the policy is only used for
  IP-to-Label forwarding, NOT Label-to-Label (already-labeled traffic). The destination's algo-0
  Prefix-SID continues via the plain IGP path.
- Reason: algo-0 Prefix-SIDs mean "follow IGP shortest path," and since SR-TE policies aren't
  circuit-based, steering already-labeled traffic into one risks compounding, unpredictable redirection if
  a downstream router also has its own policy for the same SID.
- **Fix 1 — algo-1 (strict-SPF) Prefix-SID**:
  ```
  #R5
  router isis 1
   int lo1
    add ipv4 uni
     prefix-sid strict-spf index 105
  ```
  Removes the loop risk since algo 1 guarantees the literal IGP path with no further redirection
  possible. (Lab notes this still didn't work in testing — flagged as a possible platform bug, not a
  fundamental limitation.)
- **Fix 2 — force-sr-include**:
  ```
  policy POL5
   autoroute
    force-sr-include
  ```
  Successfully programs the label-swap entry, but causes a classic GRE-style route-recursion problem: the
  destination's Prefix-SID (16005) is used internally by the policy but is now also pointed at that same
  policy for outbound forwarding — a self-referential loop causing the **policy itself to flap**.
- Automated Steering always overrides autoroute — AS uses the policy's BSID for recursion (color-driven
  intent), while autoroute works off the plain RIB (a blunter mechanism).
- Static route alternative:
  ```
  router static
   address-family ipv4 unicast
    7.7.7.1/32 sr-policy srte_c_10_ep_5.5.5.1
  ```

**4. Real-world use case**
Smaller deployments or early SR-TE testing where full BGP-coloring infrastructure hasn't been built out
yet.

**5. Failure scenario**
Enabling force-sr-include to fix MPLS-to-MPLS forwarding caused the policy to actively flap — a
self-inflicted route-recursion loop directly analogous to a GRE tunnel reachable via itself.

**6. Design insight**
The lab's own conclusion: autoroute include and static-route-to-policy are explicitly discouraged in
production. The correct approach is always coloring BGP service routes and using Automated Steering/ODN —
intent-driven, not blunt RIB-based redirection that sweeps all uncolored traffic onto a policy regardless
of actual SLA fit.

**7. Interview-ready answer**
"Autoroute include makes an SR-TE policy appear as an IGP-graph link for included prefixes, but by default
only affects IP-to-Label forwarding, not MPLS-to-MPLS, because algo-0 Prefix-SIDs plus SR-TE's
non-circuit-based nature create real loop risk. Forcing this with force-sr-include can cause the policy
itself to flap from a GRE-style recursion loop — exactly why autoroute-include is an anti-pattern; the
correct approach is coloring BGP routes and using Automated Steering or ODN."

---

## Quick-Reference Summary Table

| # | Concept | Key Mechanism | Hidden Detail / Risk |
|---|---|---|---|
| 1 | SR-TE policy basics | Color + endpoint, candidate paths | Highest preference tried first (opposite RSVP-TE) |
| 2 | Multi-color AS | Highest color wins, auto fallback | additive needed on each set line separately |
| 3 | Dynamic metric types | IGP/TE/hopcount/latency | Only IGP metric gets ECMP (Algo-0 semantics) |
| 4 | Affinity constraints | include-all/any, exclude-any | Over-constraint can produce zero valid paths |
| 5 | Metric margin | absolute/relative tolerance | Requires equal underlying IGP cost to work at all |
| 6 | Candidate path fallback | Fails-only, not tie-based | BSID stays same across a fallback event |
| 7 | Explicit paths: addr vs label | Only first label validated | Broken extra label still shows policy "up" |
| 8 | Explicit paths are loose | No CSPF, no reduction | Missing final endpoint still shows "up" |
| 9 | Disjoint planes | Anycast SID as first hop | Doesn't scale; no ODN compatibility |
| 10 | BSID allocation | SRLB-based, conflict handling | Incompatible with ODN templates |
| 11 | SR/RSVP-TE stitching | BSID on tunnel-te interface | Tunnel BSID trusted, not independently validated |
| 12 | Autoroute include | IGP-graph injection per-prefix | force-sr-include can cause GRE-style policy flap |

