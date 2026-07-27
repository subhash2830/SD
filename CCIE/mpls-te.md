# MPLS-TE — CCDE Notes

## 1. Subtopics

### 1.1 RSVP-TE Signaling and LSP Setup
**What:** MPLS Traffic Engineering uses RSVP-TE (extended RSVP with Path/Resv messages) to explicitly signal and reserve resources for a label-switched path (TE tunnel) along a computed path, rather than relying on IGP shortest-path forwarding.

**Why it matters (CCDE lens):** RSVP-TE's soft-state refresh model (periodic Path/Resv refresh, default 30s) is a real scalability concern at high tunnel counts — CCDE tests whether you know to enable refresh reduction (RFC 2961: summary refresh, bundle messages) at scale, and understand that RSVP state is maintained hop-by-hop (every LSR on the path keeps per-LSP state), unlike pure IGP/LDP forwarding which is stateless per-hop.

**Real-world example:** A carrier running several thousand TE tunnels across a core without refresh reduction saturates control-plane CPU on transit P routers purely from RSVP refresh overhead — a design oversight caught only under full production tunnel count, not in lab testing.

**CLI:**
```
mpls traffic-eng tunnels
interface Gi0/0/0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 1000000
!
interface Tunnel1
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 10.0.0.5
 tunnel mpls traffic-eng bandwidth 50000
 tunnel mpls traffic-eng path-option 10 dynamic
```

### 1.2 CSPF (Constrained Shortest Path First)
**What:** The head-end router computes the TE tunnel's path using CSPF — a modified Dijkstra run against the TE database (link bandwidth, affinity/admin-group, metric) rather than plain IGP metric — selecting the shortest path that satisfies the tunnel's constraints.

**Why it matters (CCDE lens):** CSPF requires an IGP extended with TE link-state information (OSPF-TE or IS-IS-TE) flooding bandwidth/attribute data — a common design gap is forgetting that plain OSPF/IS-IS doesn't carry this by default, so CSPF silently fails to find constrained paths without TE extensions enabled. CCDE also tests understanding that CSPF runs only at the head-end (distributed, not centralized like a PCE) unless PCE is explicitly introduced.

**Real-world example:** A network migrates to MPLS-TE but forgets `mpls traffic-eng router-id` and TE-extension flooding on some IS-IS interfaces — tunnels along those links fail to signal because CSPF simply doesn't see them as TE-capable, defaulting to a longer, non-optimal path or failing entirely if no path satisfies constraints.

**CLI:**
```
router isis
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
```

### 1.3 Bandwidth Reservation and Admission Control
**What:** Each TE tunnel reserves a specified bandwidth along its path; RSVP tracks per-link reserved bandwidth so CSPF admission control at future tunnel setup only considers links with sufficient remaining bandwidth, preventing oversubscription of engineered paths.

**Why it matters (CCDE lens):** Static bandwidth reservation is a capacity-planning tool, not a hard guarantee of actual forwarding priority unless paired with MPLS DiffServ-aware TE (DS-TE) and proper QoS queuing — CCDE tests whether you understand that RSVP reservation alone doesn't police traffic; a tunnel can still be signaled for 50 Mbps and carry 200 Mbps of actual traffic unless enforced by shaping/policing at the tunnel head-end.

**Real-world example:** An operator relies purely on RSVP bandwidth reservation assuming it enforces traffic limits, then experiences congestion because the reserved bandwidth value was never tied to an actual QoS policy on the tunnel interface — a classic "reservation ≠ enforcement" design gap.

**CLI:**
```
interface Tunnel1
 tunnel mpls traffic-eng bandwidth 50000
!
policy-map TUNNEL-SHAPE
 class class-default
  shape average 50000000
interface Tunnel1
 service-policy output TUNNEL-SHAPE
```

### 1.4 Affinity / Administrative Groups (Link Coloring)
**What:** Links are tagged with administrative group bits (colors); tunnels specify `affinity`/`include`/`exclude` constraints so CSPF only considers links matching the required color pattern — used to force or forbid tunnels from specific link types (e.g., avoid satellite backhaul, prefer fiber).

**Why it matters (CCDE lens):** This is the primary tool for CCDE-level policy-based path steering beyond pure bandwidth/metric — but it's a manual, static classification that must be maintained as the topology changes; a common real-world failure is stale affinity bits after a link technology upgrade (e.g., microwave replaced with fiber but the "avoid-microwave" color bit never removed), silently constraining tunnels to suboptimal paths indefinitely.

