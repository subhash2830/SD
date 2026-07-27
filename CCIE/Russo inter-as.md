# Russo Inter-AS (NNI Multi-Service Inter-AS) — CCDE Notes

> Named after Nick Russo's SPv4 workbook topology (used as the "russo-spv4" lab topology in the CCIE-SP v5.1 GitBook). This section goes beyond basic Option A/B/C L3VPN and drills the same inter-AS options carrying multiple service types (L3VPN, L2VPN, mVPN) across a formal Network-to-Network Interface (NNI) boundary between two independently administered providers.

## 1. Subtopics

### 1.1 NNI (Network-to-Network Interface) as a Design Concept
**What:** An NNI is the formal demarcation and interconnection point between two independently managed provider networks — distinct from a UNI (User-Network Interface, customer-facing). It's the boundary where per-service inter-AS mechanisms (Option A/B/C) are actually applied.

**Why it matters (CCDE lens):** CCDE treats the NNI as a governance boundary, not just a physical link — SLAs, trust, capacity planning, and failure-domain isolation all get defined at the NNI. A common design trap is treating NNI capacity/redundancy as an afterthought while over-engineering the per-VRF inter-AS mechanism; if the NNI itself has no redundancy, the elegance of Option C's scalability is irrelevant during a single link failure.

**Real-world example:** Two providers formalize a single redundant NNI (dual diverse-path links) carrying L3VPN, L2VPN, and mVPN traffic simultaneously — the NNI design (link diversity, capacity, QoS trust boundary) is negotiated once, then each service type layers its inter-AS option on top.

**CLI:**
```
interface Gi0/0/0
 description NNI-to-ASxxxxx-primary
interface Gi0/0/1
 description NNI-to-ASxxxxx-backup
```

### 1.2 Option A L3NNI (VRF-to-VRF over NNI)
**What:** The classic Option A back-to-back VRF technique applied specifically at a formal NNI boundary — dedicated sub-interfaces per VRF, standard eBGP IPv4 between ASBRs acting as mutual CEs.

**Why it matters (CCDE lens):** At NNI scale (many customer VRFs crossing between two large providers), Option A's per-VRF sub-interface requirement becomes a serious capacity-planning and change-management burden — every new shared customer requires physical/logical NNI changes on both sides. CCDE expects you to identify the inflection point (VRF count) where Option A NNI design must transition to B or C.

**Real-world example:** A wholesale peering arrangement with only 8 shared enterprise VRFs uses Option A L3NNI because the NNI change volume is low and both providers want zero VPN-route visibility into each other's core.

**CLI:**
```
interface Gi0/0/0.810
 encapsulation dot1Q 810
 vrf forwarding CUST_810
 ip address 172.16.10.1 255.255.255.252
router bgp 65000
 address-family ipv4 vrf CUST_810
  neighbor 172.16.10.2 remote-as 65001
```

### 1.3 Option A L2NNI (L2VPN Back-to-Back over NNI)
**What:** The L2VPN analog of Option A — each provider terminates its own pseudowire/EVPN instance locally and stitches it to the peer's L2 service at the NNI via a locally significant VLAN/attachment circuit, rather than extending a single end-to-end pseudowire across the boundary.

**Why it matters (CCDE lens):** L2 stitching at Option A introduces a data-plane MTU and VLAN-translation coordination problem that L3 Option A doesn't have — CCDE will test whether you account for VLAN tag rewrite at the NNI and end-to-end MTU consistency across two independently operated MPLS cores with potentially different max-MTU support.

**Real-world example:** Two providers stitch a customer's Ethernet Virtual Private Line at the NNI using locally significant VLAN 810 on each side, translated at the boundary switch — a VLAN mismatch here silently drops the customer's L2 service without any routing protocol ever indicating failure.

**CLI:**
```
l2vpn
 xconnect group NNI-STITCH
  p2p CUST_810_STITCH
   interface Gi0/0/0.810
   neighbor 203.0.113.5 pw-id 810
```

### 1.4 Option A mVPN (Multicast VPN Back-to-Back over NNI)
**What:** Extending multicast VPN (Rosen mVPN/MDT or MVPN with BGP-based auto-discovery) across an Option A boundary — each provider runs its own multicast core and stitches customer multicast state at the NNI, typically via PIM between ASBRs per VRF.

**Why it matters (CCDE lens):** mVPN inter-AS is where most candidates fail — it's not just "add multicast to the existing VRF," because Default-MDT/Data-MDT group addressing and RP placement must be coordinated (or translated) across two independently numbered multicast domains. CCDE will test whether you understand that PIM state, not just BGP routes, must traverse the NNI correctly for this to work — a common failure is customer multicast streams working for low-bandwidth Default-MDT traffic but failing to cut over to Data-MDT thresholds across the boundary.

**Real-world example:** A financial customer's market-data multicast feed spans two providers; Option A mVPN with mismatched Data-MDT threshold configuration on each side causes intermittent stream drops only under high subscriber load — a classic hard-to-diagnose inter-AS mVPN bug.

