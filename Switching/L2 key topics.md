## LAYER 2 FUNDAMENTALS

*1. MAC Address Table (CAM Table)*
Content Addressable Memory stores MAC-to-port mappings. Switch learns source MAC on ingress, looks up destination MAC for forwarding decision. Unknown unicast, broadcast, and multicast are flooded out all ports in the VLAN except ingress. CAM table overflow (MAC flooding attack) forces unknown unicast flooding — mitigated by port security or dynamic ARP inspection.

*2. TCAM (Ternary Content Addressable Memory)*
Unlike CAM (binary match), TCAM supports three states — 0, 1, and wildcard (X). Used for ACLs, QoS, routing lookups. Lookups happen in hardware at line rate. TCAM is expensive and finite — exhaustion causes software fallback, which is catastrophic for performance. CCDE-level concern is TCAM carving and partition sizing on platforms.

*3. Spanning Tree Protocol (STP) — IEEE 802.1D*
Prevents Layer 2 loops by electing a Root Bridge (lowest Bridge ID = priority + MAC), computing shortest path tree, and blocking redundant ports. Port states: Blocking → Listening → Learning → Forwarding. Convergence is slow (30–50 seconds default). Root bridge election, port cost calculation, and tie-breaking (port priority, port ID) are critical to understand deeply.

*4. RSTP — Rapid Spanning Tree (802.1w)*
Reduces convergence to sub-second by introducing port roles (Root, Designated, Alternate, Backup) and link types (point-to-point, edge, shared). Uses proposal/agreement handshake instead of timer-based convergence. Alternate port pre-computed as backup to root port — instantly transitions on failure. CCDE must understand how RSTP interoperates with legacy 802.1D.

*5. MST — Multiple Spanning Tree (802.1s)*
Maps multiple VLANs to a limited number of spanning tree instances (MSTIs), reducing CPU/memory overhead vs PVST+ (one STP per VLAN). MST region concept — switches with same MST name, revision, and VLAN-to-instance mapping form a region. IST (Internal Spanning Tree, instance 0) represents the region to the outside world. CCDE must understand MST region boundary behavior and how CST interacts with IST.

*6. PVST+ and Rapid PVST+*
Cisco's per-VLAN spanning tree — one STP instance per VLAN, allowing load balancing by making different VLANs prefer different root bridges. High VLAN count means high CPU overhead — MST addresses this. Rapid PVST+ adds RSTP behavior per VLAN.

*7. STP Portfast, BPDU Guard, BPDU Filter, Root Guard, Loop Guard*
- *PortFast*: skips Listening/Learning, jumps directly to Forwarding — for host ports only
- *BPDU Guard*: shuts port if BPDU received on a PortFast port — prevents rogue switches
- *BPDU Filter*: suppresses BPDU sending/receiving — dangerous globally, use carefully
- *Root Guard*: prevents a port from becoming root port — protects root bridge placement
- *Loop Guard*: prevents alternate/root port from transitioning to designated forwarding when BPDUs stop — guards against unidirectional link failures

---

## VLAN & TRUNKING

*8. VLANs and 802.1Q Trunking*
VLANs segment broadcast domains logically. 802.1Q inserts a 4-byte tag into the Ethernet frame (TPID 0x8100 + PCP + DEI + VID). Native VLAN frames are sent untagged on 802.1Q trunks — native VLAN mismatch is a security risk (VLAN hopping). Extended VLANs (1006–4094) not stored in NVRAM on older platforms — VTP mode considerations apply.

