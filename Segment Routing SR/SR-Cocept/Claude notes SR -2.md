# CCDE-Level Segment Routing Deep Notes — Part 2
### Source labs: CCIE-SP v5.1 — Basic SR (ISIS/OSPF), Adjacency-SID, LAN Adj-SID, SR/RSVP-TE Interaction,
### Inter-Area (ISIS/OSPF), Inter-IGP Redistribution, Inter-AS BGP, BGP Data Center (eBGP/iBGP)
### Compiled: 2026

> Continuation of the first CCDE SR deep-notes file (SRGB, PHP, ExpNull, Anycast SID). This file covers
> the foundational SR-with-IGP mechanics, Adjacency-SID internals, SR/RSVP-TE coexistence, multi-area,
> multi-IGP, and multi-AS/BGP-based SR designs, extracted from 11 additional lab pages.

---

## Table of Contents
1. Why Segment Routing Replaces LDP and RSVP-TE (Core Philosophy)
2. Prefix-SID Configuration Mechanics — Index vs. Absolute Value
3. ISIS Prefix-SID Sub-TLV Flags (Authoritative Definitions + Hidden Behavior)
4. ISIS Router Capabilities TLV (242) — RID Selection, SR-Capability, SR-Algorithm, Max SID Depth
5. SR Algorithms — Algorithm 0 (SPF) vs. Algorithm 1 (Strict SPF)
6. Adjacency-SID Fundamentals — Dynamic Allocation, FRR/Non-FRR Pair
7. Adjacency-SID Flags, Persistence, and Static Allocation via SRLB
8. LAN Adjacency-SID — ISIS Pseudonode Model vs. OSPF DR/BDR Model
9. OSPF SR Control Plane — Opaque LSA Types (1, 4, 7, 8)
10. OSPF vs. ISIS — Structural Differences in SR Implementation
11. OSPF Prefix-SID Flags — Extended Prefix TLV (A, N) and SID Sub-TLV (NP, M, E, V, L)
12. OSPF Adjacency-SID Flags — Extended Link TLV (B, V, L, S)
13. SR-TE Algorithm Interaction with RSVP-TE (Autoroute-Announce Hijacking Algo-0 Traffic)
14. Inter-Area SR Propagation — Automatic Behavior, and Why No-PHP Without Explicit-Null Matters
15. OSPF Inter-Area SR — Type-3 Route-Type, A-Flag Semantics, and ABR Loop-Prevention
16. Inter-IGP SR Redistribution — RIB-Driven Index Derivation and the SRGB-Match Requirement
17. SR for BGP — BGP Prefix-SID Attribute as a Label-Allocation "Hint" over BGP-LU
18. BGP-SR Operational Pitfalls — allocate-label all, eBGP Local-Peer Label Allocation, and PE/ASBR Ordering Bugs

---

## 1. Why Segment Routing Replaces LDP and RSVP-TE (Core Philosophy)

**1. Definition**
SR is not a separate control-plane protocol — it is a set of extensions carried inside the IGP itself
(ISIS/OSPF) that let routers build MPLS label-switched paths without any dedicated label-signaling
protocol.

**2. Why it exists**
LDP requires a completely separate session per adjacency, plus the operational burden of LDP-IGP
synchronization (making sure LDP has converged before the IGP considers a link usable — a common source of
transient black-holing during convergence, since two independent protocols must reach consistent state).
RSVP-TE solves TE but at a heavy cost: it requires soft-state refresh (periodic PATH/RESV re-signaling to
keep tunnel state alive — a scaling tax that grows with tunnel count), a conceptually full-mesh of tunnels
for any-to-any TE, cannot natively follow ECMP (an RSVP-TE LSP is a single deterministic path, not a
load-shared set), and its Fast Reroute (FRR) mechanism injects even more signaling state into the network
for backup paths. SR was designed to solve both problems: give you LDP's simplicity (all state rides the
IGP) plus RSVP-TE's traffic-engineering power (explicit paths via label stacks) — without soft state,
without a full mesh, and with native ECMP awareness.

**3. How it works**
- Every SR node floods its Prefix-SID(s) and Adjacency-SID(s) through the IGP's existing reliable flooding
  mechanism (LSP for ISIS, LSA for OSPF).
- Because every node in the area learns every other node's SIDs, any node can independently compute an
  explicit end-to-end path as a label stack (list of Prefix-SIDs and/or Adjacency-SIDs) with zero
  additional signaling — no PATH message, no RESV message, no per-LSP state maintained hop-by-hop in the
  core.
- Because Prefix-SIDs by default follow the IGP shortest path (and, critically, prefix-SID-based paths
  naturally load-balance across ECMP next-hops the same way plain IP forwarding does), SR-TE paths are
  ECMP-aware "for free" — something RSVP-TE structurally cannot do.
- TI-LFA (Topology-Independent Loop-Free Alternate) uses the same flooded SID information to pre-compute
  backup paths, eliminating the need for RSVP-TE's FRR and its associated extra signaling state.

**4. Real-world use case**
Any modern SP or hyperscale core migrating off legacy MPLS-TE — this is the dominant justification cited
industry-wide for SR adoption: eliminating the RSVP-TE control-plane scaling ceiling (soft-state refresh
load, full-mesh tunnel count = O(n^2)) while retaining explicit-path TE capability via SR-TE
policies/PCE.

