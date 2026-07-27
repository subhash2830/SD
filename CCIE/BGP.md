# CCIE/CCDE — BGP (General Topic)
*Simple explanations, CCDE-level design depth, interview answers, CLI, and a concept memory map — covering all 31 labs in the standalone "BGP" section.*

---

## PART A — Path Manipulation & Migration Fundamentals

### 1. Conditional Advertisement
**What:** Advertises a route only when a *different* condition-prefix is present (or absent) in the BGP table — `advertise-map` paired with `exist-map`/`non-exist-map`.
**Why it matters (CCDE lens):** This is a general-purpose "don't falsely promise reachability" mechanism — the exact same design principle seen in ISIS's Conditional ATT Bit and EVPN's DF/core-isolation logic, just expressed at the BGP policy layer. The canonical use case: an AS with two upstreams should stop advertising its routes to Upstream A if it has already lost its *own* better path (e.g., to a downstream customer or preferred transit), preventing Upstream A from blackholing traffic through a now-suboptimal or nonexistent path.
**Real-world example:** A multihomed enterprise stops advertising a specific prefix to its backup ISP unless its primary ISP link is actually down — avoiding asymmetric/suboptimal inbound routing during normal operation while still providing a real backup path during a genuine outage.

### 2. Aggregation and Deaggregation
**What:** `aggregate-address` summarizes multiple specific routes into one covering prefix; deaggregation is the deliberate reverse — advertising more-specific routes alongside or instead of a summary, typically for traffic engineering (a more-specific route always wins path selection over a less-specific one).
**Why it matters (CCDE lens):** Aggregation is a scale lever (smaller tables, faster convergence, hiding internal instability from external peers via `summary-only`), but it's a real trade-off: summarizing away specifics also summarizes away granular traffic-engineering ability and can create routing black holes if the aggregate is advertised while a component route becomes unreachable (the classic "aggregate exists but a specific /24 inside it is actually down" trap) — the `as-set` keyword preserves AS_PATH information from all contributors to avoid loop-detection issues, at the cost of instability propagating through the AS_SET whenever a contributor changes.
**Real-world example:** An SP aggregates a /16 for external advertisement (reducing global table impact) but deliberately keeps deaggregated /24s internally for finer-grained internal traffic engineering — a common two-tier design (coarse externally, granular internally).

### 3. Local AS
**What:** Lets a router peer using a *different* ASN than its real/global ASN — critical for ASN migrations where you can't flag-day-cutover every peer simultaneously. Three keywords control exact behavior: `dual-as` (accept peering on either ASN), `no-prepend` (don't add the local-real-ASN hop to the AS_PATH on received routes), `replace-as` (send routes as if genuinely originated from the local-as, hiding the real ASN from that peer).
**Why it matters (CCDE lens) — the clearest "safe live migration" pattern in the whole BGP topic:** this is the direct, real-world mechanism for a **non-disruptive ASN renumbering** project — the same recurring "how do you change something live without an outage" theme from ISIS metric-style transition, IPv6 ST→MT migration, and LDP password rollover. `dual-as` specifically lets you migrate one peer at a time onto the new ASN with zero coordinated cutover, since the peer can reach you on either identity during the transition window. A subtlety worth internalizing: there's no way to prepend *only* the global ASN — if the peer uses `bgp enforce-first-as` (on by default on many platforms), this constrains which keyword combinations are even viable.
**Real-world example:** An enterprise merging two previously-independent ASNs into one uses Local AS on the router being migrated so its BGP peer never needs simultaneous reconfiguration — the peer session survives the ASN change transparently, and the peer relationship can later be re-pointed to the real new ASN on the peer's own schedule.

