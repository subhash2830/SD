
## Scenario 1: Design Provider Network with SR-EPE, BGP-LS, and PCE

### Scenario Overview

| Parameter | Details |
|-----------|---------|
| Company | Global Service Provider (GSP) |
| Network Size | 500 routers, 100 edge routers, 20 ISP peers |

**Requirements:**
- Optimize egress traffic to specific ISP peers (cost optimization)
- Centralized TE for low-latency paths
- BGP-free core (core only runs IGP, not full BGP)
- Sub-50ms failover for all critical paths

---

### Layer 1: Core Network (IGP Domain)

```
Core Routers (50 routers):
├── OSPF/IS-IS Area 0 (SR-enabled)
├── SRGB: [16000-23999]
├── Node SIDs: 101-150 (per core router)
├── Flex-Algo 128: Low-latency path
└── TI-LFA: <50ms failover
```

**Key Design Points:**
- SR-MPLS enabled on all core routers
- Node SIDs for each router (global uniqueness)
- Flex-Algo 128 for low-latency traffic (video/VoIP)
- TI-LFA for fast reroute (no RSVP-TE needed)

---

### Layer 2: Edge Network (BGP EPE)

```
Edge Routers (100 routers):
├── BGP MP (multiprotocol) with ISP peers
├── BGP EPE SIDs enabled (peer-level segments)
├── BGP-LS enabled (advertise topology to PCE)
└── SR Policy headend (steers traffic based on PCE)
```

**BGP EPE Configuration (Cisco IOS XR):**
```
router bgp 65000
 bgp router-id 10.0.0.1
 bgp bestpath egress-peer-engineering enable
 !
 neighbor 203.0.113.1
  remote-as 65100              ! ISP-1
  description ISP-1-Peering
  egress-peer-engineering
   peer-type external
   segment-routing
    peer-sid 32001              ! EPE SID for ISP-1
 !
 neighbor 198.51.100.1
  remote-as 65200              ! ISP-2
  egress-peer-engineering
   peer-type external
   segment-routing
    peer-sid 32002              ! EPE SID for ISP-2
```

**How EPE Works:**
```
Traffic to 8.8.8.8 (Google):
  - Headend sees multiple exit paths (ISP-1, ISP-2)
  - PCE computes: ISP-1 is cheaper (cost optimization)
  - SR Policy: [Node-Edge-A, Peer-SID-ISP-1]
  - Traffic goes via ISP-1 (not ISP-2)
```

---

### Layer 3: SDN Controller (PCE + BGP-LS)

```
SDN Controller (2 nodes for HA):
├── PCE (Path Computation Element)
├── BGP-LS Collector (receives topology from all routers)
├── ODN templates (automatic SR Policy creation)
├── PCEP server (programs SR Policies into headends)
└── RestAPI (northbound for applications)
```

**BGP-LS Configuration (All Routers):**
```
router bgp 65000
 bgp link-state
  report all
 !
 neighbor 10.0.100.1          ! SDN Controller
  remote-as 65000
  address-family link-state link-state
```

**What BGP-LS Advertises:**

| Information | Example |
|-------------|---------|
| Nodes | Router IDs, loopbacks (10.0.0.1/32, 10.0.0.2/32) |
| Links | Interfaces (192.168.1.0/30), bandwidth (10G) |
| Prefixes | IP prefixes + Node SIDs (16101, 16102) |
| SR Capabilities | SRGB [16000-23999], Flex-Algo 128 |
| EPE SIDs | Peer-SID 32001 (ISP-1), 32002 (ISP-2) |

---

### Layer 4: SR Policy Programming (PCEP)

**PCEP Configuration (Edge Routers):**
```
router segment-routing
 pce
  pce-name SDN-Controller
  address 10.0.100.1
  password PCEP-SECRET
  stateful
   active
```

**How PCE Programs SR Policy:**
```
Edge Router A requests path to 8.8.8.8 (ODN template matches):
  PCE computes: [Node-Edge-A, Peer-SID-ISP-1, Node-Core-1, ...]
  PCEP sends SR Policy to Edge Router A:
    Binding SID: 40001
    Segment List: [16101, 32001, 16105, 16110, ...]
  Edge Router A steers traffic:
    Match: 8.8.8.8/32 → Binding SID 40001
    Forward: [16101, 32001, 16105, ...]
```

---

### Complete Traffic Flow Example