**CLI:**
```
vrf definition CUST_MC
 rd 65000:810
 address-family ipv4
  mdt default 232.0.0.1
router pim vrf CUST_MC
 rp-address 10.255.255.1
```

### 1.5 Option B L3NNI and Option B mVPN
**What:** ASBR-to-ASBR MP-eBGP VPNv4 exchange (as in standard Option B) plus, for mVPN, MDT-SAFI (BGP-based MVPN auto-discovery, RFC 6514) exchanged between ASBRs so multicast state doesn't require raw PIM peering across the trust boundary.

**Why it matters (CCDE lens):** BGP-based MVPN (MDT-SAFI/Type routes) is architecturally cleaner at Option B than PIM-based mVPN because it rides the same MP-eBGP session already carrying VPNv4 — no separate PIM adjacency needs to cross the NNI. CCDE candidates are expected to recommend BGP-based MVPN over PIM/mGRE-based mVPN specifically at inter-AS boundaries for this reason — fewer moving parts, same trust model as unicast.

**Real-world example:** A carrier upgrading from Rosen-GRE mVPN (PIM-based) to BGP-based MVPN at their Option B NNI eliminates a whole class of PIM-neighbor-flap incidents that had nothing to do with the underlying unicast VPNv4 session health.

**CLI:**
```
router bgp 65000
 address-family ipv4 mvpn
  neighbor 203.0.113.1 activate
  neighbor 203.0.113.1 send-community extended
```

### 1.6 Option C L3NNI, Option C L3NNI w/ L2VPN, and Option C mVPN
**What:** Multi-hop PE-to-PE MP-eBGP (Option C) extended to carry not just VPNv4 but also L2VPN (VPWS/VPLS via BGP-based signaling, e.g., BGP AD or EVPN) and MVPN AFI/SAFI end-to-end between PEs in different ASes, with ASBRs remaining pure label-switched transit via labeled BGP.

**Why it matters (CCDE lens):** This is the "everything scales, but everything is exposed" endpoint — running L3VPN, L2VPN signaling, and MVPN all as direct PE-PE sessions across an AS boundary means both providers' PEs must have fully coordinated RT policy across three different service AFI/SAFIs simultaneously, and a single ASBR labeled-BGP misconfiguration silently breaks every service type at once (versus Option A/B where failures are typically isolated per-service). CCDE will test your ability to articulate this combined blast radius versus the scale benefit.

**Real-world example:** A single company operating two ASes for historical reasons runs Option C for L3VPN, L2VPN, and mVPN all together across their internal NNI — appropriate because it's the same trust domain, but the design must still isolate labeled-BGP failures at the ASBR from taking down all three service types via redundant ASBR pairs and per-AFI monitoring.

**CLI:**
```
router bgp 65000
 neighbor 198.51.100.5 remote-as 65001
 neighbor 198.51.100.5 ebgp-multihop 5
 neighbor 198.51.100.5 update-source Loopback0
 address-family vpnv4
  neighbor 198.51.100.5 activate
 address-family l2vpn evpn
  neighbor 198.51.100.5 activate
 address-family ipv4 mvpn
  neighbor 198.51.100.5 activate
```

---

## 2. Interview Q&A

**Q1: Why is an NNI treated as a governance/SLA boundary rather than just a physical interconnect in CCDE design terms?**
A: Because the NNI is where two independently managed trust domains meet — capacity, redundancy, QoS trust, and failure-domain isolation must be explicitly negotiated there regardless of which inter-AS mechanism (A/B/C) is layered on top; an elegant Option C design is worthless if the underlying NNI itself is a single non-redundant link.

**Q2: Why does Option A L2NNI introduce an MTU/VLAN coordination problem that Option A L3NNI doesn't have as severely?**
A: L3 Option A just exchanges IP routes over a routed sub-interface — normal IP fragmentation/PMTUD concepts apply. L2 Option A stitches Ethernet frames end-to-end across two independently operated cores with locally significant VLAN tags; a VLAN mismatch or MTU mismatch at the stitch point silently breaks the service at the data plane with no routing protocol signal.

**Q3: Why is BGP-based MVPN (MDT-SAFI) preferred over PIM-based mVPN specifically at inter-AS boundaries?**
A: BGP-based MVPN rides the same MP-eBGP session already used for VPNv4, so no separate PIM adjacency needs to be established and maintained across the trust boundary — this removes an entire class of inter-provider PIM-neighbor-flap failures unrelated to the health of the underlying unicast session.

**Q4: In Option C combining L3VPN, L2VPN, and MVPN over one PE-PE session set, what's the blast-radius risk compared to running separate Option B sessions per service?**
A: A single labeled-BGP or ASBR failure in the shared transit path can silently break all three service AFI/SAFIs simultaneously, since they all ride the same underlying LSP and PE-PE session infrastructure — whereas Option B's per-service ASBR handling tends to isolate failures to one service type at a time.

**Q5: What operational inflection point should trigger moving an NNI from Option A to Option B or C?**
A: When per-VRF sub-interface and session count at the NNI becomes an unmanageable change-management burden — every new shared customer under Option A requires a physical/logical config change on both sides' ASBRs, which doesn't scale past a modest VRF count.