### 4. Non-Optimal eBGP Routing
**What:** A catch-all lab category for scenarios where BGP's default best-path algorithm produces a technically-correct-but-suboptimal outbound or inbound path — e.g., traffic exiting via a farther/more-expensive eBGP peer because AS_PATH length (or another earlier tie-breaker) wins over a shorter, cheaper, or lower-latency alternative that BGP has no way to "know" is actually better.
**Why it matters (CCDE lens):** This is the practical, hands-on version of the theoretical "BGP has no native cost/latency awareness" limitation — BGP's best-path algorithm is a deterministic, policy-driven process, not a "shortest/cheapest path" algorithm like an IGP. Fixing non-optimal eBGP routing always means deliberately intervening at a specific step in the best-path algorithm (Local Preference, AS_PATH prepending, MED, or a lower-level tie-breaker) — a CCDE candidate should be able to identify *which* best-path step is responsible for a given non-optimal outcome and pick the least-disruptive lever to correct it, rather than reaching for a blunt instrument.

---

## PART B — QoS & Enterprise Multihoming Design

### 5. BGP QoS Policy Propagation (QPPB)
**What:** Uses BGP attributes (typically communities) received on a prefix to drive **local QoS classification/marking decisions** — QPPB (QoS Policy Propagation via BGP) lets an edge router classify traffic based on the BGP community attached to the destination/source prefix, rather than requiring a locally-maintained ACL/prefix-list for QoS classification.
**Why it matters (CCDE lens):** This decouples QoS policy *distribution* from QoS policy *enforcement* — a central point (e.g., a route server or the originating AS) tags prefixes with a community once, and every downstream router enforcing QoS just reads that community locally, instead of every enforcement point needing its own independently-maintained prefix-list that must be kept in sync as prefixes change. This is the same "let BGP carry the policy metadata, don't hand-maintain scattered local config" philosophy as BGP AD in the VPWS/VPLS notes, applied to QoS marking instead of service discovery.
**Real-world example:** A provider offers a premium-tier service where customer prefixes are tagged with a specific BGP community at the point of origination; every PE across the network then applies matching QoS treatment purely by reading that community via QPPB — adding or removing a customer from the premium tier is a one-place community change, not an N-router ACL update.

### 6. Multihomed Enterprise Challenge (+ XRv variant)
**What:** A comprehensive scenario lab combining multiple multihoming techniques (from earlier sections and the BGP Multi-Homing series) into one realistic enterprise-multihoming design exercise — typically involving asymmetric inbound/outbound path control across two or more upstream ISPs.
**Why it matters (CCDE lens):** Real enterprise multihoming design is rarely a single clean technique in isolation — it's usually a *combination* (AS_PATH prepending for inbound influence + Local Preference for outbound influence + conditional advertisement for backup-only paths + communities for provider-side policy triggers) applied together to meet a specific business requirement (primary/backup, cost optimization, redundancy without asymmetric routing surprises). This "challenge" format specifically tests whether a candidate can synthesize multiple individually-simple techniques into one coherent design, which is exactly the skill a CCDE interview is testing for.

### 7. Provider Communities (+ XRv variant)
**What:** Standardized, well-known community values a provider publishes for customers to use to influence how the provider treats their routes — e.g., communities meaning "prepend N times to this specific peer," "don't advertise to peer X," or "set local-preference to Y" — giving customers **limited, safe, policy-scoped control** over the provider's own routing decisions without needing direct config access to the provider's routers.
**Why it matters (CCDE lens):** This is the standard mechanism that makes customer-influenced traffic engineering possible at any real SP without opening a support ticket for every change — the provider defines the "menu" of allowed community actions in their own route-map/policy once, and customers self-service by simply tagging their own advertisements. Designing a good provider-community scheme is itself a real CCDE-relevant skill: too few options limits customer flexibility, too many creates unpredictable interactions and support burden.

---

## PART C — Load Balancing (Cross-Reference)

### 8. DMZ Link BW Lab1 & Lab2 (UCMP)
**What:** Same DMZ link-bandwidth / UCMP mechanism covered in the BGP Multi-Homing notes (Lab2) — proportional load-sharing across unequal-bandwidth eBGP paths via the `dmzlink-bw` extended community.
**Why it matters (CCDE lens):** Its inclusion here (as a standalone "BGP" topic lab, separate from the Multi-Homing series) reinforces that UCMP is a **general eBGP capability**, not something specific to the R4/R5/R6 multihoming scenario — worth applying anywhere unequal-bandwidth eBGP paths exist. See the BGP Multi-Homing notes for the full XE/XR unit-encoding interop gotcha (kilobytes vs. bytes-per-second) — that caveat applies identically here.

