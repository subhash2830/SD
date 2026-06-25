# Segment Routing: All Key Concepts (CCDE-Level)

---

## Q1. What is Segment Routing (SR)?

**Simple Answer:**
Segment Routing is a source-based routing technique where the ingress router decides the entire path and encodes it in the packet. No per-flow state in the core.

**CCDE-Level Answer:**
Segment Routing is a source-routing architecture that:
- **Encodes path** as an ordered list of segments (instructions) in packet header
- **No signaling protocol** needed (eliminates LDP/RSVP-TE)
- **Stateless core** — transit routers only process top segment
- **IGP-distributed** — OSPF/IS-IS advertises segments (no separate protocol)
- **Two data planes:** SR-MPLS (MPLS label stack) or SRv6 (IPv6 + SRH)

| Benefit | Why It Matters |
|---------|---------------|
| Simplify | Removes LDP/RSVP-TE, fewer protocols |
| Scalable | Per-flow state only at ingress (not core) |
| Robust | Sub-50ms failover with TI-LFA |
| Efficient | Up to 80% capacity utilization with dynamic rerouting |
| Seamless | Coexists with LDP, simple software upgrade |

> **CCDE Design Point:** SR is fundamental for modern networks — it's the foundation for TE, SDN, network slicing, and automation.

---

## Q2. What is a Segment?

**Simple Answer:**
A Segment = A single instruction telling the packet where to go or what to do.

**CCDE-Level Answer:**
A Segment is the basic unit of a network path — an instruction that can represent:

| Segment Type | What It Represents |
|-------------|-------------------|
| Node | A router/switch (e.g., "go to Router A") |
| Link | A specific connection (e.g., "use link A→B") |
| Service | A function (e.g., firewall, load balancer) |
| Policy | A traffic engineering policy |

**Key Properties:**
- **Ordered** — segments are in a list (first to last)
- **Encoded in packet** — MPLS label stack or IPv6 SRH
- **Source-routed** — ingress decides order, not each hop
- **No additional protocol** — IGP distributes segments

> **CCDE Design Point:** Think of segments as "building blocks" — combine them to create any path (simple to complex).

---

## Q3. What is a SID (Segment Identifier)?

**Simple Answer:**
SID = The ID of a segment — how you reference it in the packet.

**CCDE-Level Answer:**
SID is a unique identifier for a segment. Its format depends on data plane:

| Data Plane | SID Format | Size |
|-----------|-----------|------|
| SR-MPLS | MPLS label | 20-bit |
| SRv6 | IPv6 address | 128-bit |
| SRv6 uSID | Compressed | 32-bit |

SID is indexed from SRGB:
```
Label = SRGB_base + SID_index
Example: SRGB = [16000-23999], Index = 100 → Label = 16100
```

**Key Properties:**
- Global or local scope (depends on SID type)
- IGP-distributed — flooded via OSPF/IS-IS TLV
- Unique — no duplicates unless Anycast

---

## Q4. What is SRGB (Segment Routing Global Block)?

**Simple Answer:**
SRGB = The label range reserved for Segment Routing on a router.

**CCDE-Level Answer:**

| Parameter | Value |
|-----------|-------|
| Default (Cisco) | [16000-23999] (8000 labels) |
| Scope | Global (same across SR domain) |
| Purpose | Where Node SIDs / Prefix SIDs are allocated from |
| Formula | Label = SRGB_base + SID_index |

**Example:**
```
SRGB = [16000-23999]
Node SID index = 101
Actual MPLS label = 16000 + 101 = 16101
```

> **CCDE Design Points:**
> - Keep SRGB consistent across all routers (easier troubleshooting)
> - Ensure SID indices don't overlap with static/other labels
> - Plan for growth (8000 SIDs = 8000 routers)

---

## Q5. What is a Node SID?

**Simple Answer:**
Node SID = SID for a router's loopback. Tells network: "Go to this router via shortest path."

**CCDE-Level Answer:**

| Property | Value |
|----------|-------|
| Scope | Global (unique across SR domain) |
| Prefix | Router loopback (10.0.0.1/32) |
| Path | IGP shortest path (ECMP-aware) |
| N-flag | Set (Node) |
| P-flag | Set (Persistent) |
| Uniqueness | Must be unique per router |

**Forwarding:**
```
Node SID 16101 → All routers forward via IGP shortest path to that node
```

> **CCDE Design Point:** Use Node SID for basic connectivity — it's ECMP-aware and stateless.

---

## Q6. What is a Prefix SID?

**Simple Answer:**
Prefix SID = SID for any IP prefix (not just loopback). Node SID is a type of Prefix SID.

**CCDE-Level Answer:**

