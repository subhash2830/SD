# Intra-AS MPLS L3VPN — CCDE Notes

## 1. Subtopics

### 1.1 VRF (Virtual Routing and Forwarding)
**What:** A VRF is a per-customer routing/forwarding instance on a PE router — separate RIB, separate CEF/FIB table, and (usually) separate interface set — allowing overlapping customer address space on shared PE hardware.

**Why it matters (CCDE lens):** VRF is the isolation primitive that makes multi-tenant WAN economically viable. A CCDE-level design question is never "what is a VRF" but "how do you scale VRF count and RT/RD policy across hundreds of PEs without an operational meltdown" — i.e., naming conventions, automation (RD/RT allocation schemes), and blast-radius containment when a VRF config error occurs.

**Real-world example:** A service provider hosting 3 enterprise customers on one PE, each with a 10.0.0.0/8 internal range — VRFs prevent route leaking/collision between them despite identical prefixes.

**CLI:**
```
vrf definition CUST_A
 rd 65000:100
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:100
 exit-address-family
!
interface Gi0/0/0
 vrf forwarding CUST_A
 ip address 10.1.1.1 255.255.255.252
```

### 1.2 Route Distinguisher (RD)
**What:** An 8-byte value prepended to an IPv4 prefix to make it unique in the VPNv4 address family (RD:prefix), solving the overlapping-address problem across VRFs.

**Why it matters (CCDE lens):** RD is purely a disambiguator, NOT a policy tool — a common design mistake is conflating RD with RT. CCDE scenarios test whether you know RD uniqueness only needs to be per-VRF-per-PE (not globally unique), and that using the same RD on two PEs for the same customer VRF causes BGP to treat routes as the same path (best-path suppresses the "backup" one) — this is actually a deliberate design trick for certain fast-convergence/multi-homing patterns, not always a bug.

**Real-world example:** Two PEs serving the same customer site in an active/standby DC pair intentionally share one RD so BGP best-path (not multipath) picks a single active path automatically.

**CLI:**
```
vrf definition CUST_A
 rd 65000:100
```

### 1.3 Route Target (RT) — Import/Export Policy
**What:** BGP extended community attached to VPNv4 routes controlling which VRFs import which routes — the actual topology-building mechanism (hub-spoke, full mesh, extranet).

**Why it matters (CCDE lens):** RT design IS the network topology. This is the single highest-value CCDE topic in L3VPN — full mesh (same RT import/export everywhere) vs hub-and-spoke (spokes export RT-S/import RT-H, hub exports RT-H/imports RT-S) vs extranet (shared services VRF exports a common RT that multiple customer VRFs import). Get RT design wrong and you either leak routes between customers (security/compliance incident) or fail to establish required connectivity (outage).

**Real-world example:** A shared-services extranet: a "Internet-VRF" exports RT 65000:999; every customer VRF imports RT 65000:999 to reach a shared internet breakout without those customer VRFs seeing each other.

**CLI:**
```
vrf definition CUST_SPOKE
 rd 65000:200
 address-family ipv4
  route-target export 65000:200
  route-target import 65000:100  ! import Hub's RT
```

### 1.4 MP-BGP VPNv4 Address Family (PE-PE control plane)
**What:** MP-iBGP session between PEs (typically via RR) carrying the VPNv4 AFI/SAFI (RD+prefix, RT extcommunity, VPN label, next-hop) to distribute customer routes across the core without core routers ever seeing customer prefixes.