---

## PART D — Fast Convergence: PIC (Prefix-Independent Convergence)

### 9. PIC Edge in the Global Table
**What:** Pre-computes and pre-installs a backup path in the FIB for global-table (non-VPN) BGP routes, so failure of a primary eBGP/iBGP path triggers a local forwarding-table flip instead of a full BGP reconvergence cycle.
**Why it matters (CCDE lens):** This is the generalized, standards concept behind the BGP Multi-Homing series' Backup Path (Lab3) and Repair Path (Lab11) labs — same underlying idea (pre-stage the alternative before it's needed), applied here to the global routing table specifically. "Prefix-independent" is the key word: the fix scales to however many prefixes share the failed next-hop, converging all of them with a single FIB update instead of one recomputation per prefix.

### 10. PIC Edge Troubleshooting
**What:** Diagnosing why PIC Edge *isn't* actually providing the expected fast-failover benefit — commonly because a genuine backup path doesn't actually exist in the BGP table (PIC can only pre-stage a path that BGP has actually learned and considered as a viable alternate; it cannot invent a backup route where none exists).
**Why it matters (CCDE lens):** The most common real-world PIC failure mode is a **design** problem, not a **configuration** problem — if only one truly viable path exists (e.g., due to route-policy filtering, or a topology where all traffic genuinely funnels through one point), no amount of PIC tuning will produce fast failover, because there's nothing to fail over *to*. This reinforces a recurring CCDE lesson: fast-convergence *features* (PIC, BFD, TI-LFA, IP FRR) are all fundamentally dependent on the underlying *topology and path diversity* actually providing a real alternative — the feature accelerates using a backup that must already exist by design.

### 11. PIC Edge for VPNv4
**What:** Extends PIC Edge into the L3VPN/VPNv4 address family — directly connects to the BGP Multi-Homing series' Lab11 (Same RD + Add-Path + Repair Path), where the RD-design decision (same RD across redundant PEs) was shown to be a prerequisite for this kind of VPNv4 path diversity/fast-convergence to even be possible.
**Why it matters (CCDE lens):** Reinforces the cross-topic dependency chain worth being able to narrate in an interview: **RD design decision (BGP notes) → whether multipath/backup paths are even visible (BGP Multi-Homing Lab9-10) → whether PIC Edge for VPNv4 has anything to pre-stage (this lab)**. A CCDE candidate who can trace that chain end-to-end demonstrates real systems-level understanding, not just isolated feature knowledge.

---

## PART E — Cross-AS Metric Propagation: AIGP & Cost-Community

### 12. AIGP (Accumulated IGP Metric)
**What:** A BGP path attribute that carries the accumulated IGP cost along the path — normally used within a single AS (or a confederation) to let BGP best-path selection consider true end-to-end IGP distance, something plain BGP has no native visibility into once traffic crosses an ASBR.
**Why it matters (CCDE lens):** AIGP directly answers "why is my BGP path selection choosing a topologically-worse exit point" in networks with multiple exit points and meaningfully different internal IGP distances to each — without AIGP, BGP simply doesn't have this information available as a tie-breaker at all. It's a genuinely different signal from MED (which is a *policy* signal the remote AS advertises about its own preference) — AIGP represents *actual measured IGP cost*, giving it a fundamentally more objective character, closer to "real distance" than "requested preference."

