# CCDE-Level Segment Routing Deep Notes — Part 3 (Fast Reroute: LFA, RLFA, TI-LFA)
### Source labs: CCIE-SP v5.1 — LFA, LFA Tiebreakers (ISIS), Remote LFA, RLFA Tiebreakers, TI-LFA,
### Remote LFA or TI-LFA?, TI-LFA Node Protection, TI-LFA SRLG Protection,
### TI-LFA Protection Priorities (ISIS), TI-LFA Protection Priorities (OSPF)
### Compiled: 2026

> Continuation of the earlier CCDE SR deep-notes files (SRGB/PHP/ExpNull/Anycast, and IGP/BGP SR
> fundamentals). This file covers the full Fast-Reroute family: LFA, Remote LFA, and TI-LFA, written in
> plain, easy-to-learn language while keeping full protocol-level and design depth.

---

## Table of Contents
1. LFA (Loop-Free Alternate) — The Basic Idea
2. LFA Verification Flags (P, NP, TM, LC, D, SRLG) and Coverage Reporting
3. Excluding Interfaces from LFA Candidacy
4. LFA Tiebreakers (ISIS) — Controlling Which Backup Path Gets Chosen
5. Remote LFA (RLFA) — Using a Tunnel to Reach Further Backup Nodes
6. Remote LFA Limitations
7. Extended P-Space — Getting More Coverage from Remote LFA
8. TI-LFA (Topology-Independent LFA) — Fundamentals
9. TI-LFA's Dependency on Prefix-SIDs for Intermediate Nodes
10. TI-LFA's Adjacency-SID Selection — Why It Deliberately Uses the Unprotected Label
11. TI-LFA Node Protection
12. TI-LFA SRLG Protection
13. TI-LFA Protection Priority Ordering — and the Real Differences Between ISIS and OSPF

---

## 1. LFA (Loop-Free Alternate) — The Basic Idea

**1. Definition**
LFA is a simple, pre-computed backup path. Before any failure happens, the router already picks a backup
neighbor to send traffic to, so if the primary link or path goes down, it can switch to the backup
instantly, without waiting for the whole network to recalculate routes.

**2. Why it exists**
Normally, when a link fails, the IGP (ISIS/OSPF) has to notice the failure, tell every router in the
network, and every router has to recompute its shortest path — this is called convergence, and it can take
hundreds of milliseconds to seconds. During that time, traffic is dropped. For real-time traffic like
voice or financial trading, even 200ms of loss is a big problem. LFA solves this by having the backup path
ready and installed before the failure ever happens, so the switch-over is nearly instant (sub-50ms).

**3. How it works (with the real formula)**
- Every router picks one backup next-hop per destination prefix, in advance.
- Danger: what if the backup neighbor's own best path to the destination actually goes back through you?
  Then during a failure, it just sends the traffic right back to you — a loop.
- To avoid this, LFA uses an inequality check before accepting a neighbor as a valid backup:
  ```
  Dist(N, D) < Dist(N, PLR) + Dist(PLR, D)
  ```
  - N = the candidate backup Neighbor
  - D = the Destination prefix
  - PLR = Point of Local Repair (the router doing the protecting — you)
- In plain words: "the neighbor's own distance to the destination must be shorter than going through me."
  If true, the neighbor will never loop the packet back to you.

**Example from the lab:** Topology: R3 connects to R4 (normal best path to 10.10.10.1/32) and to R5
(candidate backup). All links cost 10.
- R3's cost to 10.10.10.1/32 (via R4) = 20
- R5's own cost to 10.10.10.1/32 (via R6) = 20
- R5's cost to R3 = 10
- Check: `20 < 10 + 20` -> `20 < 30` -> True -> R5 is a valid LFA.

```
router isis 1
 interface Gi0/0/0/4
  address-family ipv4 unicast
   fast-reroute per-prefix
```
Once enabled, the backup via R5 is pre-installed in the FIB as a repair route, ready to go the instant the
link fails.

**4. Real-world use case**
LFA is the simplest, cheapest form of fast-reroute — used anywhere sub-50ms protection is needed but the
topology doesn't justify the extra complexity of SR/TI-LFA (e.g., smaller access/aggregation rings, or as
a baseline protection layer before a full SR rollout).

**5. Failure scenario**
LFA doesn't always find a backup for every prefix — it depends entirely on the physical topology. In this
lab, enabling LFA on one interface protected just 1 of 4 prefixes (25%); a second interface brought
coverage to 50%. In many real topologies (rings, hub-and-spoke), a large percentage of prefixes simply
have no valid LFA neighbor — those prefixes fall back to slow convergence during a real failure, which is
a false sense of security if the operator assumes "LFA is on" means "everything is protected."

