# Segment Routing: Advanced Concepts (CCDE-Level)

> Covers EPE, BGP-LS, PCE, ODN, SDN, Service Chaining, SR-LDP Interworking, Flex-Algo, BGP SR-TE, Seamless MPLS, and more.

---

## Complete List of All Segment Routing Concepts

| # | Concept | Status |
|---|---------|--------|
| 1 | Segment | Core |
| 2 | SID (Segment Identifier) | Core |
| 3 | SRGB | Core |
| 4 | Node SID / Prefix SID | Core |
| 5 | Adjacency SID | Core |
| 6 | Segment List / Stack | Core |
| 7 | SR-MPLS | Core |
| 8 | SRv6 | Core |
| 9 | SR-TE | Core |
| 10 | TI-LFA | Core |
| 11 | Binding SID | Core |
| 12 | SR Policy | Core |
| 13 | BGP EPE | Advanced |
| 14 | BGP-LS | Advanced |
| 15 | PCE | Advanced |
| 16 | ODN | Advanced |
| 17 | SDN + SR | Advanced |
| 18 | Service Chaining | Advanced |
| 19 | SR-LDP Interworking | Advanced |
| 20 | Flex-Algo | Advanced |
| 21 | BGP SR-TE | Advanced |
| 22 | Seamless MPLS | Advanced |
| 23 | SR Authentication | Advanced |
| 24 | SR Telemetry | Advanced |
| 25 | AS (Automated Steering) | Advanced |
| 26 | uSID (micro-SID) | Advanced |
| 27 | SR Policy Color/Endpoint | Advanced |
| 28 | MSD (Maximum SID Depth) | Advanced |
| 29 | PCEP Protocol | Advanced |
| 30 | SR Multi-Domain | Advanced |

---

## Q16. What is BGP EPE (Egress Peer Engineering)?

**Simple Answer:**
BGP EPE = Control which egress peer traffic goes to (not just which egress router).

**CCDE-Level Answer:**
BGP EPE is Segment Routing for BGP peering — steers traffic to specific BGP peer (not just destination).

| Feature | Traditional | With EPE |
|---------|------------|----------|
| Destination | Egress router (Node SID) | Egress router + specific peer |
| Control | BGP best path only | Explicit peer selection |
| Use case | Reach network | Reach network via specific ISP |

**How it works:**
```
Peer SID = SID for specific BGP peering (e.g., Router A → ISP X)
Segment List: [Node-A, Peer-SID-ISP-X]
```

**Use Cases:**
- **Multi-homing** — send traffic via specific ISP based on cost/SLA
- **Traffic engineering** — optimize egress point selection
- **Cost optimization** — use cheaper transit for certain traffic

**BGP-LS Extensions:**
EPE SIDs are advertised via BGP-LS so controller knows all available peers.

> **CCDE Design Point:** EPE is critical for service providers with multiple ISPs — you can optimize egress selection.

---

## Q17. What is BGP-LS (BGP Link-State)?

**Simple Answer:**
BGP-LS = BGP extension that collects network topology (link-state database) and sends it to SDN controller.

**CCDE-Level Answer:**
BGP-LS is a BGP address family that distributes IGP link-state information (OSPF/IS-IS LSDB) to external entities (SDN controller, PCE).

**How it works:**
```
IGP Domain (OSPF/IS-IS) → BGP-LS → Route Reflector → SDN Controller
       (collects LSDB)              (distributes)
```

**What BGP-LS advertises:**

| Information | Description |
|-------------|-------------|
| Nodes | Router IDs, loopbacks, capabilities |
| Links | Interfaces, IP addresses, bandwidth |
| Prefixes | IP prefixes, Segment Routing SIDs |
| SR Capabilities | SRGB, SID indices, Flex-Algo |
| EPE SIDs | BGP peer segments (for Egress Peer Engineering) |

**Use Cases:**
- SDN Controller — builds network-wide topology
- PCE — computes paths based on full topology
- Multi-domain — connect multiple IGP domains via BGP-LS
- Traffic Engineering — controller knows link bandwidth, latency, etc.

> **CCDE Design Point:** BGP-LS is mandatory for Carrier SDN — controller needs network-wide visibility to compute optimal paths.

---

## Q18. What is PCE (Path Computation Element)?

**Simple Answer:**
PCE = Centralized path computation engine that calculates optimal paths for SR-TE policies.

**CCDE-Level Answer:**
PCE is a network component (controller) that:
1. Receives network topology via BGP-LS
2. Computes optimal paths based on constraints (bandwidth, latency, cost)
3. Programs SR Policies into headend routers via PCEP protocol

**How it works:**
```
PCEP (Path Computation Element Protocol)
Headend ←→ PCE (Controller)
           (programs SR Policy)
```

**PCE Functions:**

| Function | Description |
|----------|-------------|
| Stateful PCE | Maintains state of all computed paths |
| Stateless PCE | Computes paths on request (no state) |
| Active PCE | Programs paths into network (SR-TE) |
| Passive PCE | Only computes, doesn't program |