### 13. AIGP Translation
**What:** Because AIGP is only trusted/propagated within a single AIGP-enabled domain by default, AIGP Translation lets you carry the *effect* of an AIGP value across an AS boundary where the remote AS doesn't (or won't) run AIGP — by re-encoding the AIGP value into either MED or a (specially-marked, transitive) Cost-Community with a **pre-bestpath** point-of-insertion.
**Why it matters (CCDE lens) — genuinely one of the most nuanced BGP-attribute-interaction topics in the whole workbook:** Translating into MED is weak — it isn't propagated end-to-end by the receiving AS to ITS OWN IGP-based decisions, and the remote AS's Local Preference still overrides it entirely. The workable technique is translating into a **cost-community with `poi pre-bestpath`, marked `transitive`** — a "big hammer" approach that makes the value the very *first* path-selection criterion evaluated (even before Weight), effectively letting the originating AS **override the remote AS's own best-path decision entirely**. This is explicitly noted as the *only* scenario where a cost-community is allowed to be transitive across an eBGP boundary — worth remembering as a specific, quotable exception to the normal rule that cost-communities are iBGP/confederation-local.
**Real-world example:** Two ASes with an extensive shared internal topology (e.g., merged companies, or a confederation-like relationship) want AS200's routers to genuinely respect AS100's real internal-distance-based path preference rather than AS200 independently re-deciding based on its own policy — pre-bestpath cost-community translation is the mechanism that achieves this level of cross-AS trust and override.