**6. Design insight**
LFA coverage is topology-dependent, not something you can just turn on and get everywhere. An architect
must actually calculate/simulate LFA coverage across the real topology before promising sub-50ms
protection as an SLA — this is exactly why Remote LFA and TI-LFA were later developed, to close the
coverage gap LFA leaves behind. Also worth noting: by default, /32 host routes get "medium" FRR priority,
while /30 interface prefixes get "low" priority.

**7. Interview-ready answer**
"LFA pre-computes a backup next-hop for a destination before any failure happens, using the inequality
`Dist(N,D) < Dist(N,PLR) + Dist(PLR,D)` to guarantee the backup neighbor won't loop traffic back to you.
It gives sub-50ms protection with zero extra protocol signaling, but coverage is entirely
topology-dependent — some prefixes simply have no valid LFA neighbor, which is why Remote LFA and TI-LFA
exist to close that gap."

---

## 2. LFA Verification Flags (P, NP, TM, LC, D, SRLG) and Coverage Reporting

**1. Definition**
`show isis fast-reroute` shows a set of flags describing the type of protection each backup path actually
provides — these flags tell you what you got, which is not always what you asked for.

**2. Why it exists**
An operator needs to know, for each protected prefix, exactly what kind of protection the backup path
gives — link-only, or does it also survive a full router failure. Without these flags, you'd have no
visibility into the real protection level.

**3. How it works**
- **P** = Primary path
- **NP** = Node Protecting — the backup path does not go through the same next-hop router
- **TM** = Total Metric via the backup path
- **LC** = Line Card disjoint — the backup uses a different line card than the primary
- **D** = Downstream — true if the backup neighbor's own cost to the destination is lower than yours
- **SRLG** = the backup path is SRLG-disjoint from the primary

