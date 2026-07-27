# Inter-AS MPLS L3VPN — CCDE Notes

## 1. Subtopics

### 1.1 Inter-AS Option A (VRF-to-VRF / Back-to-Back)
**What:** ASBRs in each AS are directly connected via sub-interfaces, one per VRF, running standard eBGP IPv4 (not VPNv4) between them — each ASBR treats the other as a CE. No MPLS label exchange crosses the AS boundary.

**Why it matters (CCDE lens):** Option A is the "boring but safe" answer — simplest to implement, no trust extended across the AS boundary (each provider controls its own label space fully), works with any PE/ASBR platform, but scales terribly: one sub-interface + one eBGP session per VRF per ASBR pair. CCDE scenarios test whether you can articulate WHY a design uses A despite the scaling cost — usually regulatory/trust boundaries (must not run MPLS across an untrusted AS boundary) or a small, stable VRF count.

**Real-world example:** Two providers merging via an M&A event use Option A at a handful of shared customer boundary points because neither trusts the other's label allocation practices yet, and only 5 VRFs need to cross.

**CLI:**
```
interface Gi0/0/0.100
 encapsulation dot1Q 100
 vrf forwarding CUST_A
 ip address 192.168.1.1 255.255.255.252
!
router bgp 65000
 address-family ipv4 vrf CUST_A
  neighbor 192.168.1.2 remote-as 65001
  neighbor 192.168.1.2 activate
```

### 1.2 Inter-AS Option B (ASBR-to-ASBR MP-eBGP)
**What:** ASBRs exchange labeled VPNv4 routes directly via MP-eBGP; ASBRs re-originate routes (next-hop-self) and typically swap/re-allocate the VPN label at the boundary, so PEs in each AS never peer directly across AS.

**Why it matters (CCDE lens):** Option B centralizes trust and control at the ASBR (single policy enforcement/summarization point) at the cost of ASBR becoming both a scaling bottleneck (every VPNv4 route from every VRF must pass through and be re-labeled by the ASBR) and a single point with full VPN route visibility (a security/trust consideration — the ASBR sees all customer routes, unlike Option A). CCDE will probe redundant-ASBR failure scenarios and next-hop-unchanged variants.

**Real-world example:** A large national carrier connecting two regional autonomous systems (post-merger, same company) uses Option B — trust exists (same org) so full VPNv4 label re-origination at 2 redundant ASBR pairs is acceptable and far more scalable than Option A's per-VRF session sprawl.

**CLI:**
```
router bgp 65000
 neighbor 203.0.113.1 remote-as 65001
 address-family vpnv4
  neighbor 203.0.113.1 activate
  neighbor 203.0.113.1 send-community extended
  neighbor 203.0.113.1 next-hop-self
```

### 1.3 Inter-AS Option C (Multi-hop MP-eBGP, PE-to-PE)
**What:** PEs in different ASes establish a multi-hop MP-eBGP session directly (through, not to, the ASBRs), carrying VPNv4 routes end-to-end; ASBRs only exchange PE loopback /32s (via labeled BGP / RFC 3107, or an IGP+redistribution+LDP scheme) to build the inter-AS LSP, and never see VPN routes at all.

**Why it matters (CCDE lens):** Option C is the most scalable (ASBRs do zero per-VPN-route work — pure label-switched transit for PE loopbacks) but has the largest trust footprint: PEs across two different administrative domains must peer directly, meaning both providers must agree on route-target/RD policy coordination and, critically, this exposes PE loopbacks and requires a shared label-switched path across the boundary — a design most providers reject unless it's the same company or an extremely tight partnership (this is the classic CCDE "why would you NOT use the most scalable option" trap question).

**Real-world example:** A single global carrier with two ASes purely for BGP policy/historical reasons (not different trust domains) uses Option C between regional PE routers to get full any-to-any VPNv4 scale without ASBR bottlenecking.

**CLI:**
```
! On PE (multihop eBGP to remote PE loopback)
router bgp 65000
 neighbor 198.51.100.5 remote-as 65001
 neighbor 198.51.100.5 ebgp-multihop 5
 neighbor 198.51.100.5 update-source Loopback0
 address-family vpnv4
  neighbor 198.51.100.5 activate
  neighbor 198.51.100.5 send-community extended
! ASBRs run labeled BGP (RFC 3107) for PE loopback reachability
router bgp 65000
 neighbor 203.0.113.1 remote-as 65001
 address-family ipv4 labeled-unicast
  neighbor 203.0.113.1 activate
```

### 1.4 Labeled BGP (RFC 3107) as the Option C Glue
**What:** IPv4+label BGP address family used between ASBRs to advertise PE loopback /32s with a label, extending the LSP across the AS boundary without either ASBR needing VPNv4/VRF awareness.