**5. Failure scenario**
Networks that don't fully commit to SR and instead run SR and RSVP-TE side-by-side indefinitely (rather
than as a migration bridge) inherit the operational complexity of both systems simultaneously — two
different path-computation logics, two different failure/protection models, and unexpected interactions
where RSVP-TE's autoroute-announce can silently hijack SR Algo-0 traffic into a tunnel the operator didn't
intend (see Concept #13).

**6. Design insight**
The strategic architecture lesson: SR's value isn't "a faster LDP," it's the elimination of soft state as
a scaling constraint. An architect evaluating SR vs. RSVP-TE for a new TE deployment should weight this
heavily — SR-TE's state lives only at the ingress node (as an encoded label stack), not maintained
hop-by-hop, which is what allows SR-TE policy counts to scale to thousands of paths without the RSVP-TE
soft-state refresh tax growing linearly with tunnel count.

**7. Interview-ready answer**
"SR eliminates LDP's separate signaling session (and the IGP-sync problem that comes with it) and
eliminates RSVP-TE's soft-state refresh, full-mesh requirement, and non-ECMP-awareness — by carrying all
label information inside the IGP's existing flooding, so any node can compute an explicit label-stack path
with zero additional protocol state, and TI-LFA replaces RSVP-TE FRR without extra signaling."

---

## 2. Prefix-SID Configuration Mechanics — Index vs. Absolute Value

**1. Definition**
A Prefix-SID is enabled per-loopback with a single command specifying either an index (an offset into the
SRGB) or an absolute label value (which the router silently converts into an index for IGP advertisement).

**2. Why it exists**
Configuration needs to be both operator-friendly (some engineers prefer to think in terms of a specific
memorable label, e.g., 16001) and protocol-correct (SR fundamentally only ever advertises an index, never a
raw label). IOS-XR supports both syntaxes as a convenience, silently doing the label-to-index math for you.

**3. How it works**
```
router isis 1
 address-family ipv4 unicast
  segment-routing mpls
 interface Loopback1
  address-family ipv4 unicast
   prefix-sid index 1
```
- `prefix-sid index 1` -> advertised index = 1 directly.
- `prefix-sid absolute 16001` (equivalent alternative) -> the router computes
  `16001 - local SRGB base (16000) = index 1` and advertises index 1 into the IGP — the absolute value
  never actually appears on the wire as an index; it's purely a local configuration convenience.
- The hidden danger: if you later change the local SRGB (e.g., move it to 30000-30999) without updating
  the absolute-value configuration, the previously-valid absolute value (16001) no longer falls inside the
  new SRGB range. The router cannot compute a valid index for it anymore, and the Prefix-SID simply stops
  being advertised — silently breaking every SR-TE path or LFIB entry that depended on that prefix's SID,
  with no direct warning tying the outage back to the SRGB change.

**4. Real-world use case**
Large operators standardize on index-based configuration exclusively in production for exactly this
reason — index survives SRGB changes gracefully (the label just moves with the new SRGB base), while
absolute-value configuration creates a hidden landmine tied to the current SRGB.

**5. Failure scenario**
An engineer configures `prefix-sid absolute 16050` during initial buildout (readable, matches
expectations). Years later, during a platform consolidation project, the SRGB is resized and moved to a
different base. The absolute-value prefix silently drops out of the IGP advertisement — no explicit error
pointing at the root cause, just a prefix that stops having SR reachability, discovered only when an SR-TE
path or TI-LFA computation involving that node fails.

**6. Design insight**
This is a textbook "convenience feature vs. long-term maintainability" trade-off — an architect should
mandate index-based prefix-SID configuration as an operational standard, precisely because it decouples
prefix identity from the current SRGB value, which should be treated as an implementation detail that can
change over the network's lifetime (platform migrations, capacity growth, mergers).

**7. Interview-ready answer**
"Prefix-SIDs can be configured as an index or an absolute label, but absolute values are silently
converted to an index for advertisement — and if the SRGB is later moved so the absolute value falls
outside the new range, the SID quietly stops being advertised with no direct error. Best practice is
always index-based configuration so SID advertisement survives SRGB changes."

---

## 3. ISIS Prefix-SID Sub-TLV Flags (Authoritative Definitions + Hidden Behavior)

**1. Definition**
Each Prefix-SID is carried in a Prefix-SID sub-TLV attached to the Extended IP Reachability TLV (Type
135), containing a flags octet whose bits govern per-hop forwarding behavior for that specific SID.

**2. Why it exists**
SR has no signaling protocol, so all special per-hop behaviors (should PHP happen? should this SID be
trusted as identifying a single node? was this SID re-advertised from elsewhere?) must be encoded
declaratively as flags inside the one piece of flooded state every node already receives.

**3. How it works — flag-by-flag**
- **R = Re-advertisement.** Set only when the prefix has been propagated from another IGP level (e.g., an
  L1/L2 router leaking an L1 prefix into L2, or vice versa).
- **N = Node SID.** Set when the prefix genuinely represents the originating node itself — this is the
  flag TI-LFA relies on to safely chain Adjacency-SIDs onto this Prefix-SID.
- **P = PHP-Off.** Set when the router is explicitly requesting that directly-connected neighbors not
  perform PHP.
- **E = Explicit-Null.** Set when the router is requesting Exp-Null instead of a plain pop. P must also be
  set whenever E is set — Explicit-Null is meaningless without also disabling PHP.
- **V = Value.** Set only if the SID field carries a raw label value instead of an index. Should never be
  seen for a genuine Prefix-SID in normal deployments.
- **L = Local.** Set if the SID has only local significance. Should never appear on a genuine Prefix-SID,
  because a Prefix-SID is by definition globally significant.
- Protocol placement: the Prefix-SID sub-TLV rides inside the Extended IP Reachability TLV (135) — i.e.,
  it's attached directly to the same TLV that already advertises the prefix's reachability/metric.

**4. Real-world use case**
Operators decode these exact flags daily when troubleshooting SR-TE path failures or unexpected PHP
behavior using `show isis database detail` — recognizing "R+P set but E clear" instantly tells an
experienced engineer "this is an inter-level/inter-area re-advertised prefix" without needing to look at
topology diagrams (see Concept #14).

**5. Failure scenario**
If a platform bug or misconfiguration sets E without P (violating the documented dependency), receiving
neighbors have undefined/inconsistent behavior — the specification assumes E is never meaningful without
P. This is why the two flags are separate bits: it lets the protocol distinguish "don't pop, but I don't
need Exp-Null" (No-PHP alone — the inter-area case) from "don't pop, and additionally give me Exp-Null"
(both flags — the true QoS-preservation case).

**6. Design insight**
The flags form a genuine two-axis decision space: PHP behavior (P) is orthogonal to null-label behavior
(E), and the protocol correctly refuses to conflate them, because you frequently need No-PHP without
wanting Explicit-Null (an ABR/L1L2 router legitimately isn't the final destination, but there's no QoS
requirement forcing Exp-Null there). A CCDE-level architect should internalize this: "No-PHP" and
"Explicit-Null" answer two genuinely different questions.

**7. Interview-ready answer**
"The Prefix-SID sub-TLV flags octet has R (re-advertised across levels), N (true node SID), P (No-PHP), E
(Explicit-Null, requires P), V and L (value/local — should never appear on a genuine Prefix-SID). P and E
are separate bits because No-PHP and Explicit-Null answer different questions — you frequently need
No-PHP without Explicit-Null, such as whenever an ABR or L1/L2 router re-advertises a prefix it doesn't
own."

---

## 4. ISIS Router Capabilities TLV (242) — RID Selection, SR-Capability, SR-Algorithm, Max SID Depth

**1. Definition**
The Router Capabilities TLV (Type 242) is the general-purpose, extensible ISIS TLV that carries
node-level (not per-prefix) Segment Routing state: the node's Router ID, its SR-Capability sub-TLV
(SRGB), its SR-Algorithm sub-TLV, and its Node Maximum SID Depth sub-TLV.

**2. Why it exists**
Some SR information genuinely describes the node, not any specific prefix — how big is its SRGB, what
algorithms does it support, how many labels can its hardware push onto a stack. This needs a single,
node-scoped advertisement, so the IGP reuses the pre-existing Router Capabilities TLV framework
(originally built for other purposes like TE) as the carrier.

**3. How it works**
- **Router ID selection order** (a genuinely hidden, easy-to-miss detail): the RID advertised in this TLV
  is chosen by priority:
  1. MPLS-TE Router ID — only takes effect if MPLS-TE is explicitly activated both for the IGP process and
     globally on the router.
  2. ISIS Router ID — achieves a similar effect without needing to enable the full MPLS-TE process, but
     requires that a Prefix-SID be configured on the loopback used as the RID.
  3. Lowest-numbered loopback (e.g., Lo0 preferred over Lo1) if neither of the above is configured.
  4. Lowest-numbered physical interface as the final fallback.
- **SR-Capability sub-TLV**: carries the SRGB base + range, plus two flags: I (MPLS IPv4 support) and V
  (MPLS IPv6 support).
- **SR-Algorithm sub-TLV**: lists which SPF algorithms (0 = SPF, 1 = strict SPF, etc.) this node supports.
- **Node Max SID Depth (MSD) sub-TLV**: advertises the maximum number of labels this node's forwarding
  hardware can actually push/process on a single packet (e.g., "this box supports MSD of 10"). This lets
  other nodes — specifically a PCE or headend computing an SR-TE explicit path — know not to build a label
  stack longer than what a transit node along the path can actually forward.
- **Top-level TLV flags**: S (Scope — set when the TLV should flood domain-wide; not typically used, so
  generally not leaked across level boundaries) and D (Down — set when the TLV is leaked from L2 down into
  L1, preventing it from being re-leaked back up from L1 to L2 and looping).

**4. Real-world use case**
The MSD sub-TLV is critical in any SR-TE/PCE-driven network with long explicit paths (many hops, or SR-TE
policies stacking Binding-SIDs across multiple domains) — a controller building a segment list must
respect the lowest MSD value along the intended path, or risk building an unusable label stack.

**5. Failure scenario**
If RID-selection precedence isn't understood, an operator might configure MPLS-TE Router ID on one router
and rely on ISIS RID on another, ending up with inconsistent RID sourcing across the domain. Also, if a
headend or PCE ignores MSD and builds a segment list exceeding a transit node's advertised MSD, that node
either drops the packet or attempts partial processing, causing an intermittent, hard-to-diagnose SR-TE
policy failure.

**6. Design insight**
MSD is one of the more overlooked SR building blocks in real designs — as SR-TE and multi-domain designs
grow label-stack depth, MSD becomes a genuine scaling constraint an architect must actively track across
the hardware inventory, not just a theoretical capability field.

**7. Interview-ready answer**
"The Router Capabilities TLV (242) carries node-scoped SR state: SRGB via the SR-Capability sub-TLV,
supported SPF algorithms via SR-Algorithm, and critically, Max SID Depth (MSD) — the deepest label stack
that node's hardware can actually forward. Any PCE or SR-TE headend must respect the minimum MSD along a
path, or it can build a segment list a transit router can't process."

---

## 5. SR Algorithms — Algorithm 0 (SPF) vs. Algorithm 1 (Strict SPF)

**1. Definition**
Every Prefix-SID is associated with an algorithm value: Algorithm 0 means "follow the IGP shortest path,
but permit any node along the path to override this via local policy," while Algorithm 1 ("strict SPF")
means "the path must always follow the literal IGP-computed shortest path, with no local policy allowed to
divert it." A single prefix can have multiple Prefix-SIDs, one per algorithm.

**2. Why it exists**
Algo 0 is the default/normal mode and deliberately allows transit routers flexibility — e.g., a node with
a local RSVP-TE tunnel using autoroute-announce toward a given destination is allowed to divert Algo-0 SR
traffic into that tunnel. But this flexibility is sometimes exactly the problem — an operator may need a
genuine guarantee that traffic follows the literal IGP-computed path with zero possibility of local
diversion. Algorithm 1 exists specifically to provide that hard guarantee.

**3. How it works (example)**
- R7's Lo1 by default advertises one Prefix-SID: index 7, algo 0.
- Add a second Prefix-SID for the same prefix using algo 1:
  ```
  router isis 1
   interface Loopback1
    address-family ipv4 unicast
     prefix-sid strict-spf index 77
  ```
- Now R7's loopback (7.7.7.1/32) has two Prefix-SIDs simultaneously advertised: index 7/algo 0, and index
  77/algo 1.
- Every router in the domain computes two separate labels: 16007 (algo 0, subject to local policy override
  at any transit node) and 16077 (algo 1, guaranteed strict IGP path).
- A normal, unmodified traceroute/RIB entry uses the algo-0 label (16007) by default. To force the strict
  path, you must explicitly push the algo-1 label (16077) yourself.

**4. Real-world use case**
Foundational to Flex-Algo in production networks — operators define custom algorithms (128-255)
representing constrained topologies, and every node advertises a Prefix-SID per Flex-Algo. Algo 1
specifically is used whenever a design needs a deterministic guarantee that no local policy can hijack
traffic that's supposed to follow the literal shortest path.

**5. Failure scenario**
See Concept #13 in full — an operator assumes Algo 0 traffic will "just follow the IGP path" and is
surprised when a transit node's local RSVP-TE tunnel silently reroutes it, because Algo 0's entire design
intent is to permit exactly that kind of local override.

**6. Design insight**
An architect should treat Algo 0 vs. Algo 1 as a genuine design lever: any prefix that must have
deterministic, policy-immune reachability should have a dedicated Algo-1 Prefix-SID advertised in addition
to its normal Algo-0 SID — doubling the SID/label consumption for that prefix, a real (if small) scaling
cost that should be budgeted into SRGB sizing when this pattern is used widely.

**7. Interview-ready answer**
"Algo 0 is the default — it follows the IGP shortest path but allows local policy (like an RSVP-TE tunnel
with autoroute-announce) to override it at any transit hop. Algo 1, strict SPF, guarantees the literal IGP
path with zero possibility of local diversion. A prefix can advertise both simultaneously as two separate
Prefix-SIDs."

---

## 6. Adjacency-SID Fundamentals — Dynamic Allocation, FRR/Non-FRR Pair

**1. Definition**
An Adjacency-SID (Adj-SID) is a locally-significant label representing "pop this label and forward out
this specific link/adjacency." By default, IOS-XR allocates two dynamic Adj-SID labels per IGP adjacency —
one FRR-eligible, one not — pulled from the dynamic label range (24000+).

**2. Why it exists**
Prefix-SIDs get you "reach this node via the best path," but SR-TE explicit paths sometimes need to force
traffic over one specific link regardless of IGP cost. Adj-SIDs provide that link-level steering
primitive. The dual FRR/non-FRR allocation exists because TI-LFA fast-reroute needs a distinct label value
to represent "this adjacency, protected by a pre-computed backup path" versus "this adjacency, no backup
computed" — the label itself is the signal for which forwarding behavior to apply.

**3. How it works**
- On adjacency establishment, the router reserves both a non-FRR Adj-SID and an FRR-eligible Adj-SID from
  its dynamic label pool.
- By default (TI-LFA not enabled), only the non-FRR Adj-SID is advertised into the IGP.
- The moment TI-LFA is enabled on that interface (`fast-reroute per-prefix` + `fast-reroute per-prefix
  ti-lfa`), the router begins advertising the FRR-eligible Adj-SID as well.
- Example: R5-R7 adjacency. Before TI-LFA: only non-FRR Adj-SID advertised. After enabling TI-LFA: both
  non-FRR and FRR-eligible Adj-SIDs are advertised.

**4. Real-world use case**
SR-TE policies and PCE-computed explicit paths routinely mix Prefix-SIDs with specific Adj-SIDs at points
where the path must deviate from IGP shortest-path.

**5. Failure scenario**
If TI-LFA is never enabled on an interface but an SR-TE controller assumes FRR protection is available for
every Adj-SID in its path database, it may build a segment list depending on protection that doesn't
actually exist — leading to a false sense of resiliency.

**6. Design insight**
Because Adj-SIDs are dynamically allocated by default, their exact label values are not predictable/stable
across reboots or SR re-enablement beyond a certain time window — any design that needs a guaranteed,
stable Adj-SID value must use static SRLB-based allocation rather than relying on the dynamic default.

**7. Interview-ready answer**
"Every IGP adjacency gets two dynamically-allocated Adj-SIDs from the dynamic label range: a non-FRR one,
always advertised, and an FRR-eligible one, only advertised once TI-LFA is enabled on that interface —
because the label itself signals whether that forwarding entry is backed by a pre-computed local repair
path."

---

## 7. Adjacency-SID Flags, Persistence, and Static Allocation via SRLB

**1. Definition**
Adjacency-SIDs carry their own flags octet (F, B, V, L, S, P) distinct from the Prefix-SID flags, and can
optionally be statically pinned to a specific value drawn from the SRLB (15000-15999 by default) instead
of relying on dynamic allocation.

**2. Why it exists**
Dynamic Adj-SID allocation is convenient but inherently unstable across time — some designs (external
tooling referencing specific labels, deterministic lab/documentation environments, forcing FRR-eligible
advertisement without needing TI-LFA) require a predictable, persistent value. Static SRLB-based
configuration solves this.

**3. How it works — flags**
- **F = Address-family flag.** When set, indicates IPv6; when clear, IPv4.
- **B = Backup (FRR-protected).** Set on the FRR-eligible Adj-SID once TI-LFA is active.
- **V = Value.** Always set for Adj-SIDs — the label is always the explicit value, never an index.
- **L = Local.** Always set — always locally significant.
- **S = Set.** Indicates the Adj-SID represents a set of multiple adjacencies — not currently supported.
- **P = Persistent.** Set when the value is guaranteed stable (statically configured). By default, an
  Adj-SID's local retention is only 30 minutes: if SR is disabled/re-enabled within that window, the same
  value is likely retained, but after 30 minutes (or a reboot) the dynamically-chosen value can change. A
  backup-protected Adj-SID whose adjacency goes down is held an extra 5 minutes beyond normal teardown
  (to let in-flight FRR-protected traffic use the protection) before being released back into the
  30-minute pool.
- Configuring static values requires `segment-routing` enabled globally first (this allocates the node's
  SRLB). Then:
  ```
  router isis 1
   interface GigabitEthernet0/0/0/3
    address-family ipv4 unicast
     adjacency-sid absolute 15003
     adjacency-sid absolute 15013 protected
  ```
  This creates a non-FRR Adj-SID of 15003 and an FRR-eligible one of 15013 — both persistent (P flag set).
  The dynamically-allocated default values are not replaced — they continue to be advertised in addition,
  meaning a single adjacency can have up to four Adj-SID values simultaneously advertised.
- Internal index semantics (unrelated to Prefix-SID indices): ISIS internally tracks Adj-SIDs using its
  own small index space: 0 = protected L1, 1 = protected L2, 2 = non-protected L1, 3 = non-protected L2.

**4. Real-world use case**
Static Adj-SIDs are used when an SR-TE controller or a documentation/lab environment needs to hard-code
specific, memorable label values in explicit segment lists rather than dynamically discovering current
Adj-SID values via telemetry.

**5. Failure scenario**
An engineer relies on a dynamically-allocated Adj-SID value observed once during initial deployment,
hard-codes it into an external system, and doesn't realize the value is only guaranteed for 30
minutes/until next reboot. After a maintenance-window reboot, the dynamic Adj-SID silently changes value,
breaking the external system's hard-coded reference.

**6. Design insight**
Anything a design treats as "must be stable across time" needs to be explicitly pinned (static/persistent),
never assumed stable from dynamic allocation.

**7. Interview-ready answer**
"Adj-SID flags include F (address-family), B (backup/FRR), V and L (always set, since Adj-SIDs are always
locally-significant explicit values), S (adjacency-set, unsupported), and P (persistent). Dynamic Adj-SIDs
are only guaranteed stable for 30 minutes or until reboot, so any design needing a guaranteed-stable value
must statically allocate it from the SRLB."