### 14. Cost-Community (iBGP)
**What:** An extended community, evaluated at a configurable **point of insertion (POI)** in the best-path algorithm — either right before the "lowest RID" tie-breaker (the IGP-cost POI) or at the very first step, before even Weight (the pre-bestpath POI) — used as a tie-breaker mechanism within a single AS/confederation.
**Why it matters (CCDE lens):** The most common legitimate use is automatically inserted when EIGRP is used as a PE-CE protocol (to correctly propagate EIGRP's own internal metric semantics through BGP) — but it's also a general-purpose, manually-assignable tie-breaker tool. The **community-ID field matters as much as the value**: a lower community ID always wins regardless of the numeric cost value, and only routes sharing the *same* community ID have their cost-community values actually compared against each other — meaning consistent ID assignment across all ASBRs/PEs in a design is a real, easy-to-get-wrong prerequisite for the mechanism to work as intended at all.

### 15. Cost-Community (confed eBGP)
**What:** Applies cost-community specifically across confederation eBGP sub-AS boundaries (where BGP sessions between confederation member-ASes are technically eBGP sessions, but conceptually behave more like iBGP within the overall confederation).
**Why it matters (CCDE lens):** Confederations blur the iBGP/eBGP distinction in exactly the way that makes cost-community's normal "iBGP/confederation-only, not transitive to true eBGP" rule need careful re-examination — reinforces that a candidate must understand confederation semantics precisely (member-AS eBGP sessions are NOT the same as inter-AS eBGP sessions with respect to which attributes propagate by default) rather than pattern-matching "eBGP session = treat like any other external peer."

---

## PART F — DDoS Mitigation, Part 1: RTBH (Remotely Triggered Black Hole)

### 16–20. Destination-Based RTBH, Source-Based RTBH, and their Community-Based & VRF variants

**Core mechanism (Destination-Based, the foundation):** A central **trigger router** (often the RR, or a dedicated NOC device with iBGP peering to all edge routers) injects a static route to Null0 for the victim/attack prefix, tagged with a specific tag/community. A route-map on every edge router matches that tag/community and rewrites the **next-hop** of the *received* BGP route to a pre-agreed discard address (e.g., `192.0.2.1`) that every router has a local static Null0 route for. The moment the trigger advertises the tagged route, every edge router simultaneously starts dropping traffic to that destination **at the network edge**, in hardware/CEF — not centrally, and not by hop-by-hop propagation delay.
```
#All PEs
ip route 192.0.2.1 255.255.255.255 null0
!
#Trigger router
ip route <victim-ip> 255.255.255.255 null0 tag 666
!
route-map RTBH permit 10
 match tag 666
 set community no-export 20:666
!
#Receiving PEs
route-map IBGP_IN permit 10
 match community 1
 set ip next-hop 192.0.2.1
```
**Why it matters (CCDE lens):** This is the single most important operational-security design pattern in SP BGP — the key design insight is *where* the drop happens: without RTBH, even a locally-configured Null0 route on one PE still lets attack traffic **transit the entire internal network** to reach that PE before being dropped, wasting core bandwidth and potentially still congesting internal links. RTBH pushes the drop decision to **every edge router simultaneously** via ordinary iBGP propagation — turning a "drop it once it arrives" defense into a "never let it enter the network at all" defense, using infrastructure (iBGP, static routing, CEF) that already exists, with a scale-out cost near zero.

**Source-based RTBH** is the mirror image, using **uRPF** (`ip verify unicast source reachable-via`) instead of a next-hop rewrite: rather than dropping traffic *destined to* a bad address, it drops traffic *sourced from* a bad address, checked at ingress against the RIB. The strict vs. loose mode distinction matters: **loose mode never accepts a Null0 route as satisfying the uRPF check** (regardless of strict/loose), which is precisely the property that lets you black-hole a *source* address via RTBH — a Null0-routed source address will always fail the uRPF check and be dropped, in either mode. Critically: source-based RTBH implicitly *also* achieves destination-based RTBH for whatever prefix is null-routed, since the Null0 route itself makes that address undeliverable as a destination too — a nice "one triggering action, two protective effects" property worth knowing.

**Community-based variants** replace the tag+route-map matching with a simpler, cleaner mechanism: the trigger router directly attaches the discard-triggering community at origination (rather than a locally-matched tag translated into a community downstream) — functionally equivalent but simpler to reason about and audit, especially when the trigger router itself is not also the point performing the tag-to-community translation.

**VRF variants (Provider-triggered vs. CE-triggered)** extend RTBH into the L3VPN world — Provider-triggered means the SP's own infrastructure decides and injects the black-hole trigger (typical for SP-managed DDoS protection services); CE-triggered means the *customer* can trigger their own black-hole via a specific BGP community sent from their CE to the PE, giving the customer limited self-service control over their own traffic without SP intervention — directly analogous to Provider Communities (Section 7) but specifically scoped to the RTBH use case.

**Real-world example:** During an active volumetric DDoS attack against a specific customer prefix, the SP's NOC (acting as trigger router) injects one tagged static route — within normal iBGP convergence time (seconds), every edge router across the entire SP footprint begins dropping attack traffic at ingress, protecting both the victim and the SP's own core bandwidth, without touching a single edge router's configuration individually.

---

## PART G — DDoS Mitigation, Part 2: BGP Flowspec

### 21–26. Flowspec (Global, VRF, w/ Redirect, w/ Redirect T-Shoot, w/ CE Advertisement)

**What:** BGP Flowspec (RFC 8955) extends the RTBH concept from "match on destination prefix only" to **match on an arbitrary N-tuple** (source, destination, protocol, port, packet length, DSCP, TCP flags, etc.) — and the *action* is no longer limited to "drop" — Flowspec routes can specify rate-limit, redirect (to a VRF or a specific next-hop, e.g., a DDoS scrubbing appliance), or traffic-marking (DSCP rewrite) actions.
```
#Enable flowspec AF and local install on enforcement points
flowspec
 address-family ipv4
  local-install interface-all
 address-family ipv6
  local-install interface-all
!
#Redirect to a scrubbing VRF via route-target
class-map type traffic match-all ATTACK_FLOW
 match source-address ipv4 <bad-src>
end-class-map
!
policy-map type pbr ATTACK_PBR
 class type traffic ATTACK_FLOW
  redirect nexthop route-target <asn>:<id>
!
flowspec address-family ipv4
 service-policy type pbr ATTACK_PBR
```
**Why it matters (CCDE lens) — the direct generational successor to RTBH:** RTBH can only ever say "drop everything to/from this exact address" — a genuinely blunt instrument that, in a source-based attack from many spoofed/distributed sources, may be useless (you can't black-hole every source individually) or, in a legitimate-looking-traffic attack, may be entirely the wrong shape of tool. Flowspec's N-tuple matching lets you precisely target the **actual attack signature** (e.g., "UDP, source port 53, packet length > 512 bytes" — a classic DNS amplification signature) without black-holing the victim's legitimate traffic on other ports/protocols entirely. The **redirect-to-VRF** action is the other major capability leap: instead of dropping traffic outright, you can surgically divert *only* the matched flow to a scrubbing appliance (in a "honeypot"/scrubbing VRF) for cleaning, then (in more advanced designs) reinject clean traffic — RTBH has no equivalent capability at all; it can only drop.

**Global vs. VRF Flowspec:** Global Flowspec protects the SP's own core/internet-facing infrastructure; VRF Flowspec applies the identical mechanism scoped to a specific customer's L3VPN — directly paralleling the RTBH Provider-triggered/CE-triggered VRF distinction, letting flow-level DDoS protection be offered as a **per-customer managed service**, not just an SP-wide capability.

**CE Advertisement variant:** Lets the *customer's own CE router* originate Flowspec rules toward the PE (analogous to CE-triggered RTBH) — a genuine self-service DDoS mitigation capability the SP can expose to customers with appropriate validation/guardrails (the PE must still apply sanity/safety checks on customer-originated Flowspec rules, since a malicious or misconfigured customer rule could otherwise black-hole or redirect traffic the customer doesn't actually own).

**T-Shoot variant:** Flowspec rules can silently fail to have any effect for reasons that mirror RTBH's own troubleshooting patterns — a rule not actually being locally-installed (`local-install` scope mismatch), a redirect target VRF/route-target not correctly configured, or (a genuinely Flowspec-specific gotcha) **rule validation failing** — Flowspec includes a built-in validation procedure (checking that the Flowspec rule's implied "best path" for the destination prefix originates from the same AS/next-hop as the actual best-path route in the BGP table) specifically to prevent a compromised or malicious peer from using Flowspec to hijack traffic under the guise of DDoS mitigation; a rule failing this validation is silently NOT installed, which is a common, non-obvious troubleshooting dead-end.