| Type | Prefix | Use Case |
|------|--------|----------|
| Node SID | Loopback (10.0.0.1/32) | Reach specific router |
| Anycast SID | Shared prefix (192.0.2.1/32) | HA (multiple routers, same IP) |
| Regular Prefix SID | Subnet (10.1.1.0/24) | Reach network segment |

**Key Difference from Node SID:**
- Node SID = always single router (N-flag set)
- Prefix SID = can be shared (Anycast) or service prefix

---

## Q7. What is an Adjacency SID (Adj SID)?

**Simple Answer:**
Adj SID = SID for a specific link. Tells network: "Go out this exact interface."

**CCDE-Level Answer:**

| Property | Value |
|----------|-------|
| Scope | Local (only meaningful to advertising router) |
| Path | Strict — must use this link (no ECMP) |
| L-flag | Set (Local) |
| Uniqueness | Unique per interface |

**Use Cases:**
- Traffic Engineering — force specific link
- Fast Reroute (FRR) — backup path
- Link-specific policy — different treatment per link

**Forwarding:**
```
Adj SID 17001 (link A→B) → Must use that exact interface (no load-balancing)
```

> **CCDE Design Point:** Use Node SID for normal traffic (utilizes all paths), Adj SID for TE/backup.

---

## Q8. What is Segment List / Segment Stack?

**Simple Answer:**
Segment List = The ordered list of SIDs in the packet — the complete path.

**CCDE-Level Answer:**

```
Segment List = [SID_A, SID_B, SID_C]
               (Go to A) (Go to B) (Go to C)

Packet Header (SR-MPLS):
[Label 16101 (A), Label 16102 (B), Label 16103 (C)]
```

**Forwarding:**
1. Ingress pushes segment stack
2. Each router processes top SID
3. When top SID is processed, advance to next
4. Egress removes stack, forwards original packet

> **CCDE Design Point:** You can stack segments to create precise paths (e.g., A → specific link → B → service → C).

---

## Q9. What is SR-TE (Segment Routing Traffic Engineering)?

**Simple Answer:**
SR-TE = Using Segment Routing to steer traffic along explicit paths (not just IGP shortest path).

**CCDE-Level Answer:**

**How it works:**
1. Controller (PCE/SDN) computes path (e.g., low-latency, high-bandwidth)
2. Ingress encodes path as segment list: `[Node-A, Adj-A→B, Node-C]`
3. Packet follows exact path regardless of IGP shortest path
4. Stateless — no per-flow state in core

| Feature | RSVP-TE | SR-TE |
|---------|---------|-------|
| Signaling | RSVP protocol | No signaling |
| State | Per-flow in core | Only at ingress |
| Complexity | High | Low |
| Scalability | Limited | High |

> **CCDE Design Point:** SR-TE is centralized control + distributed forwarding — controller computes, network forwards.

---

## Q10. What is TI-LFA (Topology-Independent Loop-Free Alternate)?

**Simple Answer:**
TI-LFA = Fast Reroute (FRR) that provides sub-50ms failover for any topology (no loops).

**CCDE-Level Answer:**
TI-LFA is SR's Fast Reroute mechanism that:
- Pre-computes backup path before failure
- Encodes backup path as segment list
- Switches instantly when failure detected (no IGP convergence wait)
- Loop-free — guaranteed by segment encoding

**How it works:**
```
Primary path: A → B → C → D
Backup path:  A → E → F → D

If B fails:
  A instantly switches to backup segment list [E, F, D]
  Failover = <50ms (no IGP recalculation needed)
```

> **CCDE Design Point:** TI-LFA is built-in with SR — no extra protocol needed (unlike RSVP-TE FRR).

---

## Q11. What is SR-MPLS?

**Simple Answer:**
SR-MPLS = Segment Routing using MPLS label stack as the data plane.

**CCDE-Level Answer:**
- SID = 20-bit MPLS label
- Segment list = Label stack
- IGP distributes labels (no LDP/RSVP needed)
- Backward compatible — coexists with LDP

**How forwarding works:**
```
Packet: [Label 16101 (Node A), Label 16102 (Node B)]
         └─ Top label processed first
```

> **CCDE Design Point:** Use SR-MPLS if you have existing MPLS infrastructure — migrate from LDP gradually.

---

## Q12. What is SRv6?

**Simple Answer:**
SRv6 = Segment Routing using IPv6 header + SRH extension as the data plane.

**CCDE-Level Answer:**
- SID = 128-bit IPv6 address
- Segment list = IPv6 Routing Header Type 4 (SRH)
- Native IPv6 — no MPLS needed
- Network programming — SID can encode functions (not just forwarding)

**Header structure:**
```
IPv6 Header | SRH (Segment Routing Header) | Payload
             └─ Contains segment list [A, B, C]
```

> **CCDE Design Point:** Use SRv6 for new IPv6-only networks or long-term future-proofing.

---

## Q13. What is Binding SID?