---

## 8. LAN Adjacency-SID — ISIS Pseudonode Model vs. OSPF DR/BDR Model

**1. Definition**
On a multi-access (LAN/broadcast) segment, IGPs normally represent the segment as a single "pseudonode" in
the topology (DIS for ISIS, DR for OSPF) rather than a full mesh of point-to-point adjacencies. SR needs a
mechanism to still let any router on the LAN steer traffic to any specific other router on that LAN — this
is the LAN-Adjacency-SID, and ISIS and OSPF implement it differently.

**2. Why it exists**
Without LAN-Adj-SIDs, the pseudonode abstraction would hide individual router identity on a shared segment
from SR's per-link steering model — an SR-TE path couldn't say "go out onto this LAN specifically toward
that one neighbor," only "go out onto this LAN" generically.

**3. How it works — ISIS**
- Each router on the LAN advertises one LAN-Adj-SID per other node on the LAN (e.g., R1, R2, R3 with R3
  DIS: R1 advertises a LAN-Adj-SID for R2, and a separate one for R3).
- Each LAN-Adj-SID sub-TLV is attached to the Extended IS Reachability TLV that identifies the adjacency
  with the pseudonode.
- Both FRR and non-FRR LAN-Adj-SIDs are allocated the same way as regular Adj-SIDs.
- **Critical exception**: the DIS itself does not advertise any SR information for the LAN.
- **Hidden limitation**: while FRR-eligible LAN-Adj-SIDs are allocated, they are not actually TI-LFA
  protected.

