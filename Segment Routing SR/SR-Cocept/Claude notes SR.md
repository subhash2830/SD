# CCDE-Level Segment Routing Deep Notes

### SR: SRGB Modification, SR with ExpNull, SR Anycast SID



---

## Table of Contents

1. Segment Routing Global Block (SRGB)
2. SR Label Space Architecture (Reserved / Static / SRLB / SRGB / Dynamic)
3. Prefix-SID Index vs. Label & Cross-SRGB Label Stitching
4. Penultimate Hop Popping (PHP) in Segment Routing
5. Explicit-Null in Segment Routing (P-flag & E-flag)
6. ISIS SR Advertisement — Router Capability TLV, SR-Capability/SR-Algorithm, Prefix-SID Sub-TLV Flags
7. Anycast SID
8. N-flag Semantics and TI-LFA Interaction (Anycast SID Protection Correctness)

---

## 1. Segment Routing Global Block (SRGB)

**1. Definition** The SRGB is a locally-configured, contiguous range of MPLS label values on a router that is reserved for globally significant Segment Routing prefix segments (Prefix-SIDs). By default on IOS-XR it is **16000–23999** (16,000 labels). Every SR-enabled node advertises its own SRGB range into the IGP.

**2. Why it exists** In classic LDP, every router independently allocates a random label per FEC and advertises it via a label-mapping message — labels are _locally_ significant and unpredictable. SR removes the signaling protocol entirely; there is no LDP session, no label-mapping exchange. So SR needed a way to let a controller/operator predict labels deterministically (this is what makes SR-TE explicit label stacks, PCE, and SR Policies possible) while still allowing each router to pick its own local label block for hardware/platform reasons. The SRGB solves this: it lets each node reserve its own block, but ties the _meaning_ of a label inside that block to a globally-flooded **index**, so everyone in the domain can independently compute what a given prefix-SID means, without ever exchanging a label-mapping message.

**3. How it works (step-by-step)**

- A Prefix-SID is not configured as a raw label — it's configured as an **index** (e.g., `prefix-sid index 7`).
- Each node picks a local SRGB block (default 16000–23999, or custom, e.g., 30000–30999).
- The **local label** a node uses for its own prefix = `local SRGB base + index`.
- The SRGB itself is advertised in the IGP as a **capability**, not per-prefix:
    - ISIS: **Router Capability TLV (242)** → **SR-Capability sub-TLV** carries the SRGB start value + range size (plus the **SR-Algorithm sub-TLV** listing supported algorithms, e.g., 0 = SPF).
    - OSPF: **Router Information (RI) Opaque LSA** → **SID/Label Range TLV**.
- Every other router in the domain now knows: "Node X's SRGB starts at B, and this prefix has index I" → so any router can compute the **label X expects to receive** = `B + I`, without X ever telling anyone the label directly.

Example from the lab:

```
R3: segment-routing
      global-block 30000 30999

R5: router isis 1
      segment-routing global-block 50000 50999
```

If R5's Lo0 has prefix-SID index 7:

- On any node using the _default_ SRGB (16000-23999): the value for that index is 16007.
- On R3 (custom SRGB 30000-30999): the value is 30007.
- On R5 itself (custom SRGB 50000-50999): the value is 50007 (the label R5 actually expects on ingress).

So when R1 (default SRGB) forwards traffic destined to R5's prefix via next-hop R3, R1 doesn't push its _own_ SRGB-based label (16007) onto the packet — it pushes the label **R3 expects to receive**, i.e., 30007, because MPLS label significance is always downstream-allocated, per-hop. R1's LFIB entry: incoming label 16007 → swap to 30007 → out toward R3. R3 then swaps 30007 → 50007 toward R5.

**Configuration hierarchy**: SRGB can be set globally (`segment-routing global-block` under SR config mode) or per-IGP-instance (`router isis 1 → segment-routing global-block`). The per-IGP command **overrides** the global one — allowing a single node to run different SRGBs per IGP process (useful in multi-instance / multi-area SR designs).

**4. Real-world use case** Large SP/hyperscale networks that inherited non-uniform label ranges across different hardware platforms (older PE/ASR boxes with small LFIB TCAM vs. new P routers) use custom, non-overlapping SRGBs so each platform's forwarding-table constraints are respected while running one unified SR domain. Also critical in brownfield SR/LDP migration: SRGB ranges are carved out of a hardware's dynamic label space so they don't collide with LDP's dynamically-assigned range.

**5. Failure scenario**

- **Overlapping SRGBs** between two nodes: IGP SR capability advertisement detects the collision, and prefix-SID programming for the colliding index becomes ambiguous/unusable — the router typically refuses to program the conflicting label or marks the SID inconsistent, causing black-holing or fallback to non-SR (IP) forwarding for that prefix.
- If the **dynamic label range** isn't shrunk to account for a custom SRGB, you get label collisions between SR and dynamically-signaled (LDP/RSVP/BGP-LU) label spaces — traffic gets mis-forwarded because the LFIB can't disambiguate.
- Changing SRGB live without make-before-break planning causes transient traffic loss while adjacent routers reconverge their SR label swaps to the new base.