```
Scenario: User at Edge-A wants to reach Google (8.8.8.8)

Step 1: Edge-A receives packet (8.8.8.8)
Step 2: ODN template matches → Headend requests path from PCE
Step 3: PCE computes optimal path (cost + latency):
        [Edge-A, Peer-ISP-1, Core-1, Core-5, Core-10, Edge-B, 8.8.8.8]
Step 4: PCEP programs SR Policy into Edge-A:
        Binding SID: 40001
        Segment List: [16101 (Edge-A), 32001 (Peer-ISP-1), 16105 (Core-1), ...]
Step 5: Edge-A steers traffic using Binding SID 40001
Step 6: Core forwards using top label (ECMP-aware Node SIDs)
Step 7: Edge-B receives, removes SR stack, forwards to 8.8.8.8

Result:
  ✅ Traffic via ISP-1 (cheaper, not ISP-2)
  ✅ BGP-free core (core only runs OSPF/IS-IS)
  ✅ <50ms failover (TI-LFA if Core-1 fails)
  ✅ Centralized TE (PCE computed optimal path)
```

### Benefits Achieved

| Requirement | Solution |
|-------------|----------|
| Egress optimization | BGP EPE (peer-level control) |
| Centralized TE | PCE (global optimization) |
| Network visibility | BGP-LS (topology to controller) |
| BGP-free core | SR-MPLS (IGP distributes SIDs) |
| Fast failover | TI-LFA (<50ms, no RSVP-TE) |
| Automation | ODN (automatic SR Policy) |

---

## Scenario 2: Complete Migration from MPLS/LDP to SR

### Migration Scenario Overview

| Parameter | Details |
|-----------|---------|
| Company | Enterprise Service Provider (ESP) |
| Current Network | MPLS with LDP + RSVP-TE |
| Network Size | 300 routers, 50 edge routers |

**Migration Goals:**
- No service disruption during migration
- Gradual rollout (edge → core)
- Rollback capability if issues
- Full SR after migration (remove LDP/RSVP)

---

### Phase 1: Pre-Migration Planning (Weeks 1-2)

**Tasks:**
1. Document current network:
   - LDP sessions (300 routers × avg 4 peers = 1200 sessions)
   - RSVP-TE tunnels (5000 tunnels)
   - Label assignments (static labels, LDP labels)
   - Traffic patterns (baseline utilization)

2. Define SR parameters:
   - SRGB: `[16000-23999]` (8000 labels)
   - Node SID mapping: Router ID → SID index
   - Adj SID strategy (dynamic vs static)

3. Create migration playbook:
   - Per-router config templates
   - Rollback procedures
   - Testing checklist
   - Monitoring scripts

**Node SID Mapping Table:**

| Router ID | Node SID Index | Actual Label |
|-----------|---------------|-------------|
| 10.0.0.1 (Core-1) | 101 | 16101 |
| 10.0.0.2 (Core-2) | 102 | 16102 |
| 10.0.0.10 (Edge-1) | 201 | 16201 |
| 10.0.0.11 (Edge-2) | 202 | 16202 |

---

### Phase 2: Enable IGP SR Extensions (Weeks 3-4)

**Action:** Enable SR on all IGP routers (core + edge) but keep LDP running.

**IS-IS SR Configuration (All Routers):**
```
router isis 1
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 !
 address-family ipv4 unicast
  segment-routing
   sr-enabled
  !
  srgb
   base-index 16000
   range 8000
  !
  node-sid
   prefix 10.0.0.1/32        ! Loopback
   index 101                 ! Node SID index
   no-advertise              ! Don't advertise yet (Phase 3)
```

**State After Phase 2:**
```
┌─────────────────────────────────────────┐
│ Router: Core-1                          │
├─────────────────────────────────────────┤
│ ✅ OSPF/IS-IS (IGP)                     │
│ ✅ SR extensions enabled                │
│ ✅ SRGB configured [16000-23999]        │
│ ❌ LDP still running (unchanged)        │
│ ❌ RSVP-TE still running (unchanged)    │
│ ❌ Node SIDs not advertised yet         │
└─────────────────────────────────────────┘
```

**Verification:**
```
# Verify SR is enabled
show isis segment-routing summary

# Verify SRGB configured
show isis segment-routing srgb

# Verify LDP still running (should see sessions)
show ldp session
```

---

### Phase 3: Enable SR on Edge Routers (Weeks 5-8)

**Action:** Enable SR on edge routers first (50 routers), advertise Node SIDs.

**Edge Router SR Configuration:**
```
router isis 1
 address-family ipv4 unicast
  segment-routing
   sr-enabled
  !
  node-sid
   prefix 10.0.0.10/32       ! Edge-1 loopback
   index 201
   advertise                 ! NOW advertise Node SID
```