**How it works — OSPF**
- Every router advertises a normal (non-LAN) Adj-SID toward the DR, but a LAN-Adj-SID toward every
  BDR/DROTHER.
- Example: R3 is DR. R1 advertises a normal Adj-SID for its adjacency to R3, and a LAN-Adj-SID for R2.
- The DR ends up advertising a LAN-Adj-SID toward every other node on the segment.

**4. Real-world use case**
LAN-Adj-SIDs matter in any SR deployment with legacy Ethernet-based multi-access segments still present in
the topology.

**5. Failure scenario**
An operator assumes FRR-eligible LAN-Adj-SIDs on ISIS behave identically to regular point-to-point FRR
Adj-SIDs and designs a protection scheme relying on this — only to discover LAN-Adj-SIDs are not actually
TI-LFA-protected, leaving that segment without expected fast-reroute coverage.

**6. Design insight**
This is a strong argument for migrating legacy LAN/broadcast segments to point-to-point links wherever
TI-LFA protection is a hard requirement. Modern SR-friendly designs (especially DC Clos fabrics)
deliberately avoid multi-access segments in the underlay for exactly this reason.

**7. Interview-ready answer**
"On a LAN segment, ISIS decomposes the shared pseudonode adjacency into one LAN-Adj-SID per neighbor (with
the DIS itself advertising none), while OSPF advertises a normal Adj-SID toward the DR but a LAN-Adj-SID
toward every BDR/DROTHER. In ISIS, LAN-Adj-SIDs are FRR-eligible but not actually TI-LFA-protected, which
is why TI-LFA-dependent designs generally migrate away from multi-access segments entirely."

---

## 9. OSPF SR Control Plane — Opaque LSA Types (1, 4, 7, 8)

**1. Definition**
OSPF carries all Segment Routing state using the Opaque LSA mechanism — specifically Opaque-Area LSA
Types 1, 4, 7, and 8, each carrying a different slice of SR information.

**2. Why it exists**
OSPF reuses the pre-existing Opaque LSA extensibility mechanism (originally built to carry MPLS-TE
link-state information), keeping SR's information distribution riding entirely on OSPF's already-reliable
flooding. The type separation exists because SR's designers deliberately wanted to decouple SR from
classic MPLS-TE — SR's own information is kept in its own distinct LSA types (4, 7, 8) rather than
overloading the original TE LSA (type 1).