**Why it matters (CCDE lens, overall):** Flowspec vs. RTBH is a genuine "which tool for which job" design decision worth being able to articulate crisply: RTBH is simpler, more universally supported, and sufficient for coarse destination/source-based blocking; Flowspec is more precise (surgical N-tuple matching, non-drop actions like redirect/rate-limit) but has real validation/trust-model complexity and is a comparatively newer, less universally-deployed capability requiring more careful design (especially around who is allowed to originate rules, and to what scope).

---

## Interview Q&A

**Q1. Why can't you simply always trust the `no-prepend`/`replace-as` combination when using Local AS for an ASN migration?**
Because there's no way to prepend *only* the global (real) ASN while still hiding the local-as identity in certain combinations — and if the peer has `bgp enforce-first-as` enabled (on by default on many platforms), the peer requires the very first AS in the received AS_PATH to match the configured remote-as exactly. This constrains which keyword combination is actually viable for a given peer's configuration, and getting it wrong can prevent the session from ever establishing correctly, not just cause a cosmetic AS_PATH difference.

**Q2. Why is translating AIGP into MED considered a weak/ineffective technique compared to translating into a pre-bestpath cost-community?**
MED isn't propagated end-to-end into the receiving AS's own internal decisions the way true AIGP is, and the receiving AS's Local Preference (evaluated before MED) can still completely override it. A pre-bestpath cost-community, by contrast, is evaluated as the very first best-path criterion — even before Weight — giving the originating AS genuine override power over the remote AS's own path selection, which is why it's described as a "big hammer" approach.

**Q3. What's the fundamental capability gap between RTBH and BGP Flowspec?**
RTBH can only match on a single destination or source /32 (or similar) prefix and can only drop traffic — a blunt instrument. Flowspec matches on an arbitrary N-tuple (protocol, port, packet length, DSCP, etc.) and supports non-drop actions like rate-limiting and redirect-to-VRF for scrubbing — letting you precisely target an attack's actual signature without blanket-blocking a victim's legitimate traffic.