**6. Design insight** Strong best practice: **keep the SRGB identical on every node in the domain.** Uniform SRGB makes a prefix-SID's label globally significant everywhere (not just its index) — this is what lets you write a simple, human-readable label stack for SR-TE and know exactly which routers those labels represent, network-wide. Non-uniform SRGBs are functionally correct but destroy the "one label = one node everywhere" mental model, complicate troubleshooting, and complicate SR-TE explicit-path computation (a PCE/controller must track per-node SRGB tables instead of one global mapping). Reserve non-uniform SRGB only for platforms with genuine hardware label-space constraints, in predictable, well-documented, non-overlapping ranges.

**7. Interview-ready answer** "The SRGB is the local label range each SR node reserves for global prefix-SIDs; a node's actual label for a given prefix is `SRGB-base + index`, and the SRGB itself is flooded via the IGP (ISIS Router Capability TLV 242 / OSPF RI LSA) as a capability so every router can compute every other router's label without any label-mapping protocol. It's per-node local significance with a globally-flooded index — best practice is to keep the SRGB identical everywhere so labels are globally predictable, essential for clean SR-TE explicit paths and troubleshooting."

---

## 2. SR Label Space Architecture (Reserved / Static / SRLB / SRGB / Dynamic Ranges)

**1. Definition** On an SR-capable IOS-XR node, the full 20-bit MPLS label space (0–1,048,575) is statically partitioned into functional zones: **Reserved (0–15)**, **Static (16–14999)**, **SRLB — Segment Routing Local Block (15000–15999 by default)**, **SRGB (16000–23999 by default)**, and **Dynamic (24000–max, platform dependent)**.

**2. Why it exists** A single LFIB has to simultaneously host labels from very different sources: hardware-reserved special-purpose labels, manually-configured static labels, locally-significant SR segments (Adj-SIDs, binding SIDs), globally-significant SR segments (Prefix-SIDs), and dynamically-signaled protocols (LDP, RSVP-TE, BGP labeled-unicast). Without a partitioning scheme, two independent allocation mechanisms (e.g., LDP's dynamic pool and SR's SRGB) could hand out the same label value for two different purposes, causing an unrecoverable forwarding collision. Partitioning by function guarantees no allocator ever needs to coordinate with another allocator at run-time.

**3. How it works**

- **0–15: Reserved special-purpose labels.**
    - 0 = IPv4 Explicit Null
    - 1 = Router Alert
    - 2 = IPv6 Explicit Null
    - 3 = Implicit Null (the "pop" instruction used for PHP — never appears on the wire; a control-plane/local signal meaning "pop the label")
- **16–14999: Static label range** — for manually configured static LSPs/cross-connects.
- **15000–15999: SRLB (Segment Routing Local Block)** — reserved for _locally_ significant SR SIDs the operator explicitly wants to pin, most notably **Binding-SIDs** (used in SR-TE policies) and manually assigned Adjacency-SIDs. Unlike the SRGB, SRLB values are meaningful only on the local node.
- **16000–23999: SRGB** — for globally significant Prefix-SIDs, computed as `base + index`.
- **24000–max (platform maximum): Dynamic range** — used for LDP, RSVP-TE, BGP labeled-unicast, and dynamically assigned SR Adjacency-SIDs.
- Critically: **the SRGB "eats into" whatever range it's placed in.** Moving a node's SRGB to 30000-30999 makes the dynamic range for that node discontiguous: `24000-29999` + `31000-max`. The router recomputes available dynamic space around wherever the SRGB is carved out. Verifiable via the LSD (Label Switch Database) show commands, and the command showing default dynamic label min/max boundaries.

**4. Real-world use case** Operators verify this partitioning heavily during SR/LDP interworking and migration projects: before turning on SR, confirm the intended SRGB doesn't overlap any label already in active use by LDP's dynamic pool on every box in the path — otherwise mid-migration you get intermittent, load-dependent traffic loss that's hard to diagnose because it only affects specific FECs whose dynamically-assigned LDP label happens to collide with an SR index-derived label.

**5. Failure scenario**

- Enlarging/relocating the SRGB without checking overlap against the platform's dynamic label pool → label collision → silent double-use of one label for two different FECs → intermittent packet mis-delivery/drops that appear random and are hard to correlate without checking the LSD.
- Running out of SRLB space when a design pins too many binding-SIDs/manual Adj-SIDs on one node (heavy SR-TE policy deployment) — new local SID allocations fail or fall back to dynamic, breaking the assumption that BSIDs are stable/predictable.