**3. How it works**
LSA ID format: X.Y.Y.Y, where X = opaque type, Y = an instance-specific ID.
- **Type 1 (TE Information)**: automatically generated the moment SR is enabled — a notable difference
  from ISIS, where you must manually configure a TE Router ID and separately activate the MPLS-TE process.
  LSID = 0 for router-level; LSID = ifIndex for per-link.
- **Type 4 (Router Information)**: advertises SR capabilities — algorithms, SRGB range, SRLB range, max
  SID depth, hostname. LSID = 0. Hidden detail: like ISIS, the SRLB default range is not advertised unless
  `segment-routing` is enabled globally.
- **Type 7 (Extended Prefix)**: carries the actual Prefix-SID for a specific prefix. Only prefixes with a
  Prefix-SID get a Type 7 LSA.
- **Type 8 (Extended Link)**: carries the Adj-SID(s) for a specific link. LSID = ifIndex.

**4. Real-world use case**
Understanding this LSA-type separation is essential for OSPF-based SR troubleshooting — a missing Type 7
for a specific prefix immediately tells you that prefix has no Prefix-SID configured/advertised.

**5. Failure scenario**
If an operator forgets to run global `segment-routing` (only configuring `segment-routing mpls` under the
OSPF process), the SRLB never gets advertised in the Type 4 LSA — any subsequent attempt to statically
allocate an Adj-SID from the SRLB fails.

**6. Design insight**
The Type-1-vs-rest separation illustrates protocol layering discipline — you can run pure SR with zero
RSVP-TE/classic-TE anywhere in the design, even though they reuse the same Opaque LSA transport mechanism.

**7. Interview-ready answer**
"OSPF carries SR state in Opaque-Area LSAs: Type 1 (TE info, auto-generated with SR, unlike ISIS), Type 4
(router-level SR capabilities), Type 7 (per-prefix Prefix-SID), and Type 8 (per-link Adj-SID). These are
deliberately separate LSA types from classic MPLS-TE information, so SR can run fully independent of
RSVP-TE."

---

## 10. OSPF vs. ISIS — Structural Differences in SR Implementation

**1. Definition**
While SR for OSPF and ISIS is conceptually nearly identical, several concrete differences exist:
`sr-prefer` syntax, TI-LFA enablement location, tiebreaker preference direction/inheritance, and
IPv6/SRv6 support.

**2. Why it exists**
Both protocols converged on the same SR architecture, but each protocol's pre-existing configuration
idioms and internal data models shaped how SR-specific knobs were bolted on.

**3. How it works — the differences**
- **sr-prefer**: separate top-level command in OSPF (`segment-routing sr-prefer`); a keyword on the SR
  enabling command in ISIS (`segment-routing mpls sr-prefer`).
- **TI-LFA enablement location**: OSPF allows TI-LFA at any level (process, area, interface) with normal
  inheritance. ISIS only allows TI-LFA configuration at the interface level.
- **TI-LFA tiebreaker direction and inheritance**: OSPF prefers a higher tiebreaker index, and TI-LFA
  tiebreakers are directly comparable/shared with regular LFA tiebreakers; OSPF also supports lc-disjoint
  and lowest-backup-metric. ISIS prefers a lower tiebreaker index (opposite convention), and TI-LFA
  tiebreakers are a completely separate rule set from LFA tiebreakers — config under the ISIS
  address-family for TI-LFA tiebreakers is not inherited down to the interface. ISIS TI-LFA supports only
  node-protecting and SRLG-disjoint protection types.
- **IPv6/SRv6 support**: ISIS supports IPv6 for both the SR-MPLS and SRv6 dataplanes. SR is not currently
  supported on OSPFv3.

**4. Real-world use case**
Directly impacts IGP protocol selection during greenfield SP core design: if IPv6 SR is a requirement,
ISIS is the only viable IGP choice given current OSPFv3 limitations.

**5. Failure scenario**
An engineer familiar with ISIS's "lower index wins" convention who configures OSPF TI-LFA tiebreakers with
the same mental model gets inverted results, since OSPF prefers the higher index — a subtle cross-protocol
bug that's easy to make and hard to spot.

**6. Design insight**
An architect working across a multi-IGP environment must maintain an explicit documented cross-reference
of these divergences — "SR works basically the same on OSPF and ISIS" is true only conceptually, never at
the exact configuration/behavioral level.

**7. Interview-ready answer**
"OSPF and ISIS SR diverge in real ways: sr-prefer is separate in OSPF vs. a keyword in ISIS; OSPF allows
TI-LFA at any level with normal inheritance while ISIS only allows interface-level; OSPF prefers higher
tiebreaker index while ISIS prefers lower with completely separate TI-LFA vs. LFA tiebreaker rules; and
only ISIS currently supports IPv6 for both SR-MPLS and SRv6."

---

## 11. OSPF Prefix-SID Flags — Extended Prefix TLV (A, N) and SID Sub-TLV (NP, M, E, V, L)

**1. Definition**
OSPF's Prefix-SID information is split across two separate flag fields: the Extended Prefix TLV carries A
(Attached) and N (Node) flags, while the nested SID sub-TLV carries its own NP (No-PHP), M
(Mapping-server), E (Explicit-Null), V (Value), and L (Local) flags.

**2. Why it exists**
The Extended Prefix TLV flags describe the prefix's relationship to the advertising router, while the
nested SID sub-TLV flags describe forwarding behavior for the specific SID value — mirroring ISIS's single
flags octet, but split across two nesting levels.

**3. How it works**
- **Extended Prefix TLV flags**: A = attached (directly attached to the ABR advertising it, though the
  prefix is inter-area); N = Node SID. Observed hex: 0x40 = only N set; 0x80 = only A set; 0xC0 = both.
  Note: OSPF tooling doesn't always translate hex to flag names automatically, unlike ISIS.
- **SID sub-TLV flags**: NP = No-PHP (equivalent to ISIS's P-flag); M = Mapping-Server (no direct ISIS
  equivalent — set when the Prefix-SID was advertised by a Segment Routing Mapping Server rather than by
  the owning node, relevant for brownfield/non-SR-capable-router scenarios); E = Explicit-Null (with NP,
  gives Exp-Null-without-PHP); V/L = same as ISIS, should never be set for a genuine Prefix-SID. Observed
  hex 0x40 = only NP set.

**4. Real-world use case**
The A-flag becomes operationally relevant when troubleshooting inter-area SR reachability — seeing A=1
immediately tells you the ABR is directly attached to this prefix.

**5. Failure scenario**
Because OSPF's flag decoding tooling doesn't always translate hex automatically, an engineer under time
pressure might misread a flag value rather than correctly checking the bit position.

**6. Design insight**
The M-flag/SRMS interaction is architecturally significant for brownfield migrations: not every router in
an OSPF domain needs to be individually SR-capable to get valid Prefix-SIDs — a Mapping Server can
centrally assign SIDs on behalf of legacy nodes, enabling incremental SR rollout.

**7. Interview-ready answer**
"OSPF splits Prefix-SID flags across two levels: the Extended Prefix TLV has A (attached-to-ABR) and N
(node SID), while the nested SID sub-TLV has NP (No-PHP), M (mapping-server — assigned centrally via
SRMS), E (explicit-null), V, and L. The M flag has no ISIS equivalent and specifically enables brownfield
SR rollout without requiring every router to be individually SR-capable."

---

## 12. OSPF Adjacency-SID Flags — Extended Link TLV (B, V, L, S)