Hidden detail: NP being "Yes" doesn't mean you configured node protection — with plain LFA, the router
just picks whatever valid backup exists based on the topology, and it happens to also be node-protecting
in some cases, purely by coincidence. You must configure tiebreakers (Concept #4) to guarantee it.

**4. Real-world use case**
Operators use `show isis fast-reroute summary` for a coverage report (e.g., "1 out of 4 prefixes
protected") as an audit tool before/after topology changes.

**5. Failure scenario**
An operator sees NP=Yes for a few important prefixes and assumes guaranteed node protection everywhere —
when really it was a topology coincidence, and other prefixes might have NP=No with no way to force it
without proper tiebreaker configuration.

**6. Design insight**
Treat these flags as a verification tool, not a guarantee mechanism. A mature design should audit FRR flag
coverage as part of every change-management process, not just once at initial buildout.

**7. Interview-ready answer**
"The FRR flags — P, NP, TM, LC, D, SRLG — tell you exactly what kind of protection a backup path provides.
Critically, seeing NP=Yes with plain LFA doesn't mean you configured node protection — it just happened to
be true for that topology. You need explicit tiebreakers to guarantee it stays true."

---

## 3. Excluding Interfaces from LFA Candidacy

**1. Definition**
You can tell a router to never use a specific interface as a backup path for a given protected link — a
hard exclusion, stronger than SRLG-disjointness.

**2. Why it exists**
SRLG-disjoint tiebreakers are a preference, not a rule — if no other backup exists, the router still falls
back to an SRLG-conflicting interface (some protection is better than none, in the router's logic).
Sometimes an operator knows for certain a specific interface must never carry backup traffic — e.g., a
low-bandwidth link that would be instantly overwhelmed, or a strict segregation requirement. Exclusion
gives an absolute guarantee, not just a preference.

**3. How it works**
```
router isis 1
 interface GigabitEthernet0/0/0/4
  address-family ipv4 unicast
   fast-reroute per-prefix exclude interface GigabitEthernet0/0/0/5
```
This tells the router: when protecting Gi0/0/0/4, never consider Gi0/0/0/5 as a backup, under any
circumstances. If Gi0/0/0/5 was the only valid LFA candidate, the result is no LFA at all — the exclusion
is absolute, even if it means losing protection entirely.

**4. Real-world use case**
Used when a specific low-bandwidth or high-cost backup link (e.g., an emergency-only satellite backhaul)
must never silently absorb fast-reroute traffic, since doing so could cause a secondary outage (congestion
collapse) on top of the original failure.

**5. Failure scenario**
An operator excludes an interface thinking "SRLG-disjoint will find another path anyway," not realizing
exclusion is absolute — if the excluded interface was the only backup candidate, protection is completely
lost, silently.

**6. Design insight**
Exclusion should be reserved for genuinely hard constraints, not used casually as "a stronger
SRLG-disjoint" — unlike a tiebreaker, exclusion can completely remove protection rather than just
deprioritizing a path.

**7. Interview-ready answer**
"Interface exclusion for LFA is an absolute rule, not a preference — unlike SRLG-disjoint tiebreakers,
which still fall back to a non-preferred path if nothing else exists, excluding an interface means it will
never be used, even if that means losing protection completely for that prefix."

---

## 4. LFA Tiebreakers (ISIS) — Controlling Which Backup Path Gets Chosen

**1. Definition**
When more than one valid LFA backup path exists for a prefix, tiebreakers are the rules the router uses to
decide which one to actually pick — you can customize this order to match what your network cares about
(lowest cost, node safety, hardware diversity, etc.).

**2. Why it exists**
Different networks have different priorities — one cares most about avoiding shared physical risk (SRLG),
another cares most about surviving a full node failure, another just wants the cheapest backup regardless.
Tiebreakers let the architect encode the network's actual protection philosophy into the router's decision
logic.

**3. How it works**
Default ISIS tiebreaker order (lower index = more preferred):
- **Primary-path (index 10)**: if ECMP paths exist, use the other ECMP path as backup
- **Lowest-backup-metric (index 20)**: pick whichever backup has the lowest total IGP cost
- **Line-card-disjoint (index 30)**: pick the backup using a different line card
- **Node-protecting (index 40)**: pick the backup that avoids your next-hop router entirely

Each rule is only evaluated if the previous rule didn't already pick a clear winner.

**Worked example:** R3 has three possible LFAs for 7.7.7.1/32: via R9 (metric 30, not node-protecting),
via R6 (metric 35, node-protecting), via R4 (metric 70, not node-protecting).
- Default tiebreakers: R3 picks R9 (lowest-backup-metric wins by default).
- Reconfigured with node-protecting first:
  ```
  router isis 1
   address-family ipv4 unicast
    fast-reroute per-prefix tiebreaker node-protecting index 10
    fast-reroute per-prefix tiebreaker srlg-disjoint index 20
    fast-reroute per-prefix tiebreaker lowest-backup-metric index 30
  ```
  Now R3 picks R6 instead, even though it's a worse-metric path.

**Additional tiebreaker types:**
- **SRLG-disjoint**: prefers a backup not sharing the same SRLG value — but only considers your own local
  interface SRLG tags, not remote routers' SRLG values.
- **Secondary-path**: prefers a non-ECMP backup over one that's simply the other member of an ECMP set.
- **Downstream**: prefers a backup whose own IGP cost to the destination is lower than the PLR's own cost.
  This can override a pure lowest-metric choice — in the lab, R4 was the only downstream backup but had a
  high-cost link from R3, so without this tiebreaker R3 preferred a cheaper, non-downstream backup via R6;
  adding downstream flipped the choice to R4.

**4. Real-world use case**
A network with dual power feeds, dual line cards, and physically diverse fiber paths would typically
configure node-protecting and SRLG-disjoint tiebreakers ahead of pure cost, to actually exploit the
physical diversity that was engineered in.

**5. Failure scenario**
An operator assumes lowest-backup-metric is always safest (cheapest = best) without realizing the cheapest
backup might not survive a full node failure — during an actual router crash, a non-node-protecting backup
still tries to loop traffic toward the dead router until the IGP fully reconverges.

**6. Design insight**
Tiebreaker configuration is where an architect's risk philosophy gets encoded into the network. Also
critical: custom tiebreakers completely overwrite the default order — a partial configuration might
unintentionally drop a tiebreaker you actually wanted.

**7. Interview-ready answer**
"ISIS LFA tiebreakers decide which backup path wins when multiple valid LFAs exist, evaluated in index
order until one rule produces a clear winner. The default order prefers cheapest cost first, which can
silently pick a backup that isn't node-protecting — so any network that cares about surviving a full
router failure must explicitly reorder tiebreakers to put node-protecting ahead of cost."

---

## 5. Remote LFA (RLFA) — Using a Tunnel to Reach Further Backup Nodes

**1. Definition**
Remote LFA extends basic LFA by tunneling traffic (using LDP) past your directly-connected neighbor to a
further-away router, so you're not limited to only directly-connected routers when looking for a safe
backup.

**2. Why it exists**
Plain LFA only works if a directly-connected neighbor satisfies the loop-free inequality. In many
topologies (especially rings), none of your direct neighbors qualify. RLFA solves this by wrapping the
packet in an MPLS label that forces the neighbor to just forward it further, all the way to a genuinely
safe node further out, without the neighbor making its own IP routing decision.

**3. How it works — P-space / Q-space**
- **P-space**: nodes the PLR can reach without using the failed link.
- **Q-space**: nodes that can reach the destination without using the failed link.
- **PQ node**: a node in both spaces — the safe point where the PLR can release the packet.

**Worked example:** R3 protecting the link to R4, for 4.4.4.1/32 (owned by R4). P-space for R3 = {R5, R6}.
Q-space = {R10, R6}. The PQ node = **R6**.
- R3 tunnels using R5's LDP label for R6 (forcing an MPLS decision, not IP).
  ```
  #All routers
  router isis 1
   address-family ipv4 unicast
    mpls ldp auto-config
  mpls ldp

  #R3
  router isis 1
   interface GigabitEthernet0/0/0/4
    address-family ipv4 unicast
     fast-reroute per-prefix
     fast-reroute per-prefix remote-lfa tunnel mpls-ldp
  ```
- **Critical hidden detail**: R3 and the PQ node need a targeted LDP (tLDP) session so R3 learns R6's own
  label for the final destination.
- **If the PQ node doesn't accept targeted LDP** (common misconfiguration): R3 can still install a backup
  path for plain IP traffic — but the label stack only has ONE label. This is dangerous for MPLS services:
  a VPN packet's inner label gets exposed too early when the transit router pops the single top label,
  breaking the end-to-end LSP.
- **Fix:**
  ```
  #R6
  mpls ldp
   address-family ipv4
    discovery targeted-hello accept
  ```
  Once tLDP forms, the backup path correctly uses two labels: `<R5's label for R6>` / `<R6's label for
  4.4.4.1/32>`.

**4. Real-world use case**
RLFA was the standard FRR mechanism widely deployed in LDP-based MPLS cores before SR/TI-LFA became
common — especially useful in ring topologies where plain LFA coverage is poor.

**5. Failure scenario**
An operator enables RLFA, sees a backup path installed (looks fine), but never checks whether the PQ node
accepts targeted LDP — the single-label backup works for plain IP but breaks MPLS VPN services riding over
the same repair path. This requires checking the FIB label stack, not just route presence, to catch.

**6. Design insight**
Any RLFA deployment carrying MPLS services must verify targeted LDP session establishment with every PQ
node used — a standard post-deployment audit step. Strong argument for migrating to TI-LFA, which removes
this tLDP-dependency risk entirely.

**7. Interview-ready answer**
"Remote LFA tunnels traffic via LDP to a PQ node — a router in both your P-space and the destination's
Q-space. The dangerous hidden detail is that if the PQ node doesn't accept targeted LDP sessions, you
still get a working backup path, but with only one label instead of two — silently breaking end-to-end
MPLS service LSPs riding over that repair path, even though plain IP traffic looks fine."

---

## 6. Remote LFA Limitations

**1. Definition**
RLFA has structural limits: it cannot provide node protection or SRLG protection, doesn't use the true
post-convergence path, doesn't scale well, and doesn't always achieve full coverage.

**2. Why it exists**
Understanding RLFA's limits tells an architect exactly when they must move to TI-LFA instead of just
tuning RLFA further — these are hard ceilings baked into how the P/Q-space mechanism works.

**3. How it works — each limitation**
- **No node or SRLG protection**: proven directly in the lab — forcing a node-protecting tiebreaker on
  RLFA had zero effect; RLFA kept using the same LFA regardless.
- **Doesn't use the true post-convergence path**: RLFA's tunnel is based on the current, pre-failure
  LDP-derived paths, not a recalculation of the actual post-failure shortest path — real risk of momentary
  congestion on the repair path.
- **Requires targeted LDP sessions**: an operational dependency on every potential PQ node.
- **Doesn't guarantee 100% coverage**: depends on a PQ node actually existing for every protected prefix.

**4. Real-world use case**
Operators evaluating RLFA vs. TI-LFA should explicitly test for these gaps — if node protection matters,
RLFA is fundamentally the wrong tool.

**5. Failure scenario**
An operator needing guaranteed node protection deploys RLFA and assumes tiebreaker tuning will eventually
get them there — burning engineering time on something RLFA structurally cannot provide.

**6. Design insight**
Know your tool's structural ceiling, not just its configuration knobs — RLFA is not a strictly worse
version of TI-LFA that just needs more tuning; it's a fundamentally more limited mechanism.

**7. Interview-ready answer**
"RLFA can only ever provide link protection, never node or SRLG protection, no matter how you configure
tiebreakers — proven directly by testing. It also doesn't use the true post-convergence path and depends
on targeted LDP sessions with every PQ node — exactly the gaps TI-LFA was built to close."

---

## 7. Extended P-Space — Getting More Coverage from Remote LFA

**1. Definition**
Extended P-space lets RLFA reach further than the PLR's own direct P-space, by borrowing the P-space of
the PLR's own neighbor.

**2. Why it exists**
In some topologies, the PLR's direct P-space and the destination's Q-space don't overlap — no PQ node
exists using the basic model. But since the PLR is tunneling (MPLS decision, not IP), it can tunnel to a
neighbor safely in its own P-space, and use that neighbor's P-space to extend reach by one more hop.

**3. How it works (worked example)**
- R3 protects the link to R4. R3's normal P-space = {R5, R6}. R3 cannot safely reach R10 directly (ECMP
  paths exist, some crossing the protected link).
- R3 tunnels to R5 first (safely in its own P-space). R5's own P-space includes R10.
- R10 becomes usable as a PQ node via this "borrowed" P-space.
- Label stack: `<R5's label for R10>` / `<R10's label for 4.4.4.1/32>`.
- **Feasibility threshold**: works as long as the ring-around alternate path stays cheaper than the
  protected link. In the lab, the alternate path (R10-R6-R5-R3-R4) costs 40 — once the direct R4-R10 link
  metric reaches 40, R10 gets ECMP paths and stops being a genuine Q-node, breaking RLFA entirely.

**4. Real-world use case**
Extended P-space is what makes RLFA viable in more topologies than the naive P/Q-space model suggests —
automatic, no extra configuration needed.

**5. Failure scenario**
As link metrics change over time, a previously-valid extended-P-space backup can silently stop being
valid — exactly as shown when the R4-R10 metric was raised to 40.

**6. Design insight**
Strong argument for automated FRR coverage auditing rather than manual verification — feasibility depends
on a metric threshold that can be silently crossed by an unrelated metric change made by a different team.

**7. Interview-ready answer**
"Extended P-space lets an RLFA-protecting router borrow its own neighbor's P-space, finding a valid PQ
node even when the router's own P-space doesn't overlap the destination's Q-space. But feasibility depends
on a specific metric threshold that can silently be crossed by an unrelated metric change elsewhere,
quietly removing protection."

---

## 8. TI-LFA (Topology-Independent LFA) — Fundamentals

**1. Definition**
TI-LFA is a fast-reroute mechanism built on Segment Routing that guarantees a backup path exists for every
destination, no matter what the topology looks like — and that backup always matches the exact
post-convergence path the network would use anyway.

**2. Why it exists**
LFA leaves coverage gaps, and RLFA still has real limits: no node/SRLG protection, not the true
post-convergence path, and PQ-node dependency. TI-LFA eliminates all of these using SR's Adjacency-SIDs,
which let a router force traffic out one specific link regardless of the normal IGP shortest path —
something LDP cannot do.

**3. How it works**
- Compare to RSVP-TE FRR (facility-based): a single backup path avoids the failed facility, and all
  repaired traffic must be stitched back onto the original tunnel's next-hop, even if that means hairpin
  routing, because RSVP-TE is circuit-based.
- TI-LFA (like LFA/RLFA) is prefix-based: an independent, optimal backup computed per destination, no
  signaling needed.
- **Core computation**: TI-LFA removes the protected facility (link, node, or SRLG) from its internal
  topology view, runs CSPF, finding the genuine post-convergence path. It then builds the shortest SID
  list to steer traffic exactly along that path.
- **ECMP support built in**: since the repair path often uses a Prefix-SID, which naturally load-balances
  across ECMP, TI-LFA gets ECMP for free; multiple P/Q combinations get load-shared across prefixes.
- **Deep-stack case**: if more than 3 labels are needed, the router internally builds an SR-TE policy and
  steers the repair path through it, since the FIB can't push that many labels directly. Rare — 99% of the
  time no more than two labels are needed.
- **Protects plain IP and LDP too**: SR can be enabled purely for protection (sr-prefer off), with LDP/IP
  handling normal traffic. The destination must have a Prefix-SID (no tLDP with the Q-node like RLFA), and
  any intermediate nodes used for steering also need SR enabled.

**Worked example:** R3 protects the link to R4. R4-R10 link cost raised so high no PQ node exists (RLFA
would fail). With TI-LFA: R3 removes the R3-R4 link, runs CSPF, finds post-convergence path R3->R10->R4.
R3 pushes R10's Adjacency-SID for its link to R4 (24005) — forcing R10 to use that exact link.

**4. Real-world use case**
TI-LFA is the current industry-standard FRR mechanism in modern SR-based SP and hyperscale networks — 100%
topology coverage without RSVP-TE signaling overhead or RLFA's tLDP dependency.

**5. Failure scenario**
If SR isn't consistently enabled across every potential intermediate node, TI-LFA cannot build a repair
path through that node — silently reducing coverage without an obvious cause.

**6. Design insight**
TI-LFA's biggest win: it decouples "does a safe backup exist" from "does the topology happen to have a
directly-usable neighbor or PQ node" — with Adj-SIDs, the router can force traffic along any loop-free
path it can compute, not just paths that already exist as normal forwarding decisions.

**7. Interview-ready answer**
"TI-LFA removes the protected link, node, or SRLG from its topology view, runs CSPF to find the genuine
post-convergence path, and builds the shortest SID list to steer traffic along it using Adjacency-SIDs.
This gives 100% topology coverage and the true post-convergence path, unlike LFA or RLFA, and can even
protect plain LDP/IP traffic if SR is enabled purely for backup purposes."

---

## 9. TI-LFA's Dependency on Prefix-SIDs for Intermediate Nodes

**1. Definition**
For TI-LFA to route a repair path through a transit node, that node must have a usable Prefix-SID the PLR
can correctly identify — each IGP has its own rule for which prefix on a node counts as "the node's"
Prefix-SID.

**2. Why it exists**
TI-LFA needs to represent "go to this node" as one segment in the SID list, but a router can have many
prefixes — the protocol needs a deterministic rule for picking exactly one.

**3. How it works**
- **ISIS**: uses the TE Router ID TLV (134) by default. If TLV 134 doesn't exist, falls back to TLV 132
  (IP interface address), and that prefix must have a Prefix-SID.
- **OSPF**: checks the RID directly for a Prefix-SID; if not found, looks for the highest reachable host
  address with the N-flag set.
- **Anycast SID consequence**: this is exactly why an Anycast SID with the N-flag left set can break
  TI-LFA — the node-identification logic can incorrectly treat the anycast SID as identifying one single
  node.
- **Safe default**: assign a Prefix-SID to the same loopback used as the router ID — avoids all ambiguity.

**4. Real-world use case**
Matters most in networks with multiple loopbacks per router (IGP RID, BGP, management) — the specific
loopback used as RID must have the Prefix-SID, even if others don't.

**5. Failure scenario**
A router has multiple loopbacks, and the Prefix-SID is assigned to a different loopback than the RID.
TI-LFA silently cannot use that node as an intermediate hop, breaking coverage for destinations needing
that path, with no obvious error pointing at the cause.

**6. Design insight**
Strong argument for a network-wide standard: always use the same loopback for both the IGP router ID and
the Prefix-SID.

**7. Interview-ready answer**
"TI-LFA needs a Prefix-SID for every intermediate node it might route through. ISIS uses the TE Router ID
TLV (134), falling back to TLV 132; OSPF checks the RID, then the highest reachable host prefix with the
N-flag set. This is exactly why an Anycast SID with the N-flag left set can silently break TI-LFA — best
practice is always assigning the Prefix-SID to the same loopback used as the router ID."

---

## 10. TI-LFA's Adjacency-SID Selection — Why It Deliberately Uses the Unprotected Label

**1. Definition**
When TI-LFA needs to force traffic through a Q-node's link using an Adjacency-SID, and that Q-node has
both a protected and unprotected Adj-SID for the same link, the PLR deliberately picks the unprotected
one — intentional design, not a bug.

**2. Why it exists**
Prevents a microloop. If the PLR used the protected Adj-SID, and that same link also failed (a second,
unrelated failure close in time), the network could enter a temporary loop while two independent FRR
mechanisms try to protect overlapping failures at once.

**3. How it works**
- R3 protects its link to R4, using R10 as the Q-node.
- When TI-LFA is enabled on R10's own link to R4, R10 advertises two Adj-SIDs: unprotected and
  FRR-eligible.
- R3 always picks the unprotected Adj-SID for R10's link — confirmed even after removing and re-adding
  TI-LFA to rule out stale state.
- Reasoning (per Nick Russo's analysis cited in the lab): TI-LFA is already the protection here; layering
  the Q-node's own local FRR on top would create unnecessary microloop risk.

**4. Real-world use case**
Purely internal, automatic behavior — important to understand when reading SID-list/CEF output during
troubleshooting.

**5. Failure scenario**
If TI-LFA carelessly used the protected Adj-SID and a second unrelated failure hit that same link while
the first repair was active, a genuine transient routing loop could occur during double-reconvergence.

**6. Design insight**
Good example of engineering thought already built into TI-LFA to avoid compounding-failure edge cases —
no configuration needed, but important to understand as a safety property.

**7. Interview-ready answer**
"When a Q-node has both a protected and unprotected Adjacency-SID for the same link, TI-LFA deliberately
uses the unprotected one, specifically to avoid microloops during a second, overlapping failure — TI-LFA
intentionally keeps its own protection as the single layer handling that repair."

---

## 11. TI-LFA Node Protection

**1. Definition**
By default, TI-LFA only guarantees protection against a link failure. Node protection is an optional
add-on that makes TI-LFA compute a backup path that survives even if the entire next-hop router fails.

**2. Why it exists**
Link failure and node failure are different failure modes — a fiber cut takes out one link, but a power
supply failure or crash takes out the entire router. Default TI-LFA only removes the link from its
computation, not the node — the stronger guarantee must be explicitly requested.

**3. How it works**
- Default (link protection only): backup might still route through the same next-hop router via a
  different link — if that router dies, the "protection" does nothing.
- Enable real node protection:
  ```
  router isis 1
   address-family ipv4 unicast
    fast-reroute per-prefix tiebreaker node-protecting index 10
   !
   interface GigabitEthernet0/0/0/5
    address-family ipv4 unicast
     fast-reroute per-prefix
     fast-reroute per-prefix ti-lfa
  ```
- With this, TI-LFA first tries a backup with the entire node excluded from the topology, even at a higher
  metric than the simple link-protecting alternative.
- If only one protection type is enabled (just node-protecting, nothing else), the index value doesn't
  matter — it always beats "no extra protection."
- Fallback: if no node-protecting path exists, TI-LFA falls back to plain link protection — never fails to
  protect entirely.
- **Worked example**: R3's link to R5, by default, uses a link-protecting backup via R9. After enabling
  node-protecting, R3 computes a path via R6-R7 instead, using an Adjacency-SID to force R6 out its
  specific high-metric link toward R7 (R6's own label is Implicit-Null since it's directly connected).

**4. Real-world use case**
Expected standard for any serious SP core protecting critical P/PE routers — link protection alone is
insufficient for production backbone design.

**5. Failure scenario**
An operator enables TI-LFA but forgets the node-protecting tiebreaker, assuming TI-LFA alone implies full
protection — during a full router crash, the default backup still routes through the dead router,
resulting in a full outage exactly during the failure type they thought they were protected against.

**6. Design insight**
Critical distinction every architect must know: "TI-LFA is enabled" is not the same as "we have node
protection." Always check for the node-protecting tiebreaker explicitly, not just the presence of ti-lfa.

**7. Interview-ready answer**
"TI-LFA by default only protects against link failures. To get real node protection, you add the
node-protecting tiebreaker, which makes the router first try a backup path with the entire next-hop node
excluded, falling back to link-only protection if none exists. Enabling TI-LFA alone does not guarantee
you survive a full router crash."

---

## 12. TI-LFA SRLG Protection

**1. Definition**
SRLG protection makes TI-LFA compute a backup path avoiding any link sharing the same physical risk group
as the protected link — for example, avoiding a fiber running through the same physical conduit.

**2. Why it exists**
A backup being logically "a different link" doesn't mean it's physically independent — if primary and
backup share the same conduit, one excavation accident takes out both. SRLG tags let the router actively
avoid a backup sharing the same physical risk.

**3. How it works**
```
#R3
srlg
 interface GigabitEthernet0/0/0/2
  name RED
 !
 ...
 name RED value 1
!
router isis 1
 address-family ipv4 unicast
  fast-reroute per-prefix tiebreaker srlg-disjoint index 10
```
- The router prunes all same-SRLG links, then runs CSPF over what's left. Falls back to plain link
  protection if no SRLG-disjoint path exists.
- **Critical limitation**: SRLG checking is local-only — the router has no visibility into remote routers'
  SRLG tags. True end-to-end diversity requires consistent tagging across every router in the path.
- **Combining with node protection**: fallback order is (1) both node + SRLG protection, (2) whichever
  single type has the better index, (3) plain link protection.
- **Important gotcha**: lowest-backup-metric has zero effect on TI-LFA — confirmed directly in the lab, it
  changed nothing about TI-LFA's chosen path even while it did change the separate plain-LFA calculation
  on the same router, because TI-LFA's whole point is the genuine post-convergence path.

**4. Real-world use case**
Any physically-diverse network design (dual DC interconnects, submarine cables, metro fiber rings with
known shared-duct segments) should tag SRLG accurately and enable srlg-disjoint tiebreakers.

**5. Failure scenario**
An operator tags SRLG correctly on the PLR but forgets checking is local-only — a backup path might be
locally SRLG-disjoint but still cross a remote link sharing physical infrastructure with the primary path.

**6. Design insight**
SRLG protection is only as good as the physical-layer documentation feeding into the tags — a data-quality
problem as much as a config problem. Stale SRLG tags after a physical re-route are worse than no SRLG
protection (false confidence).

**7. Interview-ready answer**
"SRLG protection tells TI-LFA to avoid backup paths sharing physical fate with the protected link, by
pruning same-SRLG links before CSPF, falling back to link protection if none exists. Key limitations: SRLG
checking is local-only, and the lowest-backup-metric tiebreaker has zero effect on TI-LFA, since TI-LFA's
entire purpose is following the genuine post-convergence path, not the cheapest one."

---

## 13. TI-LFA Protection Priority Ordering — and the Real Differences Between ISIS and OSPF

**1. Definition**
When multiple TI-LFA protection types (node, SRLG) are configured together, the router follows a specific
fallback order to decide the final backup path — and this ordering logic, along with configuration
inheritance, works differently enough between ISIS and OSPF that assuming parity will cause real mistakes.

**2. Why it exists**
Networks often want both node and SRLG protection simultaneously, with a clear priority when both can't be
satisfied. ISIS and OSPF implemented these rules independently, with genuine behavioral differences.

**3. How it works — shared logic**
1. Try to find a path satisfying both node protection AND SRLG-disjointness.
2. If none exists, use whichever single type has the higher-priority index (ISIS: lower index wins; OSPF:
   higher index wins — opposite directions).
3. If neither can be satisfied, fall back to plain link protection.

**How it works — the real differences**
- **Index direction reversed**: ISIS prefers lower index; OSPF prefers higher index.
- **AFI-level vs. interface-level merging**: in ISIS, interface-level tiebreakers completely replace
  AFI-level ones (no merging) — configuring node-protecting only at the interface drops any AFI-level
  srlg-disjoint for that interface. In OSPF, interface-level and process/area-level configuration merge
  together.
- **Resetting to default**: ISIS uses `fast-reroute per-prefix tiebreaker default` to reset all tiebreakers
  on an interface. OSPF has no equivalent — use `... node-protecting disable` and `... srlg-disjoint
  disable` individually (sets index to 0; you cannot type "index 0" directly).
- **lowest-backup-metric / lc-disjoint effect genuinely different**: zero effect on ISIS's TI-LFA, but DOES
  affect OSPF's TI-LFA — the lab shows placing lowest-backup-metric above node-protecting in OSPF changes
  the chosen backup path (from R6 to R9).
- **OSPF has a hidden, always-on tiebreaker**: "Post Convergence Path" at index 256, not configurable, not
  seen for plain LFA — appears to be OSPF's internal mechanism guaranteeing TI-LFA always calculates the
  genuine post-convergence path.
- **OSPF supports extra tiebreaker types**: lc-disjoint and a meaningfully-effective lowest-backup-metric,
  whereas ISIS's TI-LFA tiebreaker set is effectively just node-protecting and srlg-disjoint.

**4. Real-world use case**
Organizations running mixed ISIS/OSPF environments (common after mergers) must maintain protocol-specific
runbooks for TI-LFA tiebreaker configuration — copy-pasting a template between protocols without adjusting
produces silently wrong protection priorities.

**5. Failure scenario**
An engineer configures node-protecting at index 10 and srlg-disjoint at index 20 correctly on ISIS (node
protection preferred), then uses identical values on OSPF — accidentally making SRLG protection preferred
instead, since OSPF's index direction is inverted.

**6. Design insight**
Never assume behavioral parity between IGPs just because feature names and syntax look similar. Build and
maintain a side-by-side comparison table of these divergences as living documentation.

**7. Interview-ready answer**
"TI-LFA priority ordering works conceptually the same in ISIS and OSPF — try both, then priority fallback,
then link protection — but the mechanics diverge: ISIS prefers lower index, OSPF prefers higher; ISIS
interface tiebreakers replace AFI-level ones, OSPF merges them; and critically, lowest-backup-metric has
zero effect on ISIS's TI-LFA but does affect OSPF's, alongside OSPF's extra lc-disjoint option and hidden
always-on Post Convergence Path tiebreaker."

---

## Quick-Reference Summary Table

| # | Concept | Key Mechanism | Hidden Detail / Risk |
|---|---|---|---|
| 1 | LFA basics | `Dist(N,D) < Dist(N,PLR)+Dist(PLR,D)` | Coverage is topology-dependent, often partial |
| 2 | LFA flags | P/NP/TM/LC/D/SRLG | NP=Yes can be coincidence, not guaranteed |
| 3 | Interface exclusion | Hard exclude vs SRLG preference | Can remove protection entirely if last option |
| 4 | LFA tiebreakers | 7 types, index order | Default prefers cost over node safety |
| 5 | Remote LFA | P/Q-space, PQ node, LDP tunnel | Missing tLDP = 1 label = broken VPN LSP |
| 6 | RLFA limits | Structural, not tunable | No node/SRLG protection, no post-convergence path |
| 7 | Extended P-space | Borrow neighbor's P-space | Feasibility tied to a silent metric threshold |
| 8 | TI-LFA fundamentals | CSPF minus facility, Adj-SID steering | >3 labels auto-uses SR-TE policy internally |
| 9 | TI-LFA node Prefix-SID | TLV 134/132 (ISIS), RID/N-flag (OSPF) | Wrong loopback = silent coverage loss |
| 10 | TI-LFA Adj-SID choice | Always picks unprotected Adj-SID | Prevents microloop on double failure |
| 11 | TI-LFA node protection | Exclude node from CSPF | "TI-LFA on" != node protection |
| 12 | TI-LFA SRLG protection | Prune same-SRLG links | Local-only; lowest-backup-metric has no effect |
| 13 | ISIS vs OSPF priorities | Fallback order, index direction | Index direction inverted; merging behavior differs |