**Benefits:**
- **Global optimization** — controller sees entire network (not just local view)
- **Constraint-based** — compute paths based on bandwidth, latency, admin policy
- **Dynamic** — recompute paths when network changes

> **CCDE Design Point:** PCE is centralized control plane — computes paths, headend forwards.

---

## Q19. What is ODN (On-Demand Next-hop)?

**Simple Answer:**
ODN = SDN controller automatically creates SR Policies when it sees new routes.

**CCDE-Level Answer:**
ODN is an SR Policy automation mechanism where:
- Headend router signals: "I need a path to prefix X"
- Controller automatically computes path and programs SR Policy
- No manual configuration — policy created on-demand

**How it works:**
```
Headend: "I need path to 10.0.0.5/32 (ODN template matches)"
Controller: Computes path → Programs SR Policy → Headend steers traffic
```

**ODN vs Automated Steering (AS):**

| Feature | ODN | AS |
|---------|-----|-----|
| Trigger | New route seen at headend | Route mapping matches |
| Template | ODN template (prefix-based) | Mapping (color-endpoint) |
| Automation | Automatic for new routes | Manual mapping config |

> **CCDE Design Point:** ODN is for dynamic environments where prefixes change frequently — controller handles automation.

---

## Q20. What is SDN (Software-Defined Networking) with Segment Routing?

**Simple Answer:**
SDN + SR = Centralized controller computes paths, network forwards using Segment Routing.

**CCDE-Level Answer:**

| Layer | Component | Protocol |
|-------|-----------|----------|
| Northbound | Controller → Application | REST API, NETCONF |
| Control | Controller → Router | PCEP, BGP SR-TE |
| Southbound | Router → Forwarding | SR-MPLS / SRv6 |
| Visibility | Router → Controller | BGP-LS |

**Controller Architecture:**
```
Application Layer (Northbound API)
         ↓
Controller (PCE, BGP SR-TE, ODN)
         ↓
Router (Headend) → SR Policy → Forwarding (SR-MPLS/SRv6)
         ↑
    BGP-LS (Topology)
```

**Benefits:**
- **Centralized optimization** — global view, best paths
- **Automation** — API-driven, no manual config
- **Programmability** — NETCONF/YANG, REST API
- **Dynamic** — controller reacts to network changes

> **CCDE Design Point:** SDN + SR is modern architecture — controller computes, IGP distributes, network forwards.

---

## Q21. What is Service Chaining with Segment Routing?

**Simple Answer:**
Service Chaining = Force traffic through services (firewall, load balancer, NAT) in specific order.

**CCDE-Level Answer:**

**How it works:**
```
Segment List: [Node-A, Service-Firewall, Service-LoadBalancer, Node-B]
              (Go to A)   (Firewall)      (Load Balancer)     (Go to B)
```

**Service SID Types:**

| Service SID | Purpose |
|------------|---------|
| Service Node SID | Reach service node (firewall) |
| Service Index SID | Specific service function |
| Binding SID | Aggregate multiple services |

**Use Cases:**
- **Security** — firewall → IDS → clean traffic
- **Optimization** — WAN optimizer → compressor
- **Compliance** — traffic must go through inspection

> **CCDE Design Point:** Service chaining is SR strength — no additional protocol needed, just encode service list in segment stack.

---

## Q22. What is SR-LDP Interworking?

**Simple Answer:**
SR-LDP Interworking = SR and LDP coexist during migration (SR ↔ LDP at boundary).

**CCDE-Level Answer:**

**How it works:**
```
SR Domain ←→ Interworking Gateway ←→ LDP Domain
 (SR SIDs)    (maps label ↔ SID)     (LDP labels)
```

**Migration Phases:**

| Phase | Action |
|-------|--------|
| 1 | Enable IGP extensions for SR (OSPF/IS-IS) |
| 2 | Enable SR on edge routers first |
| 3 | Core runs SR + LDP together |
| 4 | Gradually migrate core to SR-only |
| 5 | Remove LDP when all migrated |

**Benefits:**
- No service disruption — gradual migration
- Coexistence — SR and LDP work together
- Rollback — can revert if issues

> **CCDE Design Point:** SR-LDP interworking is essential for migration — plan boundary points carefully.

---

## Q23. What is Flex-Algo (Flexible Algorithm)?

**Simple Answer:**
Flex-Algo = Define custom IGP shortest path (not just metric) — e.g., lowest latency, highest bandwidth.

**CCDE-Level Answer:**
Flex-Algo allows multiple IGP topologies in same network, each with different computation:

```
Algorithm 0   (default):  Shortest path by IGP metric
Algorithm 128:            Shortest path by delay (lowest latency)
Algorithm 129:            Shortest path by bandwidth (highest capacity)
Algorithm 130:            Shortest path by admin group (exclude certain links)
```