*9. VTP (VLAN Trunking Protocol)*
Cisco proprietary — propagates VLAN database across trunk links. Modes: Server (create/modify/delete, propagate), Client (receive only), Transparent (forward but don't apply), Off. VTP revision number is dangerous — a higher-revision switch added to domain can wipe VLANs across all switches. VTPv3 adds Primary Server concept and MST/extended VLAN support. CCDE considers VTP risk vs. operational simplicity.

*10. QinQ (802.1ad) — Double Tagging*
Adds an outer S-Tag (service tag) over an existing customer C-Tag. Used in Metro Ethernet to tunnel customer VLANs through a provider network. Provider only looks at S-Tag; customer VLAN space is preserved independently per customer. TPID for S-Tag is 0x88A8. Understanding EtherType mismatches and MTU implications (extra 4 bytes) is critical.

*11. Private VLANs (PVLAN)*
Adds sub-segmentation within a VLAN. Types: Primary VLAN (container), Isolated VLAN (ports can only talk to promiscuous port, never to each other), Community VLAN (ports talk to each other and promiscuous, not to isolated). Promiscuous port (usually the gateway) can talk to all. Used heavily in hosting/DMZ environments to isolate tenants sharing a subnet.

---

## LINK AGGREGATION

*12. LACP (802.3ad / 802.1AX)*
Standard protocol to bundle multiple physical links into one logical port-channel. Modes: Active (initiates LACP), Passive (responds), On (static, no LACP). LACP PDUs exchanged to negotiate bundle membership. Key parameters: System ID, port priority, port key — must match on both sides. Fast/slow timers (1s/30s) affect failure detection speed. Max 8 active + 8 standby links per bundle.

*13. Port-Channel Hashing / Load Balancing*
Traffic is distributed across bundle members using a hash of frame fields — src/dst MAC, src/dst IP, src/dst port, or combinations. Hash is deterministic per flow (not per-packet) to avoid reordering. Polarization is a CCDE-level concern — if all upstream and downstream switches use the same hash algorithm, traffic can end up always using the same physical link.

*14. MLAG / vPC (covered earlier)*
Multi-chassis LAG splits port-channel across two switches for redundancy. Key CCDE design consideration: control plane is distributed but data plane is often centralized through peer-link, which can become a bottleneck if orphan ports or asymmetric flows cause excessive peer-link traversal.

---

## ADVANCED LAYER 2 FEATURES

*15. Storm Control*
Rate-limits broadcast, multicast, and unknown unicast traffic on a port to prevent storms from consuming all bandwidth. Configurable as % of bandwidth or absolute pps/bps thresholds. Action: drop traffic above threshold, or shutdown the port. CCDE concern: threshold tuning — too low causes legitimate traffic drops, too high allows storms to propagate before suppression kicks in.

*16. Unicast Flood Containment*
Unknown unicast flooding can be as damaging as broadcast storms in large L2 domains. Solutions: reducing MAC aging timer, configuring unicast flood limiting, or moving to VXLAN/EVPN which eliminates flooding via BGP control-plane MAC learning. Flooding behavior is often a key argument for eliminating large L2 domains at CCDE level.

*17. ARP and Proxy ARP*
ARP is L2 broadcast — scales poorly in large L2 domains. Proxy ARP lets a router answer ARP requests on behalf of remote hosts. In VXLAN/EVPN, ARP suppression is a key optimization — the VTEP answers ARP locally using its EVPN-learned MAC-IP database, eliminating flood across the fabric.

*18. Dynamic ARP Inspection (DAI)*
Validates ARP packets against a trusted DHCP snooping binding table. Blocks ARP spoofing/poisoning attacks. Trusted ports (uplinks to routers/DHCP servers) bypass inspection. Untrusted ports (host-facing) are inspected. Rate-limiting ARP on untrusted ports prevents ARP flood as a DoS vector.

*19. DHCP Snooping*
Builds a binding table of MAC, IP, VLAN, and port from DHCP exchanges. Blocks rogue DHCP servers by only allowing DHCP replies on trusted ports. Underpins DAI and IP Source Guard. Option 82 inserts relay agent info into DHCP packets — useful but can cause drops if downstream devices don't expect it.

*20. IP Source Guard*
Filters traffic based on DHCP snooping binding table — only allows frames matching a bound MAC+IP+port+VLAN combination. Prevents IP spoofing at the access layer. Requires DHCP snooping to be operational first.

---

## LAYER 2 MULTICAST

*21. IGMP Snooping*
Switch "snoops" on IGMP messages between hosts and routers to learn which ports have multicast group members, and forwards multicast only to those ports rather than flooding. IGMP Querier must exist in the VLAN — if no router is present, configure a snooping querier on the switch. Improper IGMP snooping config causes either multicast flooding or multicast drops.

*22. MLD Snooping*
IPv6 equivalent of IGMP snooping. Listens to MLD (Multicast Listener Discovery) messages. Critical in dual-stack environments because IPv6 is heavy on multicast (neighbor discovery uses multicast) — without MLD snooping, IPv6 multicast floods everywhere.

---

## VXLAN AND OVERLAY

*23. VXLAN Encapsulation*
L2 frame encapsulated in UDP (dst port 4789), IP, and outer Ethernet header. VTEP (VXLAN Tunnel Endpoint) performs encap/decap. 24-bit VNI (Virtual Network Identifier) supports 16M segments. MTU must account for 50-byte overhead (outer IP + UDP + VXLAN header) — typical requirement is 1550+ byte MTU on underlay.

*24. VTEP and Flood-and-Learn vs EVPN*
In flood-and-learn mode, unknown unicast/ARP is replicated to all VTEPs in the VNI (using multicast or ingress replication). EVPN (BGP control plane) eliminates this by distributing MAC and IP reachability via BGP Type-2 routes — VTEPs learn about remote MACs before traffic arrives, avoiding floods. CCDE strongly favors EVPN for scale and operational visibility.

*25. EVPN Route Types*
BGP EVPN (RFC 7432) defines several route types:

- *Type 1*: Ethernet Auto-Discovery (per-EVI and per-ES) — used for fast convergence and aliasing in multi-homing
- *Type 2*: MAC/IP Advertisement — carries MAC alone or MAC+IP binding
- *Type 3*: Inclusive Multicast Ethernet Tag — signals BUM (Broadcast, Unknown, Multicast) replication membership
- *Type 4*: Ethernet Segment Route — used for designated forwarder (DF) election in multi-homing
- *Type 5*: IP Prefix Route — extends EVPN to carry IP prefixes for L3 overlays

*26. Symmetric vs Asymmetric IRB in VXLAN/EVPN*
Integrated Routing and Bridging (IRB) is how VXLAN does inter-VLAN routing.
- *Asymmetric*: routing happens on ingress VTEP, egress VTEP bridges only. Egress must have the destination VLAN locally — requires all VLANs on all VTEPs (doesn't scale well).
- *Symmetric*: both ingress and egress VTEPs route. Traffic traverses an L3 VNI (routed VNI) between VTEPs. Only the source VLAN needed on source VTEP. Much more scalable — preferred design.

*27. VXLAN Multi-homing and Designated Forwarder*
When a host is multi-homed to two VTEPs (via EVPN Ethernet Segment), only one VTEP should forward BUM traffic per VNI to avoid duplication — this is the Designated Forwarder (DF). DF election uses EVPN Type-4 routes and a deterministic algorithm (modulo-based or preference-based). Non-DF VTEP still forwards known unicast but suppresses BUM forwarding.

---

## SWITCHING ARCHITECTURE

*28. Cut-Through vs Store-and-Forward Switching*
- *Store-and-Forward*: receives entire frame, checks FCS for errors, then forwards. Adds latency proportional to frame size but drops corrupted frames.
- *Cut-Through*: starts forwarding after reading destination MAC (after 14 bytes). Ultra-low latency but forwards corrupted frames. Used in HFT/latency-sensitive environments.
- *Fragment-Free*: hybrid — waits for 64 bytes (past collision fragment size) before forwarding. Compromise between latency and error filtering.

*29. Switch Fabric Architecture — Crossbar, VOQ*
High-end switches use crossbar fabric (non-blocking matrix connecting all input/output ports). Virtual Output Queuing (VOQ) places packets in per-destination queues on the ingress side to prevent Head-of-Line (HOL) blocking — without VOQ, a slow output port blocks all traffic queued behind it even for other destinations. CCDE-level: understanding HOL blocking, speedup ratios, and fabric oversubscription.

*30. Hierarchical QoS (HQoS) in Switching Context*
Switches apply QoS via TCAM-based classification (DSCP, CoS/802.1p, ACL match) and queue scheduling (WRR, DWRR, Strict Priority). 802.1p (CoS) is the L2 QoS marking — 3-bit field in 802.1Q tag (8 classes). End-to-end QoS requires consistent marking from access to core. In DC fabrics, DSCP is preserved across VXLAN encap (outer DSCP copied or mapped from inner) — a critical design detail.

*31. BFD (Bidirectional Forwarding Detection) in L2 Context*
Sub-second failure detection protocol running independently of routing protocols. Can run over port-channels to detect forwarding path failures faster than LACP timers or routing protocol hold-timers. In MLAG/vPC, BFD on the peer-keepalive path speeds up split-brain detection. Asynchronous and echo modes — echo mode sends packets through the forwarding path of the peer for more accurate detection.

*32. MACsec (802.1AE)*
Layer 2 encryption standard — encrypts and authenticates Ethernet frames hop-by-hop between directly connected devices. Uses GCM-AES-128/256. MACsec Key Agreement (MKA) protocol handles key exchange using 802.1X/EAP. Unlike IPsec (L3), MACsec protects L2 headers too and has near-zero latency overhead. Critical in campus and WAN contexts where physical taps are a threat. CCDE must understand hop-by-hop vs end-to-end security tradeoffs.

*33. 802.1X Port-Based Authentication*
NAC (Network Access Control) framework — supplicant (device), authenticator (switch port), and authentication server (RADIUS). Switch holds port in unauthorized state until RADIUS authenticates the device. Multi-auth, multi-domain (voice+data VLAN on same port), MAB (MAC Authentication Bypass for non-802.1X clients) are CCDE-level design decisions. Failure modes (RADIUS unreachable) require critical VLAN or fail-open/fail-closed policy design.

---

## DESIGN LEVEL (CCDE-SPECIFIC)

*34. Large L2 Domain Risk — Why to Avoid It*
Large broadcast domains = ARP storms, STP instability, unknown unicast floods, slow convergence, and fault domain blast radius. CCDE design principle: minimize L2 domain size, push routing as close to the edge as possible (L3 to the access or L3 to the host), and use VXLAN/EVPN to provide L2 adjacency only where truly required (VM mobility, clustering) without physically extending VLANs.

*35. Leaf-Spine Fabric Design*
Modern DC switching topology: every Leaf connects to every Spine (full mesh), no Leaf-to-Leaf or Spine-to-Spine links. Provides predictable, equal-cost paths (ECMP) between any two endpoints with maximum 2 hops. VXLAN/EVPN runs as overlay; IP underlay (OSPF/ISIS or eBGP) provides reachability between VTEPs. Spine is pure L3; Leaf does L2/L3 boundary (IRB). CCDE must size for oversubscription ratios, ECMP hashing, and failure domain isolation.



*36. Topology Change Notification (TCN) and Its Impact*
When a switch detects a port state change (port goes down or moves to forwarding), it sends a TCN BPDU upstream toward the root. Root acknowledges and sets TC bit in its BPDUs, flooding it to all switches. All switches react by reducing their MAC aging timer from 300s to Forward Delay (15s), causing massive MAC table flush and re-learning — this triggers a temporary flooding storm across the entire L2 domain. CCDE design concern: TCNs in large flat L2 domains cause widespread flooding. PortFast suppresses TCN generation on host ports since host link flaps shouldn't trigger network-wide MAC flushes. Every access port flap without PortFast causes a domain-wide MAC flood — a key argument for minimizing L2 domain size.

*37. Unidirectional Link Detection (UDLD)*
Optical or physical failures can cause a link to pass traffic in only one direction — STP and LACP may not detect this because they only need to receive BPDUs/PDUs. UDLD sends echo messages and expects them reflected back. If not received within timer threshold, port is err-disabled. Two modes:
- *Normal*: logs the error but doesn't shut the port
- *Aggressive*: retries 8 times then err-disables — recommended for fiber links

Without UDLD, a unidirectional link can cause STP to believe a port is forwarding bidirectionally while actually black-holing return traffic, or worse, creating an invisible loop. Loop Guard also addresses this but complements rather than replaces UDLD.

*38. STP Dispute Mechanism*
Introduced in RSTP — when a switch receives an inferior BPDU on a port it considers designated (i.e., it was sending superior BPDUs out that port), it detects a dispute and moves the port to discarding state. This handles scenarios where a neighbor is confused about port roles — prevents loops caused by STP role disagreements. Dispute is a defensive mechanism that adds resilience to RSTP without operator configuration.

*39. STP Root Bridge Placement Strategy*
Unplanned root bridge location is one of the most dangerous L2 design mistakes. Root should be the most central, highest-capacity switch in the L2 domain — typically the distribution or aggregation layer. Poor placement means traffic traverses suboptimal paths and TCN propagation is more disruptive. CCDE-level strategies: configure explicit priority (priority 4096 for primary, 8192 for secondary), use Root Guard on all non-core-facing ports to lock root placement, and ensure secondary root is configured for fast takeover. In modern DC designs, STP is essentially eliminated at the fabric level — but campus/enterprise still requires deliberate root placement.

---

## RESILIENCY AND RING PROTOCOLS

*40. G.8032 — Ethernet Ring Protection Switching (ERPS)*
ITU-T standard for sub-50ms L2 resiliency in ring topologies — directly competes with STP for Metro Ethernet ring designs. Designates one link as RPL (Ring Protection Link), kept in blocking state under normal operation. On failure, blocked link opens within 50ms — much faster than STP. Uses R-APS (Ring Automatic Protection Switching) messages to coordinate state. CCDE consideration: G.8032 is specifically optimized for ring topologies where STP is inefficient. G.8032v2 adds multi-ring and ladder topologies. Commonly deployed in Metro Ethernet and SP access networks.

*41. REP — Resilient Ethernet Protocol (Cisco Proprietary)*
Cisco's alternative to STP for ring or linear segment topologies. Creates a "segment" of ports with one primary and one secondary edge port. One port blocked (No-Neighbor) under normal operation. On failure, unblocks within ~50ms. Unlike STP, REP has no root bridge concept — blocking is segment-local. PREEMPT delay allows administrator to restore preferred topology after recovery without immediate flapping. Used extensively in Industrial Ethernet and Metro access rings.

*42. FlexLinks*
Simple Cisco feature providing active/standby link redundancy without STP — one port active, one on standby. Standby takes over in ~1 second on active failure. Much simpler than full STP but offers no load balancing. Suitable for simple edge redundancy where STP complexity isn't warranted. Preemption configurable for preferred link. CCDE considers FlexLinks vs REP vs ERPS based on topology, vendor, and convergence requirements.

---

## CARRIER AND METRO ETHERNET

*43. VPLS — Virtual Private LAN Service*
L2 VPN technology that creates a multipoint Ethernet service over MPLS. Each PE (Provider Edge) maintains a pseudowire mesh to all other PEs in the VPLS instance — simulates a shared LAN switch for customer sites. MAC learning happens per pseudowire. Split horizon rule prevents loops — a frame received on a pseudowire is never forwarded to another pseudowire (only to local attachment circuits). Scales poorly due to full-mesh pseudowire requirement — N*(N-1)/2 pseudowires for N PEs. H-VPLS addresses this with a hierarchy of MTUs (Multi-Tenant Units) and PE devices.

*44. H-VPLS — Hierarchical VPLS*
Adds a spoke-hub model to VPLS: spoke PEs connect via a single pseudowire to a hub PE, which connects to the full-mesh core. Reduces pseudowire count dramatically. Spoke PEs do local switching; hub PEs handle core distribution. Trade-off: suboptimal traffic paths (spoke-to-spoke via hub) and hub becomes a potential bottleneck. CCDE compares H-VPLS vs EVPN for modern Metro/DCI deployments — EVPN provides better scale and active-active multihoming.

*45. PBB — Provider Backbone Bridging (802.1ah)*
Adds a MAC-in-MAC encapsulation layer — customer MAC (C-MAC) encapsulated inside provider backbone MAC (B-MAC). Allows provider network to forward based only on B-MACs, keeping C-MACs invisible to the core — dramatically limits MAC table scale in provider core. B-VID (Backbone VLAN) + I-SID (Instance Service Identifier, 24-bit) identify services. PBB-EVPN combines 802.1ah encapsulation with BGP EVPN control plane — addresses MAC scalability at provider scale while adding EVPN's multi-homing capabilities.

*46. OTV — Overlay Transport Virtualization (Cisco)*
Cisco-proprietary L2 extension technology for Data Center Interconnect — specifically designed to stretch VLANs between geographically separated DCs over an IP transport. Key differentiators from VXLAN: built-in MAC routing (doesn't flood unknown unicast across WAN — learns remotely via OTV control protocol), automatic site detection prevents routing STP BPDUs across WAN (site VLAN concept), and native multicast or unicast adjacency modes. CCDE consideration: OTV is operationally simpler for DCI than VXLAN but Cisco-proprietary. EVPN/VXLAN has largely superseded it in modern designs.

*47. MEF Carrier Ethernet Services*
MEF defines standardized Ethernet service types:
- *E-Line (EPL/EVPL)*: point-to-point — Ethernet Private Line or Virtual Private Line
- *E-LAN (EPLAN/EVPLAN)*: multipoint-to-multipoint — full mesh connectivity
- *E-Tree (EPTree/EVPTree)*: rooted multipoint — root site can communicate with all leaf sites, but leaves cannot communicate with each other (used for broadband access)
- *E-Access*: wholesale access service

MEF attributes: CIR (Committed Information Rate), EIR (Excess Information Rate), CBS/EBS (burst sizes), color mode. CCDE must understand MEF service mapping to VPLS, EVPN, or pseudowire implementations, and how SLA attributes are enforced via QoS mechanisms.

---

## DATA CENTER BRIDGING (LOSSLESS ETHERNET)

*48. Priority Flow Control — PFC (802.1Qbb)*
Per-priority pause mechanism — extends 802.3x Ethernet PAUSE to per-CoS granularity (8 traffic classes). Allows specific traffic classes (e.g., storage/RoCE) to be paused without affecting other classes (e.g., TCP). Essential for FCoE and RoCE which cannot tolerate packet loss. PAUSE frames are sent upstream to the transmitter when buffers fill. CCDE design concern: PFC can cause Head-of-Line blocking if pause propagates too far — "PFC deadlock" is a real operational issue in DCB fabrics. Careful traffic class isolation is required.

*49. ETS — Enhanced Transmission Selection (802.1Qaz)*
Complements PFC — allocates bandwidth percentages to traffic classes on an egress port. Traffic classes assigned to groups: Strict Priority group (always served first), ETS group (bandwidth shared proportionally). Allows lossless (storage) and lossy (TCP) traffic to share the same physical link with guaranteed minimum bandwidth for each. Works alongside PFC to create converged network fabric.

*50. DCBX — Data Center Bridging Exchange*
Extension of LLDP — used to negotiate and exchange DCB capabilities (PFC, ETS, application priority) between adjacent DCB-capable switches and NICs. Ensures consistent PFC/ETS configuration end-to-end without manual per-hop configuration. If DCBX detects a mismatch, it can err-disable the port to prevent misconfigured lossless behavior. CCDE must design DCBX deployment carefully — mixing DCBX-enabled and non-DCBX switches can cause inconsistent pause/priority behavior.

*51. FCoE — Fibre Channel over Ethernet*
Encapsulates Fibre Channel frames in Ethernet (FCoE frames, EtherType 0x8906). Requires lossless Ethernet (PFC) because FC has no retransmission at L2. FCoE Forwarder (FCF) performs FC switching functions. VN_Port (virtual FC port on server NIC) connects to VF_Port (on FCF). Eliminates separate FC HBA and cabling — converged network adapter (CNA) handles both. CCDE consideration: FCoE adds significant complexity; many modern DC designs prefer iSCSI or NVMe-oF over RDMA/RoCE instead.

*52. RoCE — RDMA over Converged Ethernet*
RDMA (Remote Direct Memory Access) bypasses OS kernel — allows direct memory-to-memory transfers between servers with near-zero CPU overhead and microsecond latency. RoCEv1 runs over L2 (requires lossless, PFC). RoCEv2 encapsulates in UDP/IP — works over routed networks and benefits from ECMP. Critical for HPC, AI/ML training (GPU-to-GPU communication), and NVMe-oF storage. CCDE must design PFC and ECN (Explicit Congestion Notification) carefully for RoCE — ECN marks packets before buffers fill, signaling senders to reduce rate without dropping, complementing PFC.

---

## MONITORING, VISIBILITY, AND OAM

*53. SPAN / RSPAN / ERSPAN*
- *SPAN (Switched Port Analyzer)*: mirrors traffic from source port/VLAN to a local destination port for capture. Ingress, egress, or both. No impact to forwarded traffic.
- *RSPAN (Remote SPAN)*: extends SPAN across switches using a dedicated RSPAN VLAN — mirrored traffic flooded in that VLAN to the destination switch. Limited to L2 domain.
- *ERSPAN (Encapsulated RSPAN)*: encapsulates mirrored traffic in GRE — can traverse routed networks and reach analyzers anywhere in the IP network. ERSPAN Type III adds hardware timestamps.

CCDE consideration: SPAN introduces a copy overhead on the switch CPU/ASIC — high-volume SPAN can impact forwarding performance on some platforms. Proper sizing and filter ACLs on SPAN sources are important design decisions.

*54. Ethernet OAM — CFM (802.1ag) and Y.1731*
*CFM (Connectivity Fault Management)*: defines Maintenance Domains (MD), Maintenance Associations (MA), and MEPs (Maintenance End Points) / MIPs (Maintenance Intermediate Points). Key functions: Continuity Check Messages (CCM) for liveness detection, Link Trace (LT) for path tracing (like traceroute at L2), and Loopback (LB) for connectivity testing (like ping at L2).

*Y.1731* extends CFM with performance monitoring: Frame Delay (FD), Frame Delay Variation (FDV/jitter), Frame Loss Ratio (FLR), and Throughput measurement. SLA monitoring for Carrier Ethernet services uses Y.1731 probes. CCDE must understand how OAM domain levels (0–7) separate operator from customer monitoring domains.

*55. sFlow and NetFlow in Switching Context*
- *sFlow*: hardware-based — samples 1-in-N packets, exports header+metadata to collector. Near-zero performance impact. Loses precision but scales to high-speed links.
- *NetFlow*: flow-based — tracks active flows, exports records on expiry. Higher detail but more memory-intensive on switch ASICs.

CCDE uses both for capacity planning, anomaly detection, and traffic matrix understanding. In VXLAN fabrics, inner header export (tenant-level visibility) requires decap before sampling or use of overlay-aware flow export. Telemetry streaming (gRPC/gNMI-based) is replacing SNMP/NetFlow in modern DC for real-time buffer monitoring and congestion detection.

---

## HIGH AVAILABILITY AND REDUNDANCY

*56. VSS — Virtual Switching System (Cisco)*
Combines two Catalyst switches into a single logical switch with one control plane. Unlike vPC/MLAG (separate control planes), VSS has a single management interface, single routing process, and single STP instance. Virtual Switch Link (VSL) carries control plane and data plane traffic between chassis. Active/Standby supervisor model — Active handles all control plane functions. CCDE consideration: VSS simplifies management and eliminates STP between the pair, but the VSL becomes a critical single point — its failure is more impactful than vPC peer-link failure because control plane also traverses it.

*57. StackWise and StackWise Virtual*
*StackWise*: physical stacking — switches connected via dedicated stack cables forming a ring topology, appearing as a single switch. Shared control plane, single management IP. Stack ring provides full bandwidth between members. Member numbering and priority determine active master.
*StackWise Virtual*: logical equivalent of VSS for Catalyst 9000 series — two switches connected via StackWise Virtual Links (SVL), appearing as one switch. Eliminates STP between the pair; simplifies L2/L3 design. CCDE compares StackWise Virtual vs vPC: SWV is simpler (single control plane) but creates a tighter fault domain.

*58. NSF/SSO — Non-Stop Forwarding / Stateful Switchover*
Critical for high availability on modular chassis with redundant supervisors:
- *SSO (Stateful Switchover)*: standby supervisor maintains synchronized state (interface states, L2 tables, CEF) — on active failure, standby takes over in milliseconds with minimal traffic disruption
- *NSF (Non-Stop Forwarding)*: allows the forwarding plane to continue operating during a routing protocol restart. Requires NSF-aware neighbors. BGP/OSPF/ISIS carry NSF capability negotiation.

CCDE must understand that NSF/SSO protects against supervisor failure but not line card failure. ISSU (In-Service Software Upgrade) leverages SSO for hitless software upgrades.

*59. ISSU — In-Service Software Upgrade*
Allows software upgrade on a switch without traffic disruption — leverages SSO between old and new software versions running simultaneously during the upgrade window. Active supervisor runs new software; standby runs old, then they synchronize and switchover. Not all software versions support ISSU between them (version compatibility matrix). CCDE design implication: ISSU requires hardware platforms with redundant supervisors and constrains upgrade paths — skipping major versions may break ISSU compatibility.

*60. GR — Graceful Restart in Switching Context*
After a control plane restart (supervisor switchover, process crash), a BGP/OSPF/ISIS peer that supports GR maintains forwarding entries for the restarting speaker for a defined stale timer period — preventing route withdrawal and traffic black-holing during convergence. GR Helper mode (on the peer) vs GR-capable (on the restarting device) must both be configured. CCDE must understand the risk: if GR timer expires before convergence, all stale routes are withdrawn simultaneously — potentially worse than a clean withdrawal.

---

## SECURITY — ADVANCED

*61. VLAN Hopping Attacks*
Two methods:
- *Switch Spoofing*: attacker enables DTP on their NIC, negotiates a trunk with the switch, and gains access to all VLANs — mitigated by disabling DTP (switchport nonegotiate) and explicitly setting access/trunk mode
- *Double Tagging*: attacker sends a double-tagged frame where outer tag matches native VLAN — outer tag stripped by first switch, inner tag carried to destination VLAN. Mitigated by changing native VLAN away from VLAN 1, using a dedicated non-routable VLAN as native, or tagging native VLAN explicitly.

CCDE security design: standardize on explicit trunk configuration, disable all unused ports, place in a dead VLAN, and never use VLAN 1 as native VLAN in production.

*62. Control Plane Policing (CoPP)*
Protects the switch CPU from being overwhelmed by malicious or excessive control plane traffic (ARP floods, routing protocol messages, ICMP). Defines policy-map applied to the control plane — classifies traffic types, rate-limits each class separately. Without CoPP, a MAC flood or ARP storm can crash the control plane even while the data plane continues forwarding. CCDE must design CoPP tiers: critical (routing protocols — high CIR, no drop), important (SNMP, SSH — medium CIR), normal (ICMP, ARP — low CIR), default (everything else — aggressive policing).

*63. Port Security — Advanced*
Beyond basic max-MAC limiting: sticky MAC learning persists dynamically learned MACs to running config, surviving port bounces without full manual configuration. Violation modes: Protect (drop silently), Restrict (drop + SNMP trap), Shutdown (err-disable port). CCDE consideration: port security doesn't scale well in dynamic environments (VMs moving, BYOD) — replaced by 802.1X + MAB + dynamic VLAN assignment in modern campus designs. Port security still appropriate for static industrial or kiosk environments.

*64. DHCP Starvation and Rogue DHCP — Attack Vectors*
DHCP starvation: attacker sends DHCP Discover with thousands of spoofed MACs, exhausting the DHCP pool — legitimate hosts get no IP. Rogue DHCP: attacker runs their own DHCP server, handing out bogus gateway/DNS to redirect traffic. DHCP snooping with rate-limiting on untrusted ports mitigates both. CCDE must design DHCP snooping as a baseline on all access switches — it underpins DAI and IP Source Guard, making it a foundational security building block, not just an optional add-on.

---

## L2 TUNNELING AND EXTENSION

*65. L2 Protocol Tunneling (L2PT)*
When customer sites connect through a provider network, L2 protocols (STP, CDP, LACP, VTP) are normally terminated at the PE — customers can't run end-to-end STP or LACP across the provider. L2PT encapsulates customer protocol PDUs into a proprietary multicast MAC address, tunnels them across the provider network, and decapsulates at the far-end PE — allowing customer protocols to span the WAN as if directly connected. CCDE must understand interaction between L2PT and provider STP — provider must not process customer BPDUs or loops can result.

*66. EVPN-VPWS — Virtual Private Wire Service over EVPN*
Point-to-point L2 circuits (pseudowires) signaled via BGP EVPN (RFC 8214) instead of LDP. Uses EVPN Route Type 1 for autodiscovery and single-active/all-active multihoming. Advantages over LDP-based pseudowires: consistent BGP operational model with L3VPN and EVPN ELAN, faster convergence via BGP, and integrated multihoming support without additional protocols. CCDE compares EVPN-VPWS vs VPLS for point-to-point vs multipoint L2 VPN requirements.

*67. SRv6 for L2 Services*
Segment Routing over IPv6 enables L2 VPN services (VPWS and EVPN ELAN) using SRv6 SID functions — specifically the DX2 (Decapsulate and Cross-connect to L2 interface) and DT2U/DT2M (Decapsulate and L2 table lookup) behaviors. Eliminates MPLS label stack — uses IPv6 extension headers (Segment Routing Header). Simplifies network by unifying underlay (SRv6) and overlay (EVPN SID functions) into a single IPv6 packet. CCDE evaluates SRv6 vs MPLS/VXLAN for unified DC and WAN fabric designs.

---

## TIMING AND SYNCHRONIZATION

*68. PTP — Precision Time Protocol (IEEE 1588)*
Provides sub-microsecond clock synchronization across networks — critical for financial trading, industrial control, 5G fronthaul, and telecom networks. PTP Master clock distributes time; PTP Slaves synchronize. Boundary Clocks (BC) terminate and re-originate PTP on each hop — correcting for switch latency. Transparent Clocks (TC) add residence time correction to PTP packets as they traverse — more accurate than BC. End-to-end TC vs Peer-to-peer TC differ in delay measurement scope. CCDE must specify PTP hardware timestamping support — software timestamping introduces too much jitter for telecom-grade accuracy.

*69. SyncE — Synchronous Ethernet (ITU-T G.8261)*
Distributes frequency synchronization (not time of day) over Ethernet physical layer — recovers clock from received bit stream just like SDH/SONET. Complements PTP: SyncE provides stable frequency reference; PTP adds phase/time-of-day. ESMC (Ethernet Synchronization Messaging Channel) communicates clock quality level (QL) via SSM (Synchronization Status Messages) in slow-protocol Ethernet frames. CCDE designs telecom and 5G transport networks using SyncE + PTP combined for full phase and frequency synchronization.

---

## FABRIC AND ARCHITECTURE

*70. Fabric Extender (FEX) — Cisco Nexus*
FEX (e.g., Nexus 2000) acts as a remote line card of the parent Nexus switch — no local switching, all frames forwarded to parent for all decisions. Managed as part of parent switch — single point of management. Simplifies access layer but creates dependency on parent. FEX ports cannot form port-channels between two FEX units attached to different parents — a design constraint. CCDE evaluates FEX vs ToR (Top of Rack) switch model: FEX reduces opex but limits flexibility and creates a potential bottleneck on the FEX uplink.

*71. ACI — Application Centric Infrastructure (Cisco)*
Policy-based DC networking — abstracts network configuration into application policy objects: Tenants, Application Profiles, EPGs (Endpoint Groups), Contracts, BDs (Bridge Domains), and VRFs. APIC (controller) pushes policy to Nexus 9000 leaf/spine. EPG defines a group of endpoints with common policy — replaces VLAN-centric design. Contracts define permitted communication between EPGs. ACI uses VXLAN with IS-IS underlay and COOP (Council of Oracle Protocol) for endpoint distribution. CCDE evaluates ACI vs standard VXLAN/EVPN: ACI offers rich policy automation but tight vendor lock-in and steep operational learning curve.

*72. TRILL and SPB — Alternatives to STP (Historical CCDE Context)*
*TRILL (Transparent Interconnection of Lots of Links)*: uses IS-IS as control plane, encapsulates Ethernet frames with a TRILL header — allows optimal multipath routing in L2 fabrics without STP. RBridges (Routing Bridges) act as TRILL-capable switches. Never saw wide adoption due to proprietary implementations and VXLAN/EVPN emerging.

*SPB (Shortest Path Bridging, 802.1aq)*: also uses IS-IS, provides ECMP in L2 fabrics, preserves standard Ethernet semantics. Used by Avaya/Extreme in enterprise and campus. CCDE must understand why VXLAN/EVPN won — standard IP underlay, hardware support ubiquity, and better cloud integration.

*73. Leaf-Spine Design — Oversubscription and ECMP*
At CCDE level, leaf-spine sizing involves: oversubscription ratio (downlink bandwidth / uplink bandwidth — 3:1 to 6:1 common for compute, 1:1 for storage), ECMP hash entropy (using 5-tuple for good distribution — VXLAN adds entropy via outer UDP src port derived from inner 5-tuple), and spine redundancy (at least 2 spines for N+1, 4 for N+2). Pod design adds another hierarchy layer (multiple leaf-spine pods connected via super-spine) for very large fabrics. CCDE must balance cost (more spines = more opex) vs. resiliency and non-blocking design.

*74. Intent-Based Networking (IBN) and Model-Driven Telemetry*
Modern CCDE-level concern: traditional CLI/SNMP management doesn't scale for modern DC and campus. IBN platforms (Cisco Catalyst Center, Apstra) take high-level intent ("this EPG can talk to that EPG") and translate to device configuration automatically. Model-driven telemetry (gNMI/gRPC Dial-Out) streams structured operational data (buffer utilization, interface counters, BGP state) at sub-second intervals — far superior to SNMP polling for congestion detection. YANG models define the data schema. CCDE must understand northbound (controller to operator) and southbound (controller to device) interfaces and how they impact operational design.

*75. ECN — Explicit Congestion Notification in L2/L3 Switching Context*
ECN (RFC 3168) marks packets (sets CE bits in IP header) when switch buffers exceed threshold — signals endpoints to reduce sending rate before actual drops occur. Critical for RoCE (lossless storage) and latency-sensitive applications. Works in conjunction with PFC — ECN prevents buffer buildup before PFC PAUSE frames are needed, reducing PFC storm risk. WRED (Weighted Random Early Detection) is the drop-based precursor — randomly drops packets above threshold to signal congestion. CCDE designs ECN thresholds carefully — too aggressive and throughput suffers; too conservative and PFC kicks in more than desired.

*76. VLAN Access Control Lists (VACLs)*
VACLs filter traffic within a VLAN — unlike router ACLs (which filter between VLANs), VACLs apply to all traffic within a single VLAN including switch-to-switch and host-to-host at L2. Can match on MAC, IP, and L4 ports depending on platform. Actions: forward, drop, or redirect (useful for SPAN/capture). CCDE uses VACLs for intra-VLAN security — particularly in shared segments where DAI/IPSG isn't granular enough, or for redirecting specific traffic to inline security appliances.

*77. Multicast in L2 Switching — MVR and IGMP Proxy*
*MVR (Multicast VLAN Registration)*: allows multicast streams from a dedicated multicast VLAN to be distributed to subscriber ports in different VLANs — avoids replicating multicast streams per VLAN. Used in IPTV deployments where one stream is served to thousands of subscribers across many access VLANs without duplicating the stream per VLAN.
*IGMP Proxy*: aggregates IGMP memberships from downstream hosts, sending a single IGMP Join upstream — reduces IGMP report flooding toward the router. Reduces multicast state in the network for environments with many receivers and few sources.

*78. MACSec Key Agreement (MKA) and 802.1X Integration*
MKA (MACsec Key Agreement protocol) uses EAP-TLS or pre-shared keys to derive and refresh MACsec session keys dynamically — preventing replay and key exhaustion attacks. MKA peers exchange MKPDU (MKA Protocol Data Units) to elect a Key Server and distribute SAKs (Secure Association Keys). Integration with 802.1X: 802.1X authenticates the endpoint, EAP exchange derives keys passed to MKA for MACsec session setup — seamless encrypted access without separate key management. CCDE evaluates per-hop MACsec vs end-to-end IPsec: MACsec has near-zero latency overhead and protects L2 headers but requires MACsec-capable hardware on every hop.

*79. Buffering Architecture — Shallow vs Deep Buffer Switches*
Critical CCDE design decision: shallow buffer switches (commodity silicon — Broadcom Trident) have small on-chip buffers (~12–16MB shared), suitable for low-latency DC spine/leaf where traffic is mostly short-burst. Deep buffer switches (merchant or custom silicon — Broadcom Jericho, Cisco custom ASIC) have large external DRAM buffers (GBs), absorbing large traffic bursts without drops — essential for WAN aggregation, traffic from bursty storage workloads, or elephant flows. CCDE must match buffer depth to application profile: HFT and AI/ML training (RoCE) need different buffer strategies. Oversubscription ratio and traffic profile determine buffer requirements.

*80. Network Virtualization Overlays (NVO) Comparison*
At CCDE level: VXLAN, NVGRE, GENEVE, and STT are all NVO encapsulations:
- *VXLAN*: UDP-based, 50B overhead, 24-bit VNI, hardware-supported widely, IETF standard
- *NVGRE*: GRE-based (Microsoft), 42B overhead, 24-bit VSID — limited ECMP (GRE has no L4 header for hashing)
- *GENEVE*: UDP-based, variable-length TLV options, extensible — designed to replace VXLAN and NVGRE. Used by Open vSwitch and some NSX versions
- *STT (Stateless Transport Tunneling)*: mimics TCP header for hardware offload compatibility — used in early NSX
---