**Simple Answer:**
Binding SID = A SID that represents a segment list (advertised as single SID).

**CCDE-Level Answer:**

| Use Case | Description |
|----------|-------------|
| SR Policy | Advertise segment list as single SID |
| Hierarchy | Nest SR policies (SID of SIDs) |
| Scaling | Reduce label stack depth (MSD limit) |

**Example:**
```
Segment List: [A, B, C, D, E] → 5 labels
Binding SID:  16500            → 1 label (represents all 5)

Stack: [Binding_SID, X, Y] instead of [A, B, C, D, E, X, Y]
```

> **CCDE Design Point:** Use Binding SID to reduce stack depth when MSD (Maximum SID Depth) is limited.

---

## Q14. What is SR Policy?

**Simple Answer:**
SR Policy = A configured path (segment list) that traffic is steered into.

**CCDE-Level Answer:**

| Component | Description |
|-----------|-------------|
| Headend | Ingress router that steers traffic |
| Tailend | Egress router (destination) |
| Segment List | Explicit path (encoded as SIDs) |
| Binding SID | Optional — advertised as single SID |

**Steering methods:**
- BGP — BGP next-hop points to SR Policy
- Static route — Next-hop = SR Policy
- Policy-based routing — Match criteria → SR Policy

> **CCDE Design Point:** SR Policy is centralized control — controller computes path, headend steers traffic.

---

## Q15. What are the key benefits of Segment Routing?

| Benefit | Technical Reason |
|---------|-----------------|
| Simplify network | Removes LDP/RSVP-TE, fewer protocols |
| More robust | Sub-50ms failover with TI-LFA |
| Better utilization | Up to 80% capacity with dynamic rerouting |
| Release innovation | Enables low-latency, network slicing, SDN |
| Customer satisfaction | Efficient structure = better end-user experience |

**Additional Benefits:**
- **BGP-free core** — core only needs IGP (not full BGP)
- **Multi-domain paths** — end-to-end across multiple domains
- **Multi-topology** — separate topologies for different services
- **Automation-ready** — controller-driven, API-friendly

---

## Quick Reference: All Key Concepts

| Concept | What It Is | Scope | Data Plane | Use Case |
|---------|-----------|-------|-----------|----------|
| Segment | Instruction (where to go) | N/A | N/A | Building block |
| SID | ID of segment | Global/Local | Label/IPv6 | Reference segment |
| SRGB | Label range for SR | Global | MPLS | Where SIDs indexed from |
| Node SID | SID for router | Global | Both | Basic connectivity |
| Prefix SID | SID for prefix | Global | Both | TE, HA, services |
| Adj SID | SID for link | Local | Both | Traffic Engineering |
| Segment List | Ordered SIDs | N/A | Both | Complete path |
| SR-MPLS | SR with MPLS | Global | MPLS label | Existing MPLS network |
| SRv6 | SR with IPv6 | Global | IPv6 + SRH | Future IPv6 network |
| SR-TE | Explicit path steering | N/A | Both | Traffic Engineering |
| TI-LFA | Fast Reroute | N/A | Both | Sub-50ms failover |
| Binding SID | SID for segment list | Global | Both | Reduce stack depth |
| SR Policy | Configured path | N/A | Both | Centralized control |

---

## CCDE Interview Answer Summary

> **Interviewer:** "What are all key concepts in Segment Routing?"

**Your Answer:**

"Segment Routing is a source-routing architecture where the ingress router decides the entire path and encodes it as an ordered list of segments (instructions) in the packet. Key concepts:

1. **Segment** = Instruction (go to node, use link, apply service)
2. **SID** = ID of segment (20-bit label in SR-MPLS, 128-bit IPv6 in SRv6)
3. **SRGB** = Label range (e.g., 16000-23999) where SIDs are indexed from
4. **Node SID** = Global SID for router loopback (ECMP-aware, shortest path)
5. **Prefix SID** = SID for any prefix (Node SID is a type of Prefix SID)
6. **Adj SID** = Local SID for specific link (used for TE, no ECMP)
7. **Segment List/Stack** = Ordered list of SIDs (complete path in packet)
8. **SR-MPLS** = SR with MPLS label stack (simplifies LDP networks)
9. **SRv6** = SR with IPv6 + SRH (future-proof, network programming)
10. **SR-TE** = Traffic Engineering without RSVP-TE (controller computes path)
11. **TI-LFA** = Sub-50ms Fast Reroute (loop-free, topology-independent)
12. **Binding SID** = SID representing segment list (reduces stack depth)
13. **SR Policy** = Configured path between headend and tailend

Benefits: Simplifies network (no LDP/RSVP), scalable (stateless core), robust (<50ms failover), efficient (80% utilization), and seamless (coexists with LDP). It's fundamental for SDN, automation, and future networks."