**1. Definition**
OSPF's Adjacency-SID flags, carried in the Type 8 Extended Link Opaque LSA, use a B, V, L, S layout —
structurally similar to ISIS's Adj-SID flags, but V and L are always set in OSPF.

**2. Why it exists**
Since an OSPF Adj-SID is by definition always a locally-significant explicit label value, OSPF hard-wires
V and L to always-set rather than leaving them as theoretically-independent bits.

**3. How it works**
- B = Backup, set for FRR-protected (TI-LFA-eligible) Adj-SIDs.
- V = Value, always set.
- L = Local, always set.
- S = Set (multiple adjacencies), not currently supported.
- Practical consequence: the observed flag byte is always exactly 0xE0 (protected) or 0x60 (unprotected).
  Enabling TI-LFA flips the advertised flag from 0x60 to 0xE0.

**4. Real-world use case**
Because the flag space collapses to a single binary signal in practice, OSPF Adj-SID flag parsing in
automation/telemetry pipelines is simpler to implement correctly than the equivalent ISIS parsing logic.

**5. Failure scenario**
An engineer writing custom telemetry who doesn't realize V/L are always set might build unnecessarily
complex parsing logic checking all four bits independently.

**6. Design insight**
OSPF's simpler flag space doesn't mean weaker functionality, just a tighter implementation — an architect
shouldn't try to force symmetrical parsing logic across both protocols where the underlying bit semantics
genuinely differ.

**7. Interview-ready answer**
"OSPF Adj-SID flags are B (backup/FRR), V (value), L (local), and S (adjacency-set, unsupported) — but V
and L are always set by construction. In practice you only ever see 0xE0 (protected) or 0x60
(unprotected), simplifying OSPF Adj-SID auditing versus ISIS."

---

## 13. SR-TE Algorithm Interaction with RSVP-TE (Autoroute-Announce Hijacking Algo-0 Traffic)

**1. Definition**
An RSVP-TE tunnel configured with autoroute-announce toward a destination will silently absorb Algo-0 SR
traffic destined for that same prefix, because Algo 0's design explicitly permits exactly this kind of
local-policy override — and Algorithm 1 (strict SPF) is the architecturally correct fix.

**2. Why it exists (the problem)**
In a network transitioning from RSVP-TE to SR, a legacy RSVP-TE tunnel might still exist on a transit
node with autoroute-announce (making the tunnel appear as the preferred next-hop for any destination
reachable via/beyond it). If that node also carries Algo-0 SR traffic toward the tunnel's destination, it
has no way to distinguish traffic that should respect local policy from traffic that must ignore it —
unless the destination explicitly advertises a strict-SPF (Algo 1) Prefix-SID as an alternative.

**3. How it works (full worked example)**
- Topology: R1 -> R3 -> ... -> R7. R3 has a pre-existing RSVP-TE tunnel to R7's loopback with
  autoroute-announce.
- R1 traceroutes to R7's loopback; the path unexpectedly transits R3's TE tunnel.
- Root cause: R7's Lo1 advertises its Prefix-SID with the default Algo 0. R3's LFIB: incoming label 16007
  -> pop -> forward out the TE tunnel interface.
- Fix (with the constraint "do not touch R3"): advertise a second Prefix-SID at the destination, R7:
  ```
  router isis 1
   interface Loopback1
    address-family ipv4 unicast
     prefix-sid strict-spf index 77
  ```
- R7 now advertises two Prefix-SIDs: index 7/algo 0 (still hijacked by R3's tunnel) and index 77/algo 1.
- On R3, label 16077 (Algo 1) cannot be diverted into the tunnel — R3's LFIB: swap 16077->16077 out the
  IGP-bestpath physical interface.
- Verification nuance: a normal, unmodified OAM traceroute to 7.7.7.1/32 still uses label 16007 (algo 0)
  by default. Pushing 16077 specifically follows the strict IGP path.

**4. Real-world use case**
Common in real SP cores mid-transition, where RSVP-TE tunnels for legacy services coexist with new
SR-based services on the same infrastructure.

**5. Failure scenario**
Without understanding this distinction, an operator troubleshooting an unexpected SR path might suspect
IGP metric misconfiguration or ECMP hashing anomalies — when the actual root cause is a completely
unrelated legacy RSVP-TE tunnel with autoroute-announce.

**6. Design insight**
Algo 0's flexibility is a double-edged design choice — useful, but creates non-obvious path behavior in
mixed RSVP-TE/SR environments. An architect running long-lived coexistence should proactively decide, per
critical destination, whether a parallel Algo-1 SID needs to be provisioned in advance. The fix living
entirely at the destination rather than the misbehaving transit node is an elegant SR capability RSVP-TE-
only designs don't have.

**7. Interview-ready answer**
"An RSVP-TE tunnel with autoroute-announce will silently capture Algo-0 SR traffic at any transit node,
because Algo 0 explicitly permits local-policy overrides. The fix doesn't require touching the transit
node — you advertise a second, Algo-1 Prefix-SID at the destination, giving the network a label that
guarantees the literal IGP path with zero possibility of local diversion."

---

## 14. Inter-Area SR Propagation — Automatic Behavior, and Why No-PHP Without Explicit-Null Matters

**1. Definition**
When a Prefix-SID crosses an IGP area/level boundary, the border router automatically re-originates the
Prefix-SID on behalf of the prefix's true owner — this "just works" by default once basic route
propagation is configured.

**2. Why it exists**
Inter-area/inter-level propagation is a foundational IGP scalability mechanism. SR needed its Prefix-SID
mechanism to transparently ride along with this pre-existing propagation behavior, or SR would be
fundamentally incompatible with hierarchically-designed networks.

**3. How it works — the canonical answer to "when do you see No-PHP without Explicit-Null?"**
- Topology: R1, R3 = L1-only. R5 = L1/L2, propagating L2<->L1.
- R5 advertises R1 and R3's prefixes into L2 as if they originated at R5 itself.
- R5 does not own these prefixes, so it must not let neighbors PHP toward it for this SID — R5 sets the
  No-PHP (P) flag. It does not set Explicit-Null (E), because there's no QoS-preservation requirement
  here — the No-PHP requirement is purely topological correctness.
- The R (Re-advertisement) flag is also set.
- Symmetric behavior applies for L2-into-L1 propagation.
- Result verified via mpls oam: a genuine end-to-end LSP forms between R1 and R7, entirely automatically —
  only normal IGP inter-level route-policy propagation was configured.

**4. Real-world use case**
Every large SP core with a hierarchical ISIS design relies on exactly this automatic behavior for SR to
function correctly end-to-end.

**5. Failure scenario**
An operator who incorrectly configures Explicit-Null on the ABR/L1L2 router for these re-advertised
prefixes introduces unnecessary swap-instead-of-pop operations for prefixes that never needed Exp-Null
preservation.

**6. Design insight**
No-PHP-without-Explicit-Null is the necessary and correct signature of "a router is re-advertising a
prefix it doesn't own, for purely topological reasons, with no QoS requirement involved." Its absence on
boundary routers is a red flag worth investigating.

**7. Interview-ready answer**
"Inter-area/inter-level SR propagation works automatically once you configure normal route propagation.
The ABR/L1L2 router sets No-PHP (it's re-advertising a prefix it doesn't own) and R (re-advertisement),
but deliberately does not set Explicit-Null, since there's no QoS-preservation need — this is exactly why
No-PHP and Explicit-Null are separate flags."

---

## 15. OSPF Inter-Area SR — Type-3 Route-Type, A-Flag Semantics, and ABR Loop-Prevention

**1. Definition**
OSPF's inter-area SR propagation follows the same automatic-by-default pattern as ISIS, expressed through
OSPF's own mechanisms: the Extended Prefix TLV carries a route-type field (Type 3 = inter-area), and the
same ABR-level loop-prevention rules that govern classic Type-3 summary LSAs automatically extend to
Prefix-SID propagation.

**2. Why it exists**
OSPF already has a mature, loop-safe mechanism for inter-area propagation (Type 3 Summary LSAs, with ABRs
never re-accepting a Type-3 LSA from a non-backbone area back into another non-backbone area). SR
piggybacks on this pre-existing, well-tested safety mechanism rather than inventing new logic.

**3. How it works**
- R3's ext-prefix (Type 7) LSA for a remote prefix has route-type = 3 (inter-area).
- Extended Prefix TLV flags 0x40: only N set, A clear — the prefix is not directly attached to the
  advertising ABR.
- SID sub-TLV flags 0x40: only NP set — R3 is not the true final destination, so neighbors must not PHP.
  E is not set — same reasoning as the ISIS case, purely topological.
- Critical loop-prevention: R3 and R4 (ABRs) do not re-inject the learned prefix SID back into area 0 —
  directly following OSPF's Type-3 rule that an ABR never re-originates a Type-3 LSA received from a
  non-backbone area back into another area.
- Result: end-to-end LSP confirmed via mpls oam, with zero SR-specific configuration beyond normal OSPF
  inter-area propagation.

**4. Real-world use case**
Any large enterprise/SP network using classic OSPF multi-area hierarchy benefits from this "SR loop-safety
for free" property.

**5. Failure scenario**
An engineer unfamiliar with this inheritance might spend unnecessary effort trying to manually prevent
SR-specific propagation loops — redundant, since the underlying OSPF Type-3 rule already structurally
prevents it.

**6. Design insight**
SR was deliberately designed to inherit, not duplicate, each IGP's existing loop-prevention and hierarchy
mechanisms.

**7. Interview-ready answer**
"OSPF inter-area SR propagation marks Prefix-SIDs with route-type 3, sets No-PHP (the ABR isn't the true
destination) without Explicit-Null, and inherits OSPF's existing Type-3 loop-prevention rule automatically
— meaning SR requires zero additional loop-prevention logic of its own in a multi-area design."