**Q4. Why does source-based RTBH work correctly with uRPF in loose mode, when loose mode is normally the "less strict" check?**
Because a Null0 route is never considered valid to satisfy the uRPF check in EITHER strict or loose mode — loose mode's relaxation only concerns which *interface* a valid route can be reachable via, not whether a Null0-routed (i.e., intentionally black-holed) source passes at all. This is exactly the property that makes source-based RTBH reliable regardless of which uRPF mode a given design otherwise requires for other reasons.

**Q5. Why does PIC Edge sometimes fail to provide fast failover even when correctly configured?**
PIC can only pre-stage a backup path that BGP has actually learned and considered viable — if route policy, filtering, or the underlying topology means only one genuinely viable path exists, there is nothing for PIC to pre-stage as an alternative. This is a topology/design-level constraint, not a configuration bug, and reinforces that fast-convergence features generally accelerate the use of a backup path that must already exist by design.

**Q6. Why does a well-designed provider-communities scheme matter for a large SP, beyond just "giving customers options"?**
It's the mechanism that lets customers safely self-service traffic-engineering requests (prepending, selective non-advertisement, local-preference hints) without needing direct config access to the provider's routers or opening a support ticket for every change — the provider defines the allowed "menu" of actions once in their own policy, and customers simply tag routes. Too narrow a menu limits customer flexibility; too broad creates unpredictable interactions and support burden, making the scheme's scope itself a real design decision.

---

## Memory Map

```
BGP (General Topic)
│
├── Migration & Path-Manipulation Fundamentals (1-4)
│     Conditional Advertisement ── "don't falsely promise reachability"
│     │  (same pattern as ISIS Conditional ATT Bit)
│     Local AS ── THE safe-ASN-migration pattern (same "change live
│     │  config without an outage" theme as ISIS/LDP/BFD elsewhere)
│     Aggregation/Deaggregation ── scale vs. TE-granularity trade-off
│     Non-Optimal eBGP Routing ── "BGP has no cost-awareness natively;
│           you must deliberately intervene at a specific best-path step"
│
├── QoS & Enterprise Multihoming (5-7)
│     QPPB ── decouples policy DISTRIBUTION (BGP) from ENFORCEMENT
│     │  (local QoS marking) — same philosophy as BGP AD in VPWS/VPLS
│     Multihomed Enterprise Challenge ── SYNTHESIS of multiple
│     │  individually-simple techniques into one coherent design
│     Provider Communities ── customer self-service via a defined
│           "menu" — scope-design IS the real decision
│
├── Load Balancing (8)
│     DMZ Link BW/UCMP ── SAME mechanism as BGP Multi-Homing Lab2;
│           this entry confirms it's a general eBGP capability
│
├── Fast Convergence — PIC family (9-11)
│     PIC Edge (Global) ── generalizes Multi-Homing's Backup/Repair
│     │  Path concept to the whole global table
│     PIC Troubleshooting ── failure is usually a TOPOLOGY problem
│     │  (no real backup path exists), not a config problem
│     PIC Edge for VPNv4 ── DIRECTLY DEPENDS on the RD design
│           decision from BGP Multi-Homing Labs 9-10 (same RD
│           required for a backup path to even be visible)
│
├── Cross-AS Metric Propagation (12-15)
│     AIGP ── true measured IGP cost, unlike MED's "requested preference"
│     AIGP Translation ── pre-bestpath cost-community = ONLY case
│     │  where cost-community is allowed transitive across eBGP —
│     │  a "big hammer" override of the remote AS's own decision
│     Cost-Community (iBGP/confed) ── community-ID must match
│           across ASBRs or values are never even compared
│
└── DDoS Mitigation — two generations of the same idea (16-26)
      RTBH (16-20): drop-only, prefix-granular, but SCALES TRIVIALLY
      │  (one trigger route, propagates via existing iBGP to every
      │  edge router simultaneously) — drops AT THE EDGE, not
      │  after transiting the core
      │  Destination-based / Source-based (uRPF) / Community-based /
      │  VRF Provider-triggered vs CE-triggered (mirrors Provider
      │  Communities' self-service pattern, Section 7)
      │
      └─ Flowspec (21-26): GENERATIONAL UPGRADE — N-tuple matching,
            non-drop actions (redirect-to-VRF for scrubbing, rate-limit)
            Built-in rule VALIDATION (checks against actual best-path)
            specifically to prevent Flowspec-based traffic hijacking
            CE Advertisement variant = Flowspec's own CE-triggered
            self-service capability, same pattern as RTBH's
```