**6. Design insight** Size the SRGB _before_ deployment based on: (a) total number of loopback/Prefix-SID-carrying prefixes expected network-wide (SRGB size must be ≥ max index used, with headroom for growth), and (b) leave the SRLB at defaults unless there's a specific need for many static/binding SIDs. Treat SRGB sizing like IP address planning — undersizing forces a painful re-numbering (re-indexing) event later; oversizing wastes dynamic label space needed for LDP/RSVP-TE coexistence during migration.

**7. Interview-ready answer** "IOS-XR statically partitions the label space into reserved (0-15), static (16-14999), SRLB (15000-15999) for local SIDs like binding-SIDs, SRGB (16000-23999 default) for global prefix-SIDs, and dynamic (24000+) for LDP/RSVP/BGP-LU. This up-front partitioning avoids run-time collisions between independent allocators; relocating the SRGB automatically makes the dynamic range discontiguous around it, so SRGB sizing must be planned like address planning — undersized SRGBs force painful re-indexing later."

---

## 3. Prefix-SID Index vs. Label & Cross-SRGB Label Stitching

**1. Definition** A Prefix-SID is configured and flooded as an **index** (an offset), never as a raw label. The _label_ that appears in any given router's LFIB for that Prefix-SID is derived locally as `that router's own SRGB base + index`. When SRGBs differ node-to-node, each router along the path performs a normal MPLS label **swap**, translating from "my downstream neighbor's SRGB-based label" — functionally identical to ordinary hop-by-hop label swapping, just computed rather than signaled.

**2. Why it exists** This index/label separation is the single design decision that lets SR eliminate the LDP signaling protocol entirely. If SR required exchanging literal label values (like LDP), each node would need a session/protocol to announce its chosen label per prefix — reintroducing the very signaling overhead SR removed. By flooding a small integer index through the IGP (which is already flooding reachability information) and letting every node deterministically compute labels from `SRGB + index`, SR gets full any-to-any label agreement for free, riding on infrastructure that already exists.

**3. How it works (worked example)** Topology: R1 — R3 — R5 (ISIS, SR enabled). R5's Lo0 = prefix-SID index 7.

- Default SRGB everywhere: label for index 7 = 16007 on every node — fully globally significant.
- Now change R3's SRGB to 30000-30999 and R5's SRGB to 50000-50999.
- Every node recomputes:
    - R1 (default SRGB): local FEC label for R5's prefix = 16007.
    - R3 (custom): local incoming label for R5's prefix = 30007.
    - R5 (custom): local incoming label for its own prefix = 50007 (what R5 actually expects to receive).
- R1's forwarding entry to reach R5's prefix via next-hop R3: **push label 30007** (not 16007) — because MPLS is downstream-allocated: the label on the wire must be the value the _next-hop_ expects, regardless of the ingress router's own SRGB.
- At R3: incoming label 30007 → look up FEC → next-hop R5 → swap to 50007 → forward.
- At R5: incoming label 50007 matches its own Prefix-SID → pop (or swap to explicit-null; see #4/#5) and do an IP lookup or forward natively.
- Verified via traceroute: "R1 uses R3's correct global label, and R3 uses R5's correct global label" — proving each hop pushes the label based on the _next node's_ SRGB, computed independently, zero signaling.

**4. Real-world use case** This mechanism is exactly what makes SR interoperable across multi-vendor domains where different platforms may have hardware-dictated SRGB constraints — as long as every node correctly advertises its own SRGB via the IGP, label stitching "just works" without any cross-vendor label-mapping protocol negotiation, unlike historical LDP interop issues between vendors' targeted-LDP sessions.

**5. Failure scenario**

- If a router fails to correctly flood its SRGB capability (IGP adjacency flap mid-SRGB-change, or an implementation bug drops the SR-Capability sub-TLV), downstream routers keep using a _stale_ SRGB base → wrong label computed → receiving node's LFIB has no matching entry → **label lookup miss → drop** at that hop. Classic "SRGB change caused an outage" failure if the domain doesn't fully converge before traffic relies on the new mapping.
- A misconfigured **duplicate index** on two different prefixes on the same node is more dangerous than with LDP (which self-heals via unique per-FEC allocation) — two prefixes claiming index 7 on the same router causes an index collision the IGP flags as an error, and one or both prefix-SIDs become unusable/withdrawn.

**6. Design insight** This is why "keep SRGB uniform" matters so much architecturally: uniform SRGB turns index-based label computation into a strict 1:1 mapping (label = index globally), massively simplifying SR-TE explicit-path label-stack construction — a controller/PCE needs only one global index table instead of a per-node SRGB lookup table for every hop. In heterogeneous-SRGB designs, any centralized controller building explicit SID-lists must resolve labels per-hop against each transit node's actual SRGB, adding real computational and operational complexity that scales poorly as the domain grows.

**7. Interview-ready answer** "A Prefix-SID is flooded as an index, not a label; each router computes its local label as `own SRGB + index`. Forwarding still obeys classic downstream-allocated MPLS semantics — a router always pushes the label its next-hop expects — so with different SRGBs you effectively get normal label swapping performed via computation instead of signaling. This is the core trick that lets SR drop LDP entirely while every node still agrees on labels."

---

## 4. Penultimate Hop Popping (PHP) in Segment Routing

**1. Definition** PHP is the default behavior where the **second-to-last router** on a label-switched path removes (pops) the top label _before_ forwarding the packet to the final destination node, rather than making the final node do both a label pop and a subsequent IP/next-label lookup.

**2. Why it exists** The final egress node has to do a lookup on whatever is under the top label anyway (the underlying IP header, or another label). If it also had to pop the top MPLS label as a separate operation, that's two lookups for one packet at the most resource-constrained point in the path (often a lower-end PE, or the CPU-adjacent egress interface). PHP shifts the pop operation one hop earlier, to the penultimate node, so the final node only ever performs a single, simple operation. This optimization carried over unchanged from LDP/RSVP-TE into SR.

**3. How it works**

- In SR, PHP is signaled implicitly via the **P-flag (No-PHP flag)** in the Prefix-SID sub-TLV. By default this flag is **clear (0)**, meaning "PHP IS allowed" — the penultimate router should pop.
- Mechanically, IOS-XR represents "pop" as **Implicit-Null (label value 3)** internally in the LFIB — this value never actually goes on the wire; it's a local instruction meaning "remove the top label and forward what's underneath."
- Example: R7's Lo1 has prefix-SID index 7. R5 is directly connected to R7 (penultimate hop for that destination). By default, R5's LFIB entry for label 16007 (or whatever R7's SRGB-derived value is) says **pop and forward** — R5 sends R7 a plain IP packet (or whatever remains under the popped label), not an MPLS-labeled one.
- R7 therefore never sees an MPLS label for its own directly-owned prefix in the default case — it just receives IP.

**4. Real-world use case** PHP is the default, always-on optimization in virtually every production MPLS/SR network for ordinary prefix reachability — reducing forwarding burden on egress PEs at internet/edge scale, where PE routers handle millions of pps of customer-facing traffic and every saved lookup matters at line rate.

**5. Failure scenario**

- The problem PHP creates: once the label is popped one hop early, any information carried only in that top label is lost before it reaches the true final node. The classic casualty is **MPLS EXP bits / QoS marking** — if the final router needs to make a QoS decision based on the label's EXP bits (e.g., an egress queueing policy keyed to MPLS EXP), PHP silently strips that information one hop too early, causing **QoS misclassification** at the true edge.
- This is exactly the scenario the "SR with ExpNull" lab solves: by default, R7 wouldn't see any label at all for traffic to Lo1, so any EXP-based policy on R7 itself would have nothing to inspect.

**6. Design insight** Keeping PHP on (default, more efficient) vs. disabling it (Explicit-Null) is a genuine architecture trade-off between forwarding efficiency and QoS-classification fidelity at the edge. Large-scale SP cores generally keep PHP everywhere except at boundaries where EXP-based classification at the true final router is contractually or operationally required (e.g., certain intercarrier interconnects, or platforms whose QoS ACLs can only match MPLS EXP, not post-pop IP DSCP). Disable PHP only selectively, on the specific nodes/prefixes that need it — not network-wide.

**7. Interview-ready answer** "PHP means the penultimate router pops the top label so the final node does a single lookup instead of pop-then-lookup — an efficiency optimization inherited from LDP/RSVP-TE. In SR it's signaled by the P-flag (No-PHP flag) in the Prefix-SID sub-TLV, defaulting to 'PHP allowed.' The trade-off is that PHP strips MPLS EXP bits one hop early, so if the true egress node needs EXP-based QoS decisions, you must explicitly disable PHP for that prefix."

---

## 5. Explicit-Null in Segment Routing (P-flag & E-flag)

**1. Definition** Explicit-Null is an alternative to PHP where the penultimate router, instead of popping the top label entirely, **swaps it for a well-known reserved "explicit null" label** (0 for IPv4, 2 for IPv6) and forwards it still-labeled to the final node — preserving a labeled packet, and critically, preserving the EXP/traffic-class bits, all the way to the true egress node.

**2. Why it exists** PHP's efficiency gain (Concept #4) comes at the cost of losing EXP-carried QoS information one hop early. Explicit-Null lets an operator choose "efficiency vs. QoS fidelity" per-prefix: retain a label (with fresh, correctly-marked EXP) all the way to the final node, so it can classify traffic based on the MPLS header rather than needing to already have decoded the underlying payload.

**3. How it works (protocol-level, the "hidden" detail)**

- In LDP/RSVP-TE this was accomplished the classic way: the egress router literally advertised the reserved Explicit-Null label in its label-mapping message instead of Implicit-Null — a straightforward point-to-point signal.
- **The hidden complexity SR introduces**: SR has no label-mapping protocol at all — there is no message where a node says "please don't pop my label, use explicit-null instead." SR instead repurposes the **flags octet already embedded in the Prefix-SID sub-TLV** (the same IGP TLV carrying the index) to encode this request declaratively, flooded to the whole domain along with the prefix itself:
    - **P-flag (No-PHP flag)**: when **set**, tells every upstream neighbor "do NOT perform PHP for this Prefix-SID."
    - **E-flag (Explicit-Null flag)**: when **set** (only meaningful together with P-flag=1), tells every upstream neighbor "instead of popping, swap to Explicit-Null (0 for v4 / 2 for v6) and forward."
- Configuration example from the lab:
    
    ```
    R7: router isis 1      int Lo1       address-family ipv4 unicast        prefix-sid index 7 explicit-null
    ```
    
- Before this config: R7 advertises its Prefix-SID with **P-flag clear** and **E-flag clear** (normal PHP behavior, silently allowed by all neighbors).
- After this config: R7's IGP advertisement now carries **P-flag = 1 (No-PHP)** and **E-flag = 1 (Explicit-Null)**.
- Every directly-connected neighbor that is penultimate hop toward R7's Lo1 (e.g., R5) receives this flooded LSP, parses the flags, and reprograms its own LFIB entry: instead of "pop," R5's entry becomes **"swap top label to Explicit-Null (0) and forward."**
- Verification: enabling `mpls oam` and running a labeled traceroute — with explicit-null active, R7 receives a genuinely MPLS-labeled packet (label 0) on the final hop, not plain IP.

**4. Real-world use case** Used at PE routers or ASBRs running egress QoS policies keyed on MPLS EXP bits rather than IP DSCP/CoS — common in SP networks where the egress QoS policy was built around label-switched infrastructure and needs traffic-class marking intact on the very last hop (e.g., voice/video prioritization enforced up to the PE-CE handoff, or strict SLA enforcement at a wholesale interconnect where the underlying customer IP header can't be trusted/inspected for classification).

**5. Failure scenario**

- If explicit-null is enabled on a prefix but the penultimate router doesn't correctly re-derive/re-mark EXP bits on the swapped Explicit-Null label (implementation or policy gap), a labeled packet with EXP=0 arrives at the final node — worse than PHP, since the operator believed QoS fidelity was preserved but it silently wasn't.
- If only some upstream neighbors correctly parse/honor the E-flag (legacy/third-party bug), behavior is **inconsistent across ECMP paths** — some paths deliver a labeled packet, others deliver plain IP, breaking any downstream policy assuming uniform behavior.
- Misconfiguring explicit-null on a prefix that doesn't need EXP preservation forces every upstream neighbor into an extra swap-instead-of-pop operation, undermining PHP's efficiency for no benefit.

**6. Design insight** A textbook example of "protocol minimalism creates elegant but non-obvious mechanisms": SR achieved a behavior that used to require a separate signaling exchange (LDP label-mapping message) by adding two bits to an already-flooded TLV. Recognize this pattern broadly in SR — almost all "special per-hop behavior" requests (No-PHP, Explicit-Null, Node vs. Anycast semantics via N-flag, local significance via L-flag) are expressed as **flags on flooded state**, not point-to-point signaling. Apply explicit-null surgically, per-prefix, only where QoS-at-final-hop genuinely requires it — don't blanket-enable it network-wide.

**7. Interview-ready answer** "SR has no signaling protocol, so instead of an LDP-style explicit-null label-mapping message, it encodes the request as two flags in the Prefix-SID sub-TLV: the P-flag (No-PHP) tells upstream neighbors not to pop, and the E-flag tells them to swap to the reserved Explicit-Null label (0/2) instead — preserving EXP bits all the way to the final hop for EXP-based QoS policies at the true egress node."

---

## 6. ISIS SR Advertisement — Router Capability TLV, SR-Capability/SR-Algorithm Sub-TLVs, and Prefix-SID Sub-TLV Flags

**1. Definition** ISIS carries all Segment Routing state using standard IS-IS extensibility mechanisms (RFC 8667): node-level SR capabilities are carried in the **Router Capability TLV (Type 242)** via its **SR-Capability** and **SR-Algorithm** sub-TLVs, while per-prefix SID information rides inside existing prefix-reachability TLVs (135/235/236/237) as a **Prefix-SID sub-TLV (Type 3)** containing a flags octet, an algorithm field, and the SID index/label value.

**2. Why it exists** SR deliberately avoided inventing a new IGP or signaling protocol — instead it extends existing link-state IGP flooding, because IS-IS/OSPF already reliably and efficiently flood arbitrary opaque state to every router in a domain with guaranteed consistency (via LSP/LSA flooding and SPF-driven database sync). Reusing this machinery gets SR global, loop-free, guaranteed-consistent state distribution "for free," instead of building and hardening a whole new reliable-flooding protocol.

**3. How it works**

- **Node capability advertisement** (what makes SRGB-modification and label computation possible):
    - **Router Capability TLV (242)** — a general-purpose extensible container, reused here.
        - **SR-Capability sub-TLV**: advertises whether the node supports SR (flags: I-flag = MPLS IPv4 support, V-flag = MPLS IPv6 support), and one or more SID/Label range entries — each entry states an SRGB range (base + range-size). This is exactly the mechanism that lets a node flood a custom SRGB (e.g., 30000-30999) to the rest of the domain.
        - **SR-Algorithm sub-TLV**: lists which SPF algorithms the node supports for Flex-Algo purposes (Algorithm 0 = standard SPF, Algorithm 1 = strict SPF, 128-255 = Flex-Algo custom definitions).
- **Per-prefix SID advertisement**:
    - Piggybacked onto the standard Extended IP Reachability TLVs the router already floods for its prefixes (Type 135 for IPv4, 236/237 for IPv6, or MT variants 235/236 in multi-topology deployments).
    - Inside, a **Prefix-SID sub-TLV (Type 3)** contains:
        - **Flags octet**:
            - **R-flag** (Re-advertisement) — set when a prefix-SID is re-advertised from another IGP level/area (relevant to inter-area/inter-level leaking).
            - **N-flag** (Node-SID) — set when the SID represents a true, single-node loopback (must be cleared for anycast — see Concept #7).
            - **P-flag** (No-PHP flag) — set to disable penultimate-hop popping.
            - **E-flag** (Explicit-Null flag) — set together with P-flag to request Explicit-Null swap instead of pop.
            - **V-flag** (Value flag) — set if the SID field carries an absolute label value rather than an index (rare; index-based is standard).
            - **L-flag** (Local flag) — set if the SID has only local significance (paired with V-flag typically).
        - **Algorithm field** (1 octet) — which SPF algorithm this SID is associated with (0 = normal SPF; enables the same prefix to have different SIDs per Flex-Algo topology).
        - **SID/Index/Label field** — the actual index (typically) or absolute label (if V/L set).

**4. Real-world use case** This TLV-based, capability-driven design allows incremental, prefix-by-prefix and node-by-node SR rollout in a live production IGP domain — enable SR on one router at a time, and it simply starts flooding its SR-Capability + Prefix-SID sub-TLVs; routers that don't understand SR-specific sub-TLVs ignore the unrecognized sub-TLV type (standard forward-compatibility) and continue routing normally via plain IP reachability, while SR-capable routers use the extra information to build LFIB entries. This is exactly what makes phased SR/LDP migrations operationally viable.

**5. Failure scenario**

- If two IS-IS levels/areas are involved and the **R-flag isn't correctly honored** during route leaking/redistribution, a Prefix-SID can be interpreted as domain-local when it actually originated elsewhere, causing incorrect SID reuse or index collisions across area boundaries.
- A platform bug or misconfiguration that fails to flood the **SR-Capability sub-TLV** at all (e.g., an interface not enabled for the correct address-family under ISIS) results in that node's SRGB never being learned by the rest of the domain — every other router falls back to assuming the default SRGB for that node's index calculations, causing the same wrong-label / black-hole failure described in Concept #3.
- Missing/misconfigured SR-Algorithm sub-TLV on a node meant to participate in Flex-Algo excludes that node from Flex-Algo-constrained SPF computations network-wide.

**6. Design insight** Recognizing that SR is "just more IGP TLVs" (not a new protocol) should shape debugging and design: virtually every SR misbehavior (wrong label, missing PHP behavior, Flex-Algo exclusion, anycast/TI-LFA breakage) can be root-caused by directly inspecting the IS-IS/OSPF link-state database (`show isis database detail`, `show isis segment-routing label table`) rather than any SR-specific control-plane tool — there is no separate SR control-plane; the IGP database _is_ the SR control plane. Major operational simplification versus RSVP-TE (which has its own signaling state and troubleshooting surface).

**7. Interview-ready answer** "SR is deliberately not a new protocol — it extends IS-IS's Router Capability TLV (242) with SR-Capability and SR-Algorithm sub-TLVs for node-level state (SRGB, supported algorithms), and extends the existing prefix-reachability TLVs with a Prefix-SID sub-TLV carrying flags (N, P, E, R, V, L) and an index/label. Because it rides on standard link-state flooding, unsupported routers just ignore the sub-TLVs and keep routing normally — which is what makes incremental SR rollout and troubleshooting via the normal IGP database possible."

---

## 7. Anycast SID

**1. Definition** An Anycast SID is a Prefix-SID assigned to a prefix that is intentionally configured **identically on more than one physical router** (an anycast address), with the **N-flag explicitly cleared**, so that the SID represents "any one member of this set of nodes," not a single specific device.

**2. Why it exists** Ordinary Node-SIDs (N-flag set) are a promise: "this SID always leads to exactly this one specific router." That promise is useful for most designs, but sometimes you want traffic-engineering flexibility to reach "one of a redundant pair, whichever is closer/reachable" rather than a single named box — e.g., steering traffic toward either of two PE routers, ECMP-load-shared and mutually protecting each other, using a single SID in an explicit SR-TE path instead of needing separate paths per specific device. Anycast SIDs give SR-TE/PCE designs a compact way to express "go to this redundant group" as one segment.

**3. How it works**

- Configure the same IP address on a loopback on two (or more) routers — in the lab, `46.46.46.1/32` on `Lo46` of both R4 and R6.
- Assign the **same Prefix-SID index** (46) on both nodes, but explicitly **clear the N-flag**:
    
    ```
    R4, R6:  int Lo46   ip address 46.46.46.1/32  router isis 1   int Lo46    address-family ipv4 unicast     prefix-sid index 46 n-flag-clear
    ```
    
- With the N-flag cleared, IS-IS floods the Prefix-SID for 46.46.46.1/32 from both R4 and R6, each stating "I am also a valid instance of this SID, but I am not claiming to be the unique node identified by it."
- ECMP works exactly as expected even without touching the N-flag at all — SPF sees two equal-cost paths to the same prefix and load-shares label-switched traffic across both R4 and R6 regardless of the flag setting. **Key nuance**: normal forwarding to an anycast SID doesn't actually require the N-flag to be cleared; it works either way for plain SPF/ECMP forwarding.
- If you forget to clear the N-flag, IOS-XR detects the inconsistency and logs a warning:
    
    ```
    %ROUTING-ISIS-6-ANYCAST_N_FLAG_SET: 46.46.46.1/32 is an anycast route with thesegment-routing Node flag set on R4
    ```
    
    — but ordinary ECMP forwarding to R4/R6 still functions correctly despite the warning. The real breakage only shows up with TI-LFA (see Concept #8).

**4. Real-world use case** Anycast SIDs are widely used to represent redundant service edges or redundant transit pairs as a single steerable segment in SR-TE policies — e.g., a pair of redundant data-center gateway routers, a pair of redundant peering routers to a specific upstream, or a pair of ABRs providing dual entry into an area — letting a controller build one explicit path segment ("go via this anycast pair") instead of maintaining two separate primary/backup paths keyed to individual router IDs.

**5. Failure scenario** See Concept #8 for the detailed TI-LFA failure — but at a high level: any mechanism (not just TI-LFA) that assumes "N-flag set = this SID identifies a single specific device" makes incorrect topological assumptions if the N-flag is left set on an anycast prefix, because in reality there are two (or more) physically distinct devices that could legitimately answer for that SID.

**6. Design insight** Anycast SIDs are a good example of a feature where "it works in the happy path even when misconfigured" masks a real correctness bug — normal traffic forwarding via SPF/ECMP hides the fact that the N-flag is wrong, so this misconfiguration can sit undetected in production for a long time until a link/node failure triggers TI-LFA, at which point it silently breaks fast-reroute exactly when you need it most (during a real failure). Treat the N-flag on any anycast prefix as a mandatory design checklist item, not an optional tuning knob, and proactively audit for the `ANYCAST_N_FLAG_SET` syslog warning across the whole domain rather than waiting for a live failure to expose it.

**7. Interview-ready answer** "An Anycast SID is a Prefix-SID configured identically on multiple nodes with the N-flag cleared, so it represents 'any member of this redundant group' rather than one specific device — commonly used to steer SR-TE paths toward a redundant pair as a single segment. Forgetting to clear the N-flag doesn't break normal ECMP forwarding, which is exactly why it's a dangerous, easy-to-miss misconfiguration — it only manifests as a real problem when TI-LFA tries to compute a backup path."

---

## 8. N-flag Semantics and TI-LFA Interaction (Anycast SID Protection Correctness)

**1. Definition** The N-flag (Node-SID flag) inside the Prefix-SID sub-TLV is the signal TI-LFA's backup-path computation relies on to determine whether a given Prefix-SID uniquely and reliably identifies one specific physical router. When set, TI-LFA treats the SID as "this segment = exactly one node, safe to use as an intermediate waypoint in a repair path." When cleared, TI-LFA knows the SID cannot be trusted to represent a single node and must exclude that assumption from its repair-path logic.

**2. Why it exists** TI-LFA computes backup (P/Q-space) repair paths before a failure occurs, exploring the post-convergence topology using the SIDs already advertised in the IGP. Building a repair path sometimes requires chaining segments — e.g., "use this Node-SID to reach a specific remote router, then use that node's Adjacency-SID to force traffic over one specific onward link." This chaining is only mathematically valid if the Node-SID genuinely, unambiguously identifies one physical device, because the second segment (the Adj-SID) belongs to that specific device's local link table — if traffic actually lands on a different node than assumed, that device has no idea what the Adj-SID label even means (its local Adj-SID allocations are unrelated to the other anycast member's).

**3. How it works (failure mechanics, step-by-step)**

- Assume R4 and R6 both answer for anycast prefix 46.46.46.1/32, index 46, and (mistakenly) both still advertise the N-flag set.
- TI-LFA, computing a repair path for some other protected link/node elsewhere in the topology, decides it needs to reach "the node behind SID-46" as an intermediate hop, and then chain on that node's specific Adjacency-SID to force the repair path over one particular onward link.
- Because the N-flag told TI-LFA "SID-46 = one specific node," TI-LFA builds a segment list like `[Node-SID 46, Adj-SID X]`, assuming both segments will be processed by the same physical router.
- **The actual failure**: due to ECMP, the repair-path traffic carrying SID-46 might get load-balanced to R6, while the Adj-SID X in the label stack was only ever locally valid on R4 (Adj-SIDs are link-local and independently allocated per-node — R4 and R6 do not share Adj-SID label values for their respective links). When the packet arrives at R6 still carrying Adj-SID X, R6 has no matching LFIB entry for a label that was really R4's local adjacency label — the label is either meaningless to R6's LFIB (dropped) or, worse, accidentally matches something else entirely in R6's local label space (mis-forwarded to the wrong destination).
- Clearing the N-flag prevents this: TI-LFA sees "SID-46 does not identify a single node" and will never attempt to chain a node-specific Adj-SID onto it as an intermediate waypoint — it correctly restricts itself to using the anycast SID only as a terminal, ECMP-safe destination, never as a pivot point for further segment chaining.

**4. Real-world use case** This exact failure mode is why any SR network using anycast SIDs for redundant pairs (ABRs, DC gateways, peering routers) combined with TI-LFA fast-reroute must treat N-flag hygiene as a hard operational requirement — a common real-world gotcha specifically flagged in CCIE/CCDE-level SR design because the failure is invisible until a real network failure triggers the backup path computation, at which point it manifests as fast-reroute making things worse rather than protecting traffic.

**5. Failure scenario (summary)** Sub-50ms local repair (the entire point of TI-LFA) either fails outright (packet drop on the wrong node) or, in the worst case, silently mis-forwards traffic to an unintended destination during exactly the failure event TI-LFA was supposed to protect against — turning a single-link/node failure into a black-hole or mis-forwarding incident precisely at the moment traffic needed fast protection the most.

**6. Design insight** A canonical CCDE-level lesson in "features that only fail under second-order conditions": the anycast N-flag bug is invisible under steady-state forwarding, invisible under simple ECMP, and only surfaces when a second independent mechanism (TI-LFA) makes an assumption based on incorrect metadata during a failure event. A mature design/verification process for any SR network with TI-LFA + anycast SIDs should include explicit automated auditing (via IGP database scraping or a controller/telemetry pipeline) for any anycast prefix still advertising N-flag set, rather than relying on manual configuration review, precisely because the bug hides during all normal-condition testing and only appears during real failure events — the worst possible time to discover it.

**7. Interview-ready answer** "TI-LFA uses the N-flag to decide whether it's safe to chain a node-specific Adjacency-SID onto a Prefix-SID when building a repair path. If an anycast SID incorrectly keeps the N-flag set, TI-LFA may build a repair path assuming traffic lands on one specific anycast member, then chain that member's local Adj-SID onto it — but ECMP can send the packet to the other anycast member instead, whose LFIB has no matching entry for that Adj-SID, causing drops or mis-forwarding during the exact failure event TI-LFA was meant to protect against. Clearing the N-flag tells TI-LFA never to use the anycast SID as a chaining pivot, only as a safe ECMP-terminal segment."

---

## Quick-Reference Summary Table

|Concept|Key Mechanism|Default State|Risk if Misconfigured|
|---|---|---|---|
|SRGB|ISIS RCap TLV 242 → SR-Capability sub-TLV|16000-23999|Overlap → black-hole; non-uniform → complex SR-TE|
|Label space zones|Static partition of 20-bit label space|Reserved/Static/SRLB/SRGB/Dynamic|SRGB relocation fragments dynamic pool|
|Index vs Label|`label = SRGB base + index`, per-hop swap|Index-based (V/L clear)|Stale SRGB flood → label miss → drop|
|PHP|P-flag clear in Prefix-SID sub-TLV|PHP enabled|Loses EXP bits before final node|
|Explicit-Null|P-flag=1 + E-flag=1|Disabled|Partial support → inconsistent EXP across ECMP|
|ISIS SR TLVs|RCap TLV 242 + Prefix-SID sub-TLV (Type 3)|N/A|Missing sub-TLV → default SRGB assumed elsewhere|
|Anycast SID|Same prefix + index on 2+ nodes, N-flag clear|N-flag set (must manually clear)|ECMP fine; TI-LFA breaks if N-flag left set|
|N-flag + TI-LFA|Governs safe Adj-SID chaining|N=1 (single node)|Repair path drops/mis-forwards during failure|