---

## 16. Inter-IGP SR Redistribution — RIB-Driven Index Derivation and the SRGB-Match Requirement

**1. Definition**
When a prefix is redistributed between two different IGP processes at an ASBR, the Prefix-SID is
automatically carried along — but only if both protocols share the same SRGB, and the ASBR's own loopback
is natively advertised into both protocols independently.

**2. Why it exists**
Multi-IGP domains are common in large networks that grew through mergers or deliberate segmentation.
Redistribution is the classic mechanism for stitching these together — SR needed this to "just work" for
the MPLS label plane too, without a separate manual SID-mapping process.

**3. How it works**
- R3 runs both ISIS and OSPF as the redistribution ASBR.
- Redistribution is RIB-driven: R3 pulls a route (including its resolved MPLS label) directly out of the
  RIB, converts the label back into an index, and re-advertises it as a fresh Prefix-SID in the other
  protocol.
- **Requirement #1 — matching SRGB**: if ISIS's SRGB differs from OSPF's SRGB on R3, the prefix is still
  redistributed at the IP layer, but the Prefix-SID is not included — silent loss of SR/MPLS continuity.
- **Requirement #2 — native loopback advertisement**: R3's own loopback must be natively advertised with a
  Prefix-SID into both ISIS and OSPF independently, even though it's the same physical box — because each
  IGP process independently drives its own RIB/label advertisement.
  ```
  router ospf 1
   area 0
    interface Loopback1
     prefix-sid index 3
  !
  router isis 1
   interface Loopback1
    address-family ipv4 unicast
     prefix-sid index 3
  ```
- Flag behavior mirrors inter-area propagation: No-PHP set, R flag set (ISIS side), plus the X (External)
  flag — unrelated to SR, just normal ISIS redistribution behavior.
- LSA-type detail: ISIS-into-OSPF redistribution generates a Type 5 External LSA plus an Opaque-AS (Type
  11, domain-wide) LSA carrying the SR information — one of the few scenarios producing a Type 11 rather
  than Type 10 opaque LSA.

**4. Real-world use case**
Any SP/enterprise network built through M&A or deliberate multi-domain segmentation, where a unified SR-
MPLS LSP must span an ISIS "half" and an OSPF "half" of the infrastructure.

**5. Failure scenario**
The common trap: IP reachability works fine post-redistribution (doesn't depend on SRGB matching or
native loopback advertisement), leading an operator to assume SR/label continuity is also fine — only
discovering during MPLS OAM testing (or a live outage) that specific prefixes have no label.

**6. Design insight**
Strong argument for mandating identical SRGB values across every IGP process in a multi-IGP design as a
hard functional requirement, not just an operational-simplicity best practice, the moment inter-IGP
redistribution with SR continuity is in play.

**7. Interview-ready answer**
"SR-aware redistribution is RIB-driven — the ASBR pulls the already-resolved label out of the RIB and
converts it back to an index for the other protocol, but only if both IGPs share the same SRGB; if they
don't, the prefix still redistributes at the IP layer but silently loses its label. You also need the
ASBR's own loopback natively advertised with a Prefix-SID in both protocols independently."

---

## 17. SR for BGP — BGP Prefix-SID Attribute as a Label-Allocation "Hint" over BGP-LU

**1. Definition**
"SR for BGP" is classic BGP Labeled-Unicast (BGP-LU) plus one additional optional-transitive path
attribute (the BGP Prefix-SID attribute, carrying a label-index value) that acts as a hint telling each
receiving router to allocate its local label from its SRGB instead of a random dynamic label.

**2. Why it exists**
BGP-LU already solves inter-domain labeled transport, but classic BGP-LU labels are as unpredictable as
LDP labels — defeating SR's deterministic-label goal if BGP were the sole outlier still doing dynamic
allocation. The BGP Prefix-SID attribute closes this gap.

**3. How it works — full worked comparison**

*Classic BGP-LU (baseline):* R2 allocates local label 3 (Implicit-Null) as the ultimate hop, advertises to
R3. R3 allocates its own dynamic label (e.g., 24003, possibly reused from LDP) and advertises downstream.
R5 allocates its own dynamic label. R8 allocates its own local label and installs a two-label CEF stack.
Every hop's label is unpredictable.

*SR for BGP:*
- Prerequisite: SRGB globally defined on each participating router (`segment-routing / global-block 16000
  23999`).
- Originating router injects its prefix with a route-policy setting label-index:
  ```
  route-policy SET_PREFIX_SID($SID)
   set label-index $SID
  end-policy
  !
  router bgp 1
   address-family ipv4 unicast
    network 2.2.2.1/32 route-policy SET_PREFIX_SID(2)
    allocate-label all
  ```