**Real-world example:** A provider colors all satellite-backhaul links with admin-group bit 3 and configures latency-sensitive voice tunnels to exclude that bit — years later the satellite links are decommissioned and replaced with terrestrial fiber, but nobody removes the exclude constraint, so those tunnels keep avoiding perfectly good paths and take a longer route.

**CLI:**
```
interface Gi0/0/2
 mpls traffic-eng attribute-flags 0x8
!
interface Tunnel2
 tunnel mpls traffic-eng affinity 0x0 mask 0x8
```

### 1.5 Fast Reroute (FRR) — Link and Node Protection
**What:** FRR pre-establishes a backup LSP (bypass tunnel for facility/many-to-one protection, or detour for one-to-one) around a protected link or node, activated locally by the point of local repair (PLR) within tens of milliseconds of failure detection — before the head-end even learns of the failure via IGP reconvergence.

**Why it matters (CCDE lens):** FRR is what makes MPLS-TE competitive with SONET-class 50ms restoration — CCDE expects you to distinguish link protection (bypass around one link) from node protection (bypass around an entire downstream node, requires the PLR to know the next-next-hop, i.e., more complex CSPF computation) and to know that FRR is inherently a *local repair* mechanism: it doesn't optimize the path, it just keeps traffic flowing until the head-end can re-signal an optimal path.

**Real-world example:** A metro-core TE deployment uses facility-based FRR with node protection at every core router; a P-router hardware failure triggers sub-50ms local repair via bypass tunnels while the head-end takes several seconds to compute and signal a new primary LSP — voice/video traffic experiences no perceptible impact.

**CLI:**
```
interface Gi0/0/0
 mpls traffic-eng backup-path Tunnel99
!
interface Tunnel1
 tunnel mpls traffic-eng fast-reroute
!
interface Tunnel99
 tunnel mpls traffic-eng bandwidth 100000
 tunnel mpls traffic-eng path-option 10 explicit name BYPASS-PATH
```

### 1.6 Auto-Bandwidth
**What:** A head-end feature that periodically samples actual tunnel traffic and automatically re-signals the tunnel with an adjusted bandwidth reservation (up or down) to match real utilization, within configured min/max bounds.

**Why it matters (CCDE lens):** Auto-bandwidth trades manual capacity-planning overhead for a resignaling churn risk — every adjustment tears down and re-signals the LSP (a brief make-before-break event if configured correctly, or a hard cut if not), so CCDE tests whether you've set sane adjustment thresholds/intervals to avoid excessive resignaling on bursty traffic patterns, and whether make-before-break is actually enabled to avoid traffic loss during the resize.

**Real-world example:** A tunnel carrying bursty backup-traffic patterns with an aggressive (short-interval, low-threshold) auto-bandwidth config resignals dozens of times per day, generating unnecessary RSVP churn and occasional micro-drops during non-make-before-break resizes — tuning the sampling interval and adjustment threshold resolves it.

**CLI:**
```
interface Tunnel1
 tunnel mpls traffic-eng auto-bw
 tunnel mpls traffic-eng auto-bw frequency 300
 tunnel mpls traffic-eng auto-bw min-bw 10000
 tunnel mpls traffic-eng auto-bw max-bw 100000
```

### 1.7 DS-TE (DiffServ-Aware Traffic Engineering)
**What:** Extends basic TE bandwidth pools into multiple sub-pools (e.g., a global pool and a smaller guaranteed sub-pool) so different traffic classes can have independently constrained bandwidth admission — e.g., reserving a strict sub-pool exclusively for voice/real-time tunnels separate from best-effort data tunnels sharing the same links.

**Why it matters (CCDE lens):** DS-TE is what actually ties RSVP bandwidth reservation to true class-based guarantees — plain MPLS-TE bandwidth reservation is a single undifferentiated pool, so a bulk-data tunnel and a voice tunnel compete for the same admission-control budget. CCDE tests whether you recognize when DS-TE (vs plain TE) is required — specifically, any design claiming per-class bandwidth guarantees over shared TE-engineered links needs DS-TE, not just QoS queuing alone.

**Real-world example:** A provider offering a premium low-latency voice-transport service over the same core links as bulk enterprise data traffic uses DS-TE's sub-pool to guarantee voice tunnels always have admission-controlled bandwidth headroom, independent of how congested the global pool becomes from data tunnels.

**CLI:**
```
interface Gi0/0/0
 ip rsvp bandwidth 1000000 sub-pool 200000
!
interface Tunnel1
 tunnel mpls traffic-eng bandwidth sub-pool 50000
```