**State After Phase 3:**
```
┌─────────────────────────────────────────┐
│ Edge Routers (50 routers):              │
├─────────────────────────────────────────┤
│ ✅ LDP running (existing)               │
│ ✅ SR enabled (new)                     │
│ ✅ Node SIDs advertised (IGP)           │
│ ✅ SR-LDP interworking enabled          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Core Routers (250 routers):             │
├─────────────────────────────────────────┤
│ ✅ LDP running (existing)               │
│ ✅ SR extensions enabled (Phase 2)      │
│ ❌ Node SIDs NOT advertised yet         │
│ ❌ LDP still primary (SR backup)        │
└─────────────────────────────────────────┘
```

**SR-LDP Interworking Configuration:**
```
router ldp
 segment-routing
  interworking enable         ! LDP ↔ SR mapping

router segment-routing
 ldp
  interworking enable         ! SR ↔ LDP mapping
```

**How SR-LDP Works:**
```
Packet from Edge-1 (SR) to Core-1 (LDP-only in Phase 3):

Edge-1:
  Push SR label [16101] (Node SID for Core-1)

Edge-1 (SR-LDP boundary):
  Maps SR label 16101 → LDP label 1001
  Pushes LDP label [1001]

Core-1 (LDP-only):
  Processes LDP label 1001
  Forwards to Core-1 destination
```

---

### Phase 4: Enable SR on Core Routers (Weeks 9-16)

**Action:** Enable SR on core routers (250 routers), advertise Node SIDs in batches.

**Batch Migration Plan (25 routers per batch):**
```
Batch 1:  Core-1   to Core-25  (Week 9-10)
Batch 2:  Core-26  to Core-50  (Week 11-12)
Batch 3:  Core-51  to Core-75  (Week 13-14)
...
Batch 10: Core-226 to Core-250 (Week 17-18)
```

**Traffic Steering Options During Phase 4:**

| Option | Description | Rollback |
|--------|-------------|---------|
| Option 1: LDP Primary, SR Backup | Conservative — LDP still primary | Fast, just disable SR |
| Option 2: SR Primary, LDP Backup | Aggressive — SR becomes primary | Disable SR, LDP takes over |

> **Recommended:** Option 1 (conservative) for first 3 batches, then Option 2.

**Verification After Each Batch:**
```
# Verify Node SID advertised
show isis prefix-sid 10.0.0.1/32

# Verify SR label in forwarding
show mpls forwarding-table label 16101

# Verify LDP still working (rollback path)
show ldp session

# Verify traffic flow
ping mpls ipv4 10.0.0.1       ! LSP ping for SR
```

---

### Phase 5: Remove LDP and RSVP-TE (Weeks 17-20)

**Verification Before Removal:**
```
# Check all Node SIDs are advertised
show isis prefix-sid summary      ! Should see 300 entries

# Check SR forwarding is working
show mpls forwarding-table        ! Should see SR labels only

# Check LDP sessions (should be 0)
show ldp session                  ! Should be empty

# Check RSVP-TE tunnels (should be 0)
show mpls traffic-eng tunnels     ! Should be empty
```

**Remove LDP (All Routers):**
```
no router ldp
```

**Remove RSVP-TE (All Routers):**
```
no router mpls traffic-eng
```

**Final State After Phase 5:**
```
┌─────────────────────────────────────────┐
│ Entire Network (300 routers):           │
├─────────────────────────────────────────┤
│ ✅ SR-MPLS primary (only protocol)      │
│ ✅ Node SIDs advertised (all 300)       │
│ ✅ TI-LFA enabled (<50ms failover)      │
│ ✅ Flex-Algo 128 (low-latency paths)    │
│ ❌ LDP removed (migrated)              │
│ ❌ RSVP-TE removed (migrated)          │
└─────────────────────────────────────────┘
```

---

### Complete Migration Rollback Plan

**If Issue Detected in Any Phase:**

```
Step 1: Disable SR advertisement
  router isis 1
   address-family ipv4 unicast
    segment-routing
     node-sid
      prefix 10.0.0.1/32
       no-advertise          ! STOP advertising Node SID

Step 2: Keep LDP running (already active):
  router ldp
   (unchanged)

Step 3: Verify LDP still forwarding:
  show ldp session            ! Should see sessions
  show ldp interface          ! Should see interfaces

Step 4: Test traffic:
  ping 10.0.0.1               ! Verify reachability
  show mpls ldp forwarding    ! Verify LDP labels working
```

> **Rollback Time:** <15 minutes per router

---

### Migration Timeline Summary