- R2 injects 2.2.2.1/32 with BGP-LU label 3 (Implicit-Null) plus the Prefix-SID attribute (label-index=2).
- R3 honors the hint, allocates 16002 instead of a random dynamic value, and advertises 16002 downstream.
- R5 programs 16002 as outgoing regardless of its own SR-awareness (classic BGP-LU semantics: whatever
  label is received is what's programmed), and separately allocates its own local 16002 since it also has
  an SRGB.
- R8 similarly allocates 16002.
- Result: a fully deterministic, SRGB-derived global label end-to-end, rather than unpredictable per-hop
  values.

**Interworking property**: classic BGP-LU and SR-for-BGP interwork transparently — a node always honors
whatever label it actually receives from a peer. PHP control works purely by which label value the
upstream node advertises (Implicit-Null = "PHP for me" vs. any real label = "don't PHP, use this label").

**4. Real-world use case**
Standard mechanism for Inter-AS Option C-style deterministic labeled transport — extending SR's
deterministic-label benefits across inter-domain boundaries.

**5. Failure scenario**
Mixing SR-enabled IGP with non-SR (classic) BGP-LU on the same box: if IGP SR has already installed
16002 in the LFIB, and BGP-LU (without the SR hint) tries to allocate its own independent dynamic label
for that same prefix, BGP's label cannot be installed — the LFIB already has a conflicting entry — breaking
the end-to-end LSP.

**6. Design insight**
"SR at the IGP layer" and "SR at the BGP layer" must be deployed consistently together on any router that
runs both, because they compete for the exact same LFIB label-installation slot for a given prefix.

**7. Interview-ready answer**
"SR for BGP is BGP-LU plus an optional-transitive Prefix-SID attribute carrying a label-index — every hop
with a globally-defined SRGB honors this as a hint to allocate SRGB-base+index instead of a dynamic
value, while still interworking with classic BGP-LU. The dangerous failure mode is running IGP-SR and
classic BGP-LU together on the same box — they compete for the same LFIB slot and BGP's allocation
attempt fails outright."

---

## 18. BGP-SR Operational Pitfalls — allocate-label all, eBGP Local-Peer Label Allocation, and PE/ASBR Ordering Bugs

**1. Definition**
Beyond the core SRGB-mismatch failure, production SR-for-BGP deployments carry additional subtle
operational gotchas: needing `allocate-label all` even on non-transit PEs, an eBGP-specific quirk around
directly-peered labeled-address-family neighbors, and observed label-allocation ordering bugs requiring a
full BGP process reset.

**2. Why it exists**
BGP-LU/SR's label-allocation logic has edge cases that only manifest under specific topological roles (PE
vs. ASBR, direct-peering vs. route-reflected) — real implementation-level behaviors that must be known
before they cause a production incident.

**3. How it works — each pitfall**
- **allocate-label all needed even on non-transit PEs**: if a PE's own loopback is advertised by a
  different device (e.g., an ASBR on the PE's behalf), the PE itself must still run `allocate-label all`
  under BGP — otherwise it doesn't install the remote PE's prefix into its own LFIB with a proper label,
  breaking recursive lookup for VPN routes.
- **eBGP direct-peering label quirk**: even when a valid SR label already exists for a remote prefix, if
  two PEs establish a direct eBGP session using a labeled address-family, the router always allocates its
  own local label for the directly-peered neighbor address with a "Pop" outgoing-label action — apparently
  to support Inter-AS Option B/C architectures. Fix: peer each PE with its local Route Reflector instead
  of directly with the remote PE, or apply `ebgp-multihop mpls` on the neighbor statement.
- **Observed label-allocation ordering bugs**: both PEs and ASBRs were observed showing a BGP IPv4/LU
  prefix with what looked like a correct, matching local-and-remote label — yet pings still failed,
  because a stray dynamic local label had also been separately allocated for the same prefix and the LFIB
  resolved to the wrong entry. Fix: fully remove and re-add the BGP process, not just clear the session.

**4. Real-world use case**
Any CCDE-level engineer deploying Inter-AS SR-MPLS transport for a real VPN service should expect to
encounter at least one of these gotchas during initial deployment or subsequent topology changes.

**5. Failure scenario**
A team validates Inter-AS SR-MPLS transport successfully using route-reflectors, then a different team
establishes a direct eBGP session between two PEs for testing — and is confused when the SR/BGP-LU label
infrastructure doesn't transparently extend, not realizing direct-peering triggers an entirely separate
local-Pop-label mechanism.

**6. Design insight**
BGP-LU/SR label allocation behavior is topology-role-sensitive in ways not obvious from the base protocol
specification — an architect should never assume SR-for-BGP "works" based on validating one topological
pattern and assume it generalizes to a different pattern without separately validating that specific case.

**7. Interview-ready answer**
"SR-for-BGP has real operational gotchas beyond the core mechanism: PEs need allocate-label all even for
routes they don't originate, since that enables VPN-route recursive lookup through a remote PE's /32;
direct eBGP peering between two PEs on a labeled address-family always triggers local Pop-label
allocation regardless of any existing SR label, fixed via Route Reflector peering or ebgp-multihop mpls;
and stale duplicate dynamic-label bugs have been observed requiring a full BGP process removal-and-readd
to clear."

---

## Quick-Reference Summary Table

| # | Concept | Key Mechanism | Hidden Detail / Risk |
|---|---|---|---|
| 1 | SR vs LDP/RSVP-TE | IGP-flooded state, no soft state | Coexistence complexity if not fully migrated |
| 2 | Index vs Absolute | `SRGB base + index` | Absolute value silently breaks on SRGB move |
| 3 | ISIS Prefix-SID flags | R/N/P/E/V/L in sub-TLV (Type 3 in TLV 135) | P required whenever E is set |
| 4 | Router Capabilities TLV 242 | RID priority, SR-Cap, SR-Algo, MSD | MSD limits SR-TE label-stack depth |
| 5 | Algo 0 vs Algo 1 | SPF vs strict-SPF | Algo 0 allows local-policy hijack (RSVP-TE) |
| 6 | Adj-SID basics | Dynamic FRR/non-FRR pair | FRR only advertised once TI-LFA enabled |
| 7 | Adj-SID flags/SRLB | F/B/V/L/S/P, static via SRLB | Dynamic values only stable 30 min/reboot |
| 8 | LAN Adj-SID | ISIS per-neighbor vs OSPF DR/BDR model | ISIS LAN-Adj-SID not TI-LFA protected |
| 9 | OSPF Opaque LSAs | Types 1/4/7/8 | SRLB needs global `segment-routing` |
| 10 | OSPF vs ISIS diffs | sr-prefer, TI-LFA scope, tiebreakers | Tiebreaker direction inverted between protocols |
| 11 | OSPF Prefix-SID flags | A/N + NP/M/E/V/L | M flag enables SRMS brownfield rollout |
| 12 | OSPF Adj-SID flags | B/V/L/S | V/L always set -> only 0xE0/0x60 observed |
| 13 | SR/RSVP-TE interaction | Algo 1 bypasses autoroute-announce | Fix lives at destination, not transit node |
| 14 | Inter-area SR (ISIS) | Auto-propagation, No-PHP w/o ExpNull | R+P set is the "not the owner" signature |
| 15 | Inter-area SR (OSPF) | Route-type 3, ABR Type-3 loop rule | SR inherits OSPF's existing loop prevention |
| 16 | Inter-IGP redistribution | RIB-driven label->index | Requires matching SRGB + native loopback SID |
| 17 | SR for BGP | Prefix-SID attribute = label-index hint | Mixing SR-IGP + classic BGP-LU breaks LFIB |
| 18 | BGP-SR pitfalls | allocate-label all, RR vs direct-peer | Direct eBGP peering forces local Pop label |