### 1.8 Explicit Path Options and Path-Option Priority
**What:** Tunnels can be configured with multiple ordered path-options (explicit hop-by-hop paths, or dynamic/CSPF-computed) tried in preference order — if the primary explicit path is unavailable, the head-end falls to the next path-option.

**Why it matters (CCDE lens):** Path-option ordering is a deliberate design lever for controlled degradation — CCDE tests whether candidates design a sensible fallback chain (e.g., explicit-preferred-path → explicit-diverse-backup-path → dynamic-as-last-resort) rather than relying purely on dynamic CSPF, which can pick unpredictable paths during large-scale failures when many tunnels resignal simultaneously and compete for remaining capacity.

**Real-world example:** A latency-sensitive tunnel is configured with an explicit low-latency primary path, an explicit (slightly longer but still acceptable) secondary path, and a dynamic path-option as final fallback — ensuring predictable behavior during normal failures while still having a path during a major, unanticipated multi-failure event.

**CLI:**
```
interface Tunnel1
 tunnel mpls traffic-eng path-option 10 explicit name PRIMARY-PATH
 tunnel mpls traffic-eng path-option 20 explicit name BACKUP-PATH
 tunnel mpls traffic-eng path-option 30 dynamic
```

---

## 2. Interview Q&A

**Q1: Why does RSVP-TE's soft-state model become a scalability concern at high tunnel counts, and how do you mitigate it?**
A: Every LSR on a tunnel's path maintains per-LSP state refreshed periodically (default 30s Path/Resv messages) — at thousands of tunnels this generates significant control-plane refresh overhead on transit routers. Mitigate with RFC 2961 refresh reduction (summary refresh, message bundling, reliable messaging) to cut refresh volume.

**Q2: Why can CSPF fail to find a path even though the IGP shows full reachability?**
A: CSPF depends on TE-extended link-state flooding (OSPF-TE/IS-IS-TE carrying bandwidth and attribute data), not plain IGP reachability — if TE extensions aren't enabled on some links/routers, those links are invisible to CSPF even though normal IGP forwarding works fine over them, so CSPF may fail to find a constrained path or pick a suboptimal one.

**Q3: Does RSVP bandwidth reservation on a tunnel actually enforce a traffic limit? Why or why not?**
A: No — RSVP reservation only performs admission control at tunnel setup time (preventing new tunnels from oversubscribing already-reserved bandwidth); it does not police or shape the actual traffic inside the tunnel. Enforcement requires separate QoS shaping/policing applied to the tunnel interface.

**Q4: What's the operational risk of using affinity/admin-group link coloring as your primary path-steering mechanism?**
A: It's a static, manually maintained classification — if underlying link technology changes (e.g., a colored "avoid" link is replaced with better infrastructure) but the color/exclude constraint isn't updated, tunnels keep avoiding perfectly good paths indefinitely, a stale-config failure mode that's easy to miss since nothing breaks, it just silently underperforms.

**Q5: Explain the difference between link protection and node protection in FRR, and why node protection is more complex to compute.**
A: Link protection creates a bypass around a single protected link, rerouting to the same next-hop via an alternate path. Node protection creates a bypass around an entire downstream node, requiring the point-of-local-repair to reach the next-next-hop instead — this requires the PLR to know downstream topology beyond its immediate neighbor, making the CSPF computation for the bypass tunnel more complex than simple link protection.

**Q6: What problem does auto-bandwidth actually solve, and what's the tradeoff if tuned poorly?**
A: It automates matching a tunnel's reserved bandwidth to real traffic utilization, removing manual capacity-planning toil. Tuned poorly (too-sensitive thresholds, short sampling intervals on bursty traffic), it causes excessive resignaling churn, and if make-before-break isn't properly enabled, resize events can cause brief traffic loss.

**Q7: When is plain MPLS-TE bandwidth reservation insufficient, and what does DS-TE add?**
A: Plain TE has a single undifferentiated bandwidth pool per link, so different traffic classes (e.g., voice vs bulk data) compete for the same admission budget with no class-level guarantee. DS-TE adds sub-pools so specific classes (e.g., real-time voice tunnels) can be guaranteed bandwidth independent of how congested the global pool becomes from other traffic.