**The one throughline connecting nearly every section in this file:** BGP attributes (communities, AIGP, cost-community, Flowspec NLRIs) are fundamentally a way to **carry policy/intent as data through the control plane**, letting a single decision point (a trigger router, an originating AS, a customer's CE) influence behavior at many distributed enforcement points simultaneously — without needing direct configuration access to each one. RTBH, QPPB, Provider Communities, and Flowspec are all the same underlying design pattern applied to different problems (DDoS mitigation, QoS marking, customer self-service TE).

---

## CLI Cheat Sheet

| Purpose | Command |
|---|---|
| Conditional advertisement | `neighbor <ip> advertise-map <map> [exist-map\|non-exist-map] <cond-map>` |
| Aggregate with AS_SET | `aggregate-address <net> <mask> as-set [summary-only]` |
| Local AS (basic) | `neighbor <ip> local-as <asn>` |
| Local AS (full migration flexibility) | `neighbor <ip> local-as <asn> no-prepend replace-as dual-as` |
| DMZ link bandwidth (UCMP) | `bandwidth <kbps>` on interface + `neighbor <ip> dmzlink-bw` + `send-community both` |
| AIGP enable | `neighbor <ip> aigp` (or address-family default depending on platform) |
| AIGP → cost-community translation | `aigp send cost-community <id> poi pre-bestpath transitive` + `send-community extended` |
| Cost-community POI options | `poi igp-cost` (before lowest-RID step) or `poi pre-bestpath` (before Weight) |
| RTBH: discard next-hop static route | `ip route <discard-addr> 255.255.255.255 null0` |
| RTBH: trigger static route with tag | `ip route <victim> 255.255.255.255 null0 tag <n>` |
| RTBH: tag → community translation | `route-map RTBH` → `match tag <n>` → `set community no-export <asn>:<n>` |
| RTBH: rewrite next-hop on receipt | `route-map IN` → `match community <n>` → `set ip next-hop <discard-addr>` |
| Source-based RTBH: uRPF | `ip verify unicast source reachable-via [rx\|any] [allow-self-ping] [<acl>]` |
| Flowspec: enable + local install | `flowspec` → `address-family ipv4` → `local-install interface-all` |
| Flowspec: match + redirect to VRF | `class-map type traffic` → `match ...` + `policy-map type pbr` → `redirect nexthop route-target <rt>` |
| Flowspec: activate AF to peer | `router bgp <asn>` → `address-family ipv4 flowspec` → `neighbor <ip> activate` |
| Verify Flowspec table | `show bgp ipv4 flowspec [detail]` |
| Verify PIC backup path | Platform-specific — check FIB for pre-installed backup/repair path entries |
| Verify uRPF drops | `show ip interface <int>` (drop counters) / `show cef interface <int>` |

---
*Source: CCIE-SP v5.1 Labs — BGP (general topic) section (31 labs): Conditional Advertisement, Aggregation and Deaggregation, Local AS, BGP QoS Policy Propagation, Non-Optimal eBGP Routing, Multihomed Enterprise Challenge (+XRv), Provider Communities (+XRv), Destination-Based RTBH (+Community-Based, +VRF Provider/CE-triggered), Source-Based RTBH (+Community-Based, +VRF Provider-triggered), DMZ Link BW Lab1/Lab2, PIC Edge in the Global Table, PIC Edge Troubleshooting, PIC Edge for VPNv4, AIGP, AIGP Translation, Cost-Community (iBGP), Cost-Community (confed eBGP), Flowspec (Global IPv4/6PE, VRF, w/ Redirect ×2, w/ Redirect T-Shoot, w/ CE Advertisement). Some sub-topics supplemented with standard BGP/RTBH/Flowspec behavior (RFC 8955 and related) where specific lab page content was not directly retrievable.*