| Phase | Duration | Routers | What Changes | Risk Level |
|-------|----------|---------|-------------|-----------|
| 1. Planning | 2 weeks | 0 | Documentation, playbook | None |
| 2. IGP SR | 2 weeks | 300 | SR extensions enabled | Low |
| 3. Edge SR | 4 weeks | 50 | Node SIDs advertised, SR-LDP interworking | Medium |
| 4. Core SR | 8 weeks | 250 | Node SIDs advertised (batches) | Medium-High |
| 5. Remove LDP/RSVP | 4 weeks | 300 | LDP + RSVP removed | Low (if verified) |
| **Total** | **20 weeks** | **300** | **Full SR** | — |

### Key Success Criteria

| Criteria | How to Verify |
|----------|--------------|
| No service disruption | Ping tests during each phase, 0% packet loss |
| Rollback capability | Test rollback procedure before each phase |
| All Node SIDs advertised | `show isis prefix-sid summary` = 300 entries |
| SR forwarding working | `show mpls forwarding-table` shows SR labels only |
| TI-LFA enabled | `show segment-routing ti-lfa` shows backup paths |
| LDP/RSVP removed | `show ldp session` = empty; `show mpls traffic-eng tunnels` = empty |

### Common Migration Issues

| Issue | Root Cause | Solution |
|-------|-----------|----------|
| LDP sessions drop | SR-LDP interworking misconfig | Check `interworking enable` on both sides |
| Node SID not advertised | IGP not configured correctly | Verify `advertise` keyword on Node SID |
| Traffic loss during batch | SRGB mismatch between routers | Ensure all routers use same SRGB [16000-23999] |
| TI-LFA not working | Backup path not computed | Verify `segment-routing fast-reroute enabled` |
| SR labels not in FIB | CEF not updated | `clear mpls forwarding` or wait for CEF sync |

---

## Scenario 3: Service Chaining with Segment Routing

### Scenario Overview

| Parameter | Details |
|-----------|---------|
| Company | Cloud Service Provider |
| Requirement | Force customer traffic through security services before reaching cloud |

**Service Order:**
1. Firewall (inspect traffic)
2. IDS/IPS (detect attacks)
3. WAN Optimizer (compress data)

**Network:** 100 routers, 3 service nodes (Firewall, IDS, WAN Optimizer)

---

### Service Chaining Design

**Service Nodes:**
```
├── Firewall:  10.1.1.1/32, Node SID 16301
├── IDS/IPS:   10.1.1.2/32, Node SID 16302
└── WAN Opt:   10.1.1.3/32, Node SID 16303
```

**Service Node Configuration:**
```
router isis 1
 address-family ipv4 unicast
  segment-routing
   sr-enabled
  !
  node-sid
   prefix 10.1.1.1/32        ! Firewall loopback
   index 301
   advertise
```

**SR Policy for Service Chaining (Headend Edge-1):**
```
router segment-routing
 policy
  name SERVICE-CHAIN-1
  color 100                  ! Service chain color
  endpoint 10.2.2.1          ! Final destination (cloud)
  !
  segment-list PRIMARY
   segment 16101             ! Edge-1 (self)
   segment 16301             ! Firewall
   segment 16302             ! IDS/IPS
   segment 16303             ! WAN Optimizer
   segment 16401             ! Cloud-Edge router
```

---

### Traffic Flow

```
User → Edge-1 → [16301] → Firewall → [16302] → IDS → [16303] → WAN Opt → [16401] → Cloud

Step-by-step:
1.  Edge-1 receives packet (to 10.2.2.1)
2.  BGP matches next-hop → SR Policy SERVICE-CHAIN-1
3.  Edge-1 pushes segment stack: [16301, 16302, 16303, 16401]
4.  Firewall (16301) receives, pops 16301, processes traffic
5.  Firewall forwards with [16302, 16303, 16401]
6.  IDS (16302) receives, pops 16302, processes traffic
7.  IDS forwards with [16303, 16401]
8.  WAN Opt (16303) receives, pops 16303, compresses
9.  WAN Opt forwards with [16401]
10. Cloud-Edge (16401) receives, removes stack, forwards to cloud

Result:
  ✅ Traffic goes through all 3 services in order
  ✅ No additional protocol needed (just SR segment list)
  ✅ Centralized control (controller can modify service chain)
```

---

## Summary: All   Design Scenarios

| # | Scenario | Key Concepts Used |
|---|----------|------------------|
| 1 | Provider Network with SR-EPE, BGP-LS, PCE | BGP EPE, BGP-LS, PCE, ODN, SDN |
| 2 | Complete Migration from MPLS/LDP to SR | SR-LDP Interworking, 5-phase migration, rollback |
| 3 | Service Chaining with SR | Service Node SID, Binding SID, SR Policy |
| 4 | Flex-Algo for Low-Latency Paths | Flex-Algo 128, custom IGP metric |
| 5 | Seamless MPLS Multi-Domain | BGP Prefix SID, ABR, multi-IGP |