**Why it matters (CCDE lens):** This is the mechanism that makes Option C's "ASBR sees nothing" property actually work end-to-end — without labeled BGP, the ASBR would need a full VPNv4 table or a separate LDP-over-MPLS extension across the boundary. CCDE candidates must be able to explain the label stack at each hop (transport label from labeled-BGP + VPN label from the PE-PE MP-eBGP session).

**Real-world example:** Two ASes under one company use labeled BGP purely to keep ASBR routing tables small (only loopbacks + labels) while still supporting a full multi-hop VPNv4 mesh between PEs.

**CLI:**
```
router bgp 65000
 address-family ipv4 labeled-unicast
  neighbor 203.0.113.1 activate
  neighbor 203.0.113.1 route-map ALLOW-LOOPBACKS out
```

### 1.5 RT / RD Coordination Across AS Boundaries
**What:** In any inter-AS option, the two providers must agree (out of band, typically contractually) on which RTs are exchanged/translated at the boundary so import/export policy remains correct once routes cross administrative domains.

**Why it matters (CCDE lens):** This is an operational/process design element CCDE loves to test indirectly — RT values are AS-scoped by convention (ASN:NN) but nothing stops an RT collision between two independent providers' internal numbering. Design must specify RT translation/rewrite at the ASBR (common in Option B) to avoid unintended route leaking when merging or partnering with another AS.

**Real-world example:** Provider X uses RT 65000:100 internally for "generic customer," Provider Y independently also uses 65001:100 for an unrelated customer — if these ever got translated/mapped incorrectly at an Option B ASBR, cross-customer route leakage (a real security incident) results.

**CLI:**
```
route-map RT-REWRITE-IN permit 10
 set extcommunity rt 65000:500 additive
!
router bgp 65000
 neighbor 203.0.113.1 route-map RT-REWRITE-IN in
```

### 1.6 ASBR Redundancy and Failure Domains
**What:** Since Option B/C ASBRs (or Option A boundary routers) are the sole transit points between ASes, design must include redundant ASBR pairs with fast failure detection (BFD) and consistent VPN-label/route re-advertisement on failover.

**Why it matters (CCDE lens):** A single ASBR is a hard SPOF for every VRF crossing the AS boundary — CCDE will present a "why did the whole inter-AS VPN drop when one router failed" failure scenario expecting the answer: no ASBR redundancy, or redundancy present but BFD/timers too slow for the SLA, or asymmetric next-hop handling breaking fast reroute.

**Real-world example:** A provider running single-homed Option B ASBRs suffers a full multi-hour inter-AS VPN outage for hundreds of customer VRFs when the one ASBR's line card fails — textbook argument for dual ASBRs with BFD.

**CLI:**
```
router bgp 65000
 neighbor 203.0.113.1 fall-over bfd
!
bfd-template single-hop TEMPLATE1
 interval min-tx 100 min-rx 100 multiplier 3
```

---

## 2. Interview Q&A

**Q1: Compare Options A, B, and C in one sentence each, focused on the trust/scale tradeoff.**
A: Option A = zero trust, zero scale (per-VRF sub-interface + session); Option B = moderate trust (ASBR sees all VPN routes), good scale (single session set, but ASBR does per-route work); Option C = high trust (PEs peer directly across AS, ASBRs blind to VPN routes), best scale (ASBR only forwards loopback-labeled traffic).

**Q2: Why would a provider deliberately choose Option B over the more scalable Option C?**
A: Option C requires PEs in two different administrative domains to peer directly and requires exposing PE loopbacks plus coordinated RT policy at the PE level across organizations — most providers are unwilling to extend that much trust/coordination to an external AS. Option B confines the trust boundary to a small, controlled set of ASBRs.

**Q3: In Option C, what problem does labeled BGP (RFC 3107) solve at the ASBR?**
A: It lets the ASBR advertise PE loopback reachability with an MPLS label without needing VPNv4/VRF awareness — this keeps the ASBR's table small (just loopbacks) while still allowing the label-switched path to extend end-to-end for the PE-PE MP-eBGP VPNv4 session to ride on.

**Q4: What's the RT collision risk unique to inter-AS designs, and how do you mitigate it?**
A: RT values are only conventionally (not globally) unique per-AS (ASN:NN) — two independent providers can pick colliding RT values for unrelated customers. Mitigate with explicit RT translation/rewrite policy at the ASBR (Option B) or contractually coordinated RT allocation ranges.

**Q5: Why is Option A described as safe but not scalable — what specifically doesn't scale?**
A: Each VRF crossing the boundary needs its own dedicated sub-interface and its own eBGP IPv4 session between ASBRs — so both the physical/logical interface count and BGP session count grow linearly with VRF count, quickly becoming an operational and hardware burden at hundreds of VRFs.