**Why it matters (CCDE lens):** This is the scalability layer — RRs decouple the O(n²) iBGP full mesh problem. CCDE design must address RR placement (regional hierarchy, redundancy, RR-only vs RR+ABR combined), and next-hop-self behavior (PE must set itself as BGP next-hop for VPNv4 routes, or the label stack breaks since P routers don't run BGP/VPNv4).

**Real-world example:** 200 PE routers use 4 regional RRs (2 pairs) instead of a 200-node iBGP mesh, cutting control-plane scaling from ~20,000 sessions to ~800.

**CLI:**
```
router bgp 65000
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.1 update-source Loopback0
 address-family vpnv4
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.1 send-community extended
  neighbor 10.0.0.1 next-hop-self
```

### 1.5 Label Allocation — Per-VRF vs Per-Prefix
**What:** Each VPNv4 route carries a VPN label. Per-VRF mode assigns one label per VRF (all prefixes in that VRF share it, egress PE does a VRF lookup); per-prefix mode assigns a unique label per route (faster egress lookup, no VRF table walk, but higher label consumption).

**Why it matters (CCDE lens):** This is a scale-vs-performance tradeoff CCDE explicitly tests: per-prefix labeling is required for line-rate forwarding on high-prefix-count VRFs (avoids a second lookup) but consumes far more label space and control-plane churn on PE reload/flap. Design must estimate label budget against platform TCAM/label-space limits.

**Real-world example:** An internet-facing VRF carrying a full BGP table (~900k routes) would exhaust label space quickly under per-prefix allocation on older platforms — per-VRF is the pragmatic choice there; a small 50-prefix customer VRF can safely use per-prefix for lookup speed.

**CLI (IOS-XR):**
```
router bgp 65000
 vrf CUST_A
  address-family ipv4 unicast
   label mode per-vrf
```

### 1.6 PE-CE Routing Protocols
**What:** The protocol run between PE and CE to exchange customer routes into/out of the VRF — static, eBGP, OSPF (with sham-link for intra-area backdoor), or EIGRP (with SoO to prevent loops).

**Why it matters (CCDE lens):** eBGP is the CCDE "default good answer" for PE-CE (fast convergence tools, native loop prevention via AS-path, policy flexibility) but real customers often insist on OSPF for operational familiarity — this forces the sham-link discussion: without it, OSPF intra-area routes crossing the MPLS core become inter-area (Type 3 LSA) via BGP redistribution, breaking area 0 backdoor/summarization assumptions and causing suboptimal routing if a backdoor link exists between sites.

**Real-world example:** A customer with a backdoor leased line directly between two sites plus MPLS L3VPN to both — without a sham-link, OSPF prefers the (worse-metric but "intra-area weight" mismatched) backdoor link because the VPN path appears as external/inter-area.

**CLI:**
```
router ospf 10 vrf CUST_A
 area 0 sham-link 10.0.0.1 10.0.0.2 cost 5
```

### 1.7 Loop Prevention — Site of Origin (SoO)
**What:** An extended community tagging routes with their originating site, used to prevent readvertisement of a route back into the site it came from — critical for multi-homed CE sites and EIGRP PE-CE.

**Why it matters (CCDE lens):** Without SoO, a dual-homed customer site can create a routing loop: PE1 learns a route from CE via eBGP, sends it across the core to PE2, PE2 readvertises it back to the same site via a second CE link. CCDE scenarios on multi-homed sites almost always require SoO as part of the answer, not just AS-path loop prevention (which doesn't work the same way for EIGRP/OSPF PE-CE).

**Real-world example:** A customer dual-homed to PE1 and PE2 with EIGRP — SoO stops EIGRP routes learned from PE1 from being readvertised out PE2 back into the customer's own network.

**CLI:**
```
route-map SOO-TAG permit 10
 set extcommunity soo 65000:500
```

### 1.8 VRF-Lite vs True MPLS L3VPN
**What:** VRF-Lite is VRF without MPLS/MP-BGP — routes exchanged hop-by-hop via a regular IGP/BGP session per VRF-tagged sub-interface (no label distribution, no RD/RT-based control plane). True L3VPN uses MPLS + MP-BGP for scale.

**Why it matters (CCDE lens):** CCDE tests when VRF-Lite is the *correct* (not inferior) choice — small campus/DC multi-tenant segmentation without a full MPLS core, or as a stitching mechanism at an inter-AS/inter-provider boundary (Option A). Choosing full MPLS L3VPN where VRF-Lite would suffice adds unnecessary operational complexity (label distribution, MP-BGP, RR infra) for no scale benefit.

**Real-world example:** A campus network segmenting guest/corp/IoT traffic across 3 core switches uses VRF-Lite with sub-interfaces; no MPLS core exists or is needed.

**CLI:**
```
interface Gi0/0/1.100
 encapsulation dot1Q 100
 vrf forwarding GUEST
 ip address 10.100.0.1 255.255.255.0
```

---

## 2. Interview Q&A

**Q1: Why is RD not sufficient on its own to control VPN topology — what does RT actually do that RD doesn't?**
A: RD only disambiguates overlapping prefixes into unique VPNv4 routes; it plays no role in import decisions. RT (an extended community) is what a receiving PE's VRF import policy matches against to decide whether to install a route — RT is the actual topology-building mechanism (full mesh, hub-spoke, extranet).

**Q2: A customer has OSPF running PE-CE and also has a backdoor link between two sites. What breaks, and how do you fix it?**
A: Routes crossing the MPLS core get redistributed into OSPF as external (or inter-area) routes, losing intra-area preference, so OSPF may prefer a worse backdoor path solely due to route-type metric handling. Fix: configure a sham-link between the PEs so the core-crossing route appears as an intra-area OSPF path.

**Q3: When would you choose per-prefix over per-VRF label allocation, and what's the cost?**
A: Choose per-prefix when egress PE forwarding performance matters and a second VRF-table lookup is unacceptable (high-throughput VRFs). Cost: significantly higher label consumption and control-plane/label-table churn, which can be a problem on platforms with limited label space or VRFs carrying very large route counts (e.g., full BGP table).

**Q4: How do route reflectors solve the iBGP full-mesh problem in an L3VPN core, and what's the design risk of doing it wrong?**
A: RRs let PEs peer only with RRs instead of every other PE, turning O(n²) sessions into O(n). Risk: single RR (or under-provisioned RR pair) becomes a control-plane SPOF/bottleneck; also RR path hiding (best-path-only reflection) can suppress valid alternate paths needed for fast convergence unless add-path or diverse-path is used.

**Q5: What does Site of Origin solve that AS-path loop prevention in eBGP does not?**
A: AS-path prevention only works within an eBGP context where the AS-path grows and gets rejected if a router sees its own AS. For PE-CE protocols like EIGRP or OSPF (no AS-path concept), or for multi-homed sites where the "own AS" check doesn't apply cleanly, SoO explicitly tags and blocks re-advertisement of a route back into its originating site, closing loop vectors AS-path can't cover.

**Q6: Design scenario — a customer wants a hub site to reach all spokes but spokes should not reach each other. How do you build this with RT alone?**
A: Hub VRF exports RT-H and imports RT-S; every spoke VRF exports RT-S and imports RT-H. Spokes never import each other's export RT, so spoke-to-spoke routes are never installed, while the hub sees (and can route between) all spokes.

**Q7: Why must a PE set next-hop-self for VPNv4 routes advertised to other PEs?**
A: P routers in the core don't run BGP and have no VPNv4 knowledge — they only forward on the LDP/segment-routing label for the PE's loopback. If the PE didn't rewrite next-hop to itself, the advertised next-hop could be an address the P-core can't resolve to a valid LSP, breaking label switching end-to-end.

**Q8: When is VRF-Lite a *better* design choice than full MPLS L3VPN, not just a cheaper one?**
A: When there's no need for provider-wide any-to-any scale, no existing/justifiable MPLS core, and the segmentation is confined to a small number of directly connected boundary devices (e.g., campus/DC tenant separation, or an Option-A inter-AS stitching point) — VRF-Lite avoids the operational overhead of MP-BGP, RD/RT policy, and label distribution for a problem that per-hop VRF sub-interfaces solve just as well at that scale.

---

## 3. Memory Map

```
Intra-AS L3VPN
├── Data Plane Isolation
│   └── VRF (per-customer RIB/FIB)
│        └── enables → overlapping customer address space
├── Control Plane Uniqueness
│   └── RD (RD:prefix → unique VPNv4 route)
│        ├── does NOT control policy
│        └── same RD across PEs → intentional single-path trick
├── Control Plane Policy / Topology
│   └── RT (import/export extended community)
│        ├── Full Mesh  (same RT both ways, everywhere)
│        ├── Hub-Spoke  (asymmetric RT-H / RT-S)
│        └── Extranet   (shared-services RT imported by many VRFs)
├── PE-PE Distribution
│   └── MP-iBGP VPNv4 AFI/SAFI
│        ├── Route Reflectors (solve O(n²) mesh)
│        └── next-hop-self (required — P routers have no BGP/VPNv4)
├── Forwarding / Label Plane
│   └── VPN Label Allocation
│        ├── Per-VRF   (lookup at egress, label-space efficient)
│        └── Per-Prefix (fast egress lookup, label-space heavy)
├── PE-CE Exchange
│   ├── Static / eBGP (CCDE default — fast convergence, AS-path loop prevention)
│   ├── OSPF PE-CE
│   │    └── needs Sham-link when backdoor link exists (intra-area preservation)
│   └── EIGRP PE-CE
│        └── needs SoO (no native inter-site loop prevention)
├── Multi-Homing Loop Prevention
│   └── Site of Origin (SoO) — protocol-agnostic, blocks re-advertisement into origin site
└── Alternative Architecture
    └── VRF-Lite (no MPLS/MP-BGP) — right choice at small scale / inter-AS Option A stitching
```

---

## 4. CLI Cheat Sheet

| Task | Platform | Command |
|---|---|---|
| Define VRF with RD | IOS/IOS-XE | `vrf definition NAME` / `rd ASN:NN` |
| Set RT export | IOS/IOS-XE | `route-target export ASN:NN` (under `address-family ipv4` in vrf def) |
| Set RT import | IOS/IOS-XE | `route-target import ASN:NN` |
| Attach VRF to interface | IOS/IOS-XE | `vrf forwarding NAME` |
| Enable VPNv4 AF on BGP neighbor | IOS/IOS-XE | `address-family vpnv4` / `neighbor x.x.x.x activate` |
| Send extended communities (RT) | IOS/IOS-XE | `neighbor x.x.x.x send-community extended` |
| Force next-hop-self on PE | IOS/IOS-XE | `neighbor x.x.x.x next-hop-self` (under vpnv4 AF) |
| Set per-VRF label mode | IOS-XR | `router bgp ASN` / `vrf NAME` / `address-family ipv4 unicast` / `label mode per-vrf` |
| Redistribute connected/static into VRF BGP | IOS/IOS-XE | `address-family ipv4 vrf NAME` / `redistribute connected` |
| OSPF PE-CE sham-link | IOS/IOS-XE | `router ospf PID vrf NAME` / `area 0 sham-link SRC DST cost N` |
| Tag route with SoO | IOS/IOS-XE | `route-map NAME permit 10` / `set extcommunity soo ASN:NN` |
| EIGRP PE-CE SoO application | IOS/IOS-XE | apply SoO route-map under `address-family ipv4 vrf NAME autonomous-system N` |
| Verify VRF routes | IOS/IOS-XE | `show ip route vrf NAME` |
| Verify BGP VPNv4 table | IOS/IOS-XE | `show bgp vpnv4 unicast vrf NAME` |
| Verify RD/RT on a route | IOS/IOS-XE | `show bgp vpnv4 unicast rd ASN:NN` |
| Verify VPN label per prefix | IOS/IOS-XE | `show bgp vpnv4 unicast vrf NAME x.x.x.x` (look for `Local Label`) |
| VRF-Lite interface config | IOS/IOS-XE | `interface X.subif` / `vrf forwarding NAME` / `encapsulation dot1Q NN` |