**Q8: Why would you configure multiple ordered path-options instead of relying solely on dynamic CSPF for every tunnel?**
A: Dynamic CSPF picks unpredictable paths, especially during widescale failures when many tunnels resignal simultaneously and compete for the same remaining capacity — an ordered explicit-path-option chain gives predictable, tested primary/backup behavior for normal failure scenarios, falling back to dynamic CSPF only as a last resort for unanticipated multi-failure conditions.

---

## 3. Memory Map

```
MPLS-TE
├── Signaling
│    └── RSVP-TE (Path/Resv soft-state)
│         └── scale concern → refresh reduction (RFC 2961) at high tunnel count
├── Path Computation
│    └── CSPF (head-end, per-tunnel)
│         ├── requires → TE-extended IGP (OSPF-TE / IS-IS-TE) flooding bandwidth+attributes
│         └── without TE extensions on a link → CSPF blind to it, even if IGP reachable
├── Admission Control
│    └── Bandwidth Reservation
│         ├── prevents oversubscription at SETUP time only
│         └── does NOT enforce/police actual traffic — needs separate QoS shaping
├── Path Steering Policy
│    └── Affinity / Admin-Group Link Coloring
│         └── static — risk of stale include/exclude after topology/tech changes
├── Resiliency
│    └── Fast Reroute (FRR)
│         ├── Link Protection   — bypass around one link
│         └── Node Protection   — bypass around next-next-hop, harder to compute
│              └── local repair only — does NOT optimize path, just survives until head-end resignals
├── Dynamic Capacity Adjustment
│    └── Auto-Bandwidth
│         ├── benefit → matches reservation to real utilization automatically
│         └── risk → resignal churn if thresholds/intervals tuned poorly; needs make-before-break
├── Class-Aware Guarantees
│    └── DS-TE (sub-pools)
│         └── required whenever a design claims PER-CLASS bandwidth guarantees over shared TE links
└── Fallback Design
     └── Ordered Path-Options (explicit-primary → explicit-backup → dynamic-last-resort)
          └── predictable degradation vs pure dynamic CSPF unpredictability under mass failure
```

---

## 4. CLI Cheat Sheet

| Task | Command |
|---|---|
| Enable MPLS-TE globally | `mpls traffic-eng tunnels` |
| Enable MPLS-TE on interface | `interface X` / `mpls traffic-eng tunnels` |
| Reserve RSVP bandwidth on interface | `ip rsvp bandwidth N` |
| Reserve DS-TE sub-pool bandwidth | `ip rsvp bandwidth N sub-pool M` |
| Enable TE extensions in IS-IS | `router isis` / `mpls traffic-eng level-2` / `mpls traffic-eng router-id Loopback0` |
| Enable TE extensions in OSPF | `router ospf N` / `mpls traffic-eng area 0` / `mpls traffic-eng router-id Loopback0` |
| Create TE tunnel interface | `interface TunnelN` / `tunnel mode mpls traffic-eng` |
| Set tunnel destination | `tunnel destination x.x.x.x` |
| Set tunnel bandwidth (global pool) | `tunnel mpls traffic-eng bandwidth N` |
| Set tunnel bandwidth (DS-TE sub-pool) | `tunnel mpls traffic-eng bandwidth sub-pool N` |
| Configure dynamic path option | `tunnel mpls traffic-eng path-option N dynamic` |
| Configure explicit path option | `tunnel mpls traffic-eng path-option N explicit name NAME` |
| Define explicit path hops | `ip explicit-path name NAME` / `next-address x.x.x.x` |
| Set link admin-group/color | `interface X` / `mpls traffic-eng attribute-flags 0xN` |
| Constrain tunnel by affinity | `tunnel mpls traffic-eng affinity 0xN mask 0xM` |
| Enable FRR on a tunnel | `tunnel mpls traffic-eng fast-reroute` |
| Assign backup tunnel to protected interface | `interface X` / `mpls traffic-eng backup-path TunnelN` |
| Enable auto-bandwidth | `tunnel mpls traffic-eng auto-bw` |
| Set auto-bandwidth sampling interval | `tunnel mpls traffic-eng auto-bw frequency N` |
| Set auto-bandwidth min/max | `tunnel mpls traffic-eng auto-bw min-bw N` / `max-bw N` |
| Verify tunnel status | `show mpls traffic-eng tunnels` |
| Verify CSPF-computed path | `show mpls traffic-eng tunnels tunnel N` |
| Verify TE topology database | `show mpls traffic-eng topology` |
| Verify FRR protection status | `show mpls traffic-eng fast-reroute database` |
| Verify RSVP reservations on interface | `show ip rsvp interface` |