**Q6: Why can a Data-MDT threshold mismatch across an Option A mVPN NNI cause intermittent (not constant) failures?**
A: Default-MDT carries all multicast traffic at low volume without issue; only when traffic crosses the Data-MDT switchover threshold does the mismatch matter — if each provider's threshold configuration differs, one side may switch to a Data-MDT group the other side isn't listening on, dropping high-bandwidth streams while low-bandwidth ones continue working fine.

**Q7: Why does CCDE consider RT policy coordination "harder" in a multi-service Option C NNI versus a single-service one?**
A: Because RT/import-export policy must now be consistently coordinated across three separate AFI/SAFIs (VPNv4, L2VPN/EVPN, MVPN) between two organizations' PEs directly, with no ASBR checkpoint to catch a mismatch in any one of them — tripling the surface area for a misconfiguration-driven route leak or connectivity failure.

**Q8: When is it appropriate to run Option C across an NNI between two genuinely separate companies (not just two ASes of the same org)?**
A: Essentially only when the partnership is so tight and long-term that both organizations are willing to expose PE loopbacks and coordinate RT/RD policy directly at the PE level with no ASBR-level checkpoint — in practice this is rare between independent companies and is usually reserved for the same-company/multi-AS case Russo's labs use as the canonical example.

---

## 3. Memory Map

```
Russo Inter-AS (NNI Multi-Service)
├── NNI as Governance Boundary
│    ├── SLA / redundancy / capacity negotiated independently of service option
│    └── failure here undermines any Option A/B/C elegance above it
├── Option A (VRF-to-VRF) — per-VRF sub-interface, per-service session
│    ├── L3NNI  → standard eBGP IPv4 per VRF
│    ├── L2NNI  → pseudowire/EVPN stitching, needs VLAN + MTU coordination
│    └── mVPN   → PIM per-VRF stitching, Default/Data-MDT threshold coordination risk
├── Option B (ASBR-to-ASBR MP-eBGP)
│    ├── L3NNI  → VPNv4 exchange, ASBR re-origination
│    └── mVPN   → BGP-based MVPN (MDT-SAFI) preferred over PIM — rides same session as VPNv4
├── Option C (Multi-hop PE-to-PE)
│    ├── L3NNI            → VPNv4 direct PE-PE
│    ├── L3NNI w/ L2VPN   → adds L2VPN/EVPN AFI/SAFI on same PE-PE session set
│    └── mVPN              → adds MVPN AFI/SAFI on same PE-PE session set
│         └── combined blast radius: one ASBR/labeled-BGP failure impacts ALL service types at once
└── CCDE Decision Threads
     ├── VRF/service count → forces A → B/C transition
     ├── Same-org multi-AS vs true inter-company → determines max acceptable Option (C only for tight/same-org trust)
     └── Multi-service convergence (L3+L2+mVPN together) → higher blast radius, demands per-AFI monitoring + redundant ASBRs
```

---

## 4. CLI Cheat Sheet

| Task | Option/Service | Command |
|---|---|---|
| NNI sub-interface labeling (documentation practice) | All | `interface X` / `description NNI-to-ASxxxxx-primary` |
| Option A L3NNI per-VRF sub-interface | A / L3 | `interface X.subif` / `vrf forwarding NAME` / `encapsulation dot1Q NN` |
| Option A L3NNI eBGP session | A / L3 | `address-family ipv4 vrf NAME` / `neighbor x.x.x.x remote-as ASN` |
| Option A L2NNI pseudowire stitch | A / L2 | `l2vpn` / `xconnect group NAME` / `p2p NAME` / `neighbor x.x.x.x pw-id N` |
| Option A mVPN default-MDT | A / mVPN | `vrf definition NAME` / `address-family ipv4` / `mdt default 232.x.x.x` |
| Option A mVPN RP assignment | A / mVPN | `router pim vrf NAME` / `rp-address x.x.x.x` |
| Option B L3NNI VPNv4 session | B / L3 | `address-family vpnv4` / `neighbor x.x.x.x activate` |
| Option B mVPN (MDT-SAFI) session | B / mVPN | `address-family ipv4 mvpn` / `neighbor x.x.x.x activate` |
| Option C multi-hop PE-PE base session | C / all | `neighbor x.x.x.x ebgp-multihop N` / `update-source Loopback0` |
| Option C L3NNI VPNv4 AF | C / L3 | `address-family vpnv4` / `neighbor x.x.x.x activate` |
| Option C L2VPN (EVPN) AF | C / L2 | `address-family l2vpn evpn` / `neighbor x.x.x.x activate` |
| Option C mVPN AF | C / mVPN | `address-family ipv4 mvpn` / `neighbor x.x.x.x activate` |
| Verify mVPN MDT state | mVPN | `show ip pim vrf NAME mdt` |
| Verify BGP MVPN routes | mVPN (BGP-based) | `show bgp ipv4 mvpn all` |
| Verify L2VPN xconnect stitch status | L2NNI | `show l2vpn xconnect` |
| Verify labeled-unicast reachability at Option C ASBR | C | `show bgp ipv4 labeled-unicast` |