**Configuration example:**
```
Router A:
  Flex-Algo 128: Metric = delay,     Exclude-admin = red-links
  Flex-Algo 129: Metric = bandwidth, Include-admin = green-links
```

**Use Cases:**
- **Low-latency** — real-time traffic (video, VoIP)
- **High-bandwidth** — bulk data transfer
- **Service isolation** — different algorithms for different services
- **Network slicing** — separate topologies per tenant/service

> **CCDE Design Point:** Flex-Algo is native network slicing — no VRFs needed, just different algorithms.

---

## Q24. What is BGP SR-TE (BGP Segment Routing Traffic Engineering)?

**Simple Answer:**
BGP SR-TE = Use BGP (not PCEP) to distribute SR Policies from controller to headend.

**CCDE-Level Answer:**

| Feature | PCEP | BGP SR-TE |
|---------|------|-----------|
| Protocol | PCEP | BGP (new address family) |
| Distribution | Controller → Headend | BGP Route Reflector → Clients |
| Scalability | Controller-centric | Route reflection (proven) |
| Robustness | Single controller | Multiple RR (more robust) |
| Use case | Smaller networks | Large-scale carrier networks |

**How it works:**
```
Controller → BGP SR-TE NLRI → Route Reflector → Headend Routers
                               (programs SR Policy)
```

> **CCDE Design Point:** BGP SR-TE is more robust than PCEP — uses proven BGP infrastructure (Route Reflection).

---

## Q25. What is Seamless MPLS?

**Simple Answer:**
Seamless MPLS = Connect multiple IGP domains with BGP (no full-mesh iBGP).

**CCDE-Level Answer:**

**How it works:**
```
IGP Area 1 ←→ BGP ←→ IGP Area 2 ←→ BGP ←→ IGP Area 3
 (Node SID)   (BGP Prefix SID)    (Node SID)
```

**Key Components:**

| Component | Purpose |
|-----------|---------|
| BGP Prefix SID | Advertise prefix + SID across domains |
| BGP-LU | BGP labeled unicast (distributes labels) |
| ABR | Area Border Router (connects IGP areas) |

**Benefits:**
- No full-mesh iBGP — scales better
- End-to-end LSP — across multiple IGP domains
- SR integrated — BGP Prefix SID for end-to-end path

> **CCDE Design Point:** Seamless MPLS is carrier architecture — connects multiple IGP domains with BGP + SR.

---

## Q26. What is SR Authentication?

**Simple Answer:**
SR Authentication = Secure segment routing (prevent unauthorized SIDs).

**CCDE-Level Answer:**
SR Authentication protects SR control plane:
- **IGP authentication** — OSPF/IS-IS MD5/SHA for SID distribution
- **BGP authentication** — MD5/SHA for BGP-LS, BGP SR-TE
- **PCEP authentication** — secure PCEP session
- **SRv6 authentication** — IPsec (AH/ESP headers)

> **CCDE Design Point:** Security is critical for SR — compromised SID can redirect traffic.

---

## Q27. What is SR Telemetry (sFlow, gNMI, IPFIX)?

**Simple Answer:**
SR Telemetry = Monitor SR paths (performance, utilization, failures).

**CCDE-Level Answer:**
SR Telemetry provides real-time visibility into SR paths:
- **sFlow/gNMI** — collect statistics from routers
- **IPFIX** — flow monitoring (SR Policy utilization)
- **In-band OAM** — monitor path performance (latency, loss)
- **SR-TE Statistics** — per-Policy utilization, packet counts

**Use Cases:**
- **Performance monitoring** — latency, jitter, loss
- **Capacity planning** — link utilization
- **Troubleshooting** — verify SR Policy forwarding

> **CCDE Design Point:** Telemetry is essential for SR-TE — controller needs real-time data for optimization.

---

## CCDE Interview Answer Summary

> **Interviewer:** "What advanced Segment Routing concepts are critical for carrier networks?"

**Your Answer:**

"Beyond core concepts (SID, Node/Adj SID, SR-MPLS/SRv6), critical advanced concepts include:

1. **BGP EPE** — Control egress peer selection (not just router)
2. **BGP-LS** — Distribute topology to SDN controller (network-wide visibility)
3. **PCE** — Centralized path computation (global optimization)
4. **ODN** — Automatic SR Policy creation for new prefixes
5. **SDN + SR** — Centralized control + distributed forwarding
6. **Service Chaining** — Steer traffic through services (firewall, load balancer)
7. **SR-LDP Interworking** — Gradual migration from LDP to SR
8. **Flex-Algo** — Custom IGP algorithms (low-latency, high-bandwidth)
9. **BGP SR-TE** — BGP-based SR Policy distribution (more robust than PCEP)
10. **Seamless MPLS** — Multi-domain architecture with BGP Prefix SID

These enable Carrier SDN, automation, network slicing, and optimized traffic engineering."