**Q6: A customer VPN spans two ASes using Option B and a single ASBR pair goes down. What's the likely design flaw and fix?**
A: Likely flaw: no redundant ASBR pair, or redundancy exists but without BFD (default BGP hold timers are too slow for the SLA). Fix: dual ASBRs with BFD-triggered fast failover and verify VPN-label re-advertisement timing on the backup path.

**Q7: Why does Option C put a heavier trust burden on both providers compared to B, beyond just "PEs talk directly"?**
A: Because PEs across two orgs must coordinate RT/RD policy directly at the PE config level (no ASBR intermediary to enforce/rewrite policy), meaning a misconfiguration on either side's PE can leak or misroute VPN traffic with no boundary checkpoint to catch it — Option B's ASBR acts as that checkpoint.

**Q8: When does it make sense to mix inter-AS options (e.g., Option A for some VRFs, Option C for others) on the same ASBR pair?**
A: When VRF trust/scale requirements differ within the same partnership — e.g., a handful of high-security government VRFs use Option A for maximum isolation/auditability while the bulk of commercial VRFs use Option C or B for scale; this is common in real deployments and CCDE expects you to justify per-VRF policy rather than assuming one option fits all traffic.

---

## 3. Memory Map

```
Inter-AS L3VPN
├── Option A — VRF-to-VRF (back-to-back CE-like eBGP)
│    ├── Trust: none required (fully isolated per VRF)
│    ├── Scale: worst (1 subinterface + 1 session per VRF)
│    └── Use case: low VRF count, zero-trust boundary, M&A transition
├── Option B — ASBR-to-ASBR MP-eBGP (VPNv4)
│    ├── Trust: moderate (ASBR sees all VPN routes — policy enforcement point)
│    ├── Scale: good (single session set; ASBR still does per-route work)
│    ├── requires → RT coordination / rewrite at ASBR
│    └── requires → ASBR redundancy + BFD (SPOF risk)
├── Option C — Multi-hop PE-to-PE MP-eBGP
│    ├── Trust: highest (PEs cross-org peer directly; ASBR blind to VPN routes)
│    ├── Scale: best (ASBR only forwards on loopback label)
│    ├── enabled by → Labeled BGP / RFC 3107 (PE loopback + label across ASBR)
│    └── requires → direct RT/RD policy coordination at PE level (no ASBR checkpoint)
├── Cross-cutting Concerns
│    ├── RT/RD Coordination — collision risk across independent AS numbering
│    ├── ASBR Redundancy — SPOF for all crossing VRFs, needs BFD/fast failover
│    └── Mixed-Mode Design — different options per VRF based on trust tier
└── CCDE Decision Axis
     Trust required ↔ Scale achieved ↔ Operational complexity
     (A: low/low/low)  (B: med/med/med)  (C: high/high/high)
```

---

## 4. CLI Cheat Sheet

| Task | Option | Command |
|---|---|---|
| VRF sub-interface to peer ASBR | A | `interface X.subif` / `vrf forwarding NAME` / `encapsulation dot1Q NN` |
| eBGP IPv4 session per VRF | A | `address-family ipv4 vrf NAME` / `neighbor x.x.x.x remote-as ASN` |
| ASBR MP-eBGP VPNv4 session | B | `neighbor x.x.x.x remote-as ASN` / `address-family vpnv4` / `neighbor x.x.x.x activate` |
| Next-hop-self at ASBR (Option B) | B | `neighbor x.x.x.x next-hop-self` (under vpnv4 AF) |
| Send extended communities across boundary | B/C | `neighbor x.x.x.x send-community extended` |
| Multi-hop eBGP for PE-PE (Option C) | C | `neighbor x.x.x.x ebgp-multihop N` / `neighbor x.x.x.x update-source Loopback0` |
| Labeled BGP (RFC 3107) at ASBR | C | `address-family ipv4 labeled-unicast` / `neighbor x.x.x.x activate` |
| Restrict advertised loopbacks (Option C) | C | `neighbor x.x.x.x route-map ALLOW-LOOPBACKS out` |
| RT rewrite/translation at boundary | B | `route-map NAME` / `set extcommunity rt ASN:NN additive` |
| Apply RT rewrite inbound | B | `neighbor x.x.x.x route-map RT-REWRITE-IN in` |
| BFD for ASBR fast failover | A/B/C | `neighbor x.x.x.x fall-over bfd` |
| BFD interval template | A/B/C | `bfd-template single-hop NAME` / `interval min-tx N min-rx N multiplier N` |
| Verify labeled-unicast table | C | `show bgp ipv4 labeled-unicast` |
| Verify VPNv4 routes at ASBR | B | `show bgp vpnv4 unicast` |
| Verify inter-AS eBGP session state | A/B/C | `show bgp neighbors x.x.x.x` |
