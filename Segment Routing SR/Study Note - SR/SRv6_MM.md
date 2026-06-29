

```markdown
# SRv6 Memory Map (Bullet Hierarchy for Obsidian)

## SRv6 Overall
- Segment Routing over IPv6
  - Uses IPv6 data plane + SRH instead of MPLS labels
  - Ingress router decides end-to-end path
  - Core routers have no per-flow state

## SRv6 SID (Segment Identifier)
- 128-bit IPv6 address that encodes location + function
- Structure
  - Locator (routing prefix)
    - Routable IPv6 prefix
    - Aggregated in IGP
    - Determines which node processes the packet
  - Function (instruction)
    - Defines behavior: End, End.X, End.DX4, End.DT46, etc.
    - Executed at the node identified by locator
  - Arguments (optional)
    - VRF ID, service ID, slice ID, flow ID
    - Additional context for function

## SRv6 Behaviors (Network Programming RFC 8986)
- End
  - Basic endpoint forwarding
  - Standard IPv6 destination behavior
- End.X
  - Adjacency behavior
  - Forward via specific interface/neighbor
  - Hard traffic engineering (strict path)
- End.DX4
  - IPv4 L3VPN endpoint
  - VRF lookup for IPv4 prefixes
- End.DX6
  - IPv6 L3VPN endpoint
  - VRF lookup for IPv6 prefixes
- End.DT46
  - Dual-stack L3VPN endpoint
  - Terminates combined IPv4/IPv6 L3VPN
- End.DX2 / End.DX2V
  - L2VPN / EVPN endpoints
  - VPWS, VPLS, EVPN flexible cross-connect
- Behavior library
  - Enables network programming
  - SIDs = instructions, not just addresses
  - Supports service chaining and slicing

## SRH (Segment Routing Header)
- IPv6 Extension Header Type 4
- Contains
  - Segment List (ordered SIDs)
  - Segments Left (how many SIDs remain)
  - Last Entry (index of last segment)
  - Flags, Tag
- Operation
  - IPv6 Destination = current active SID
  - SRH = full path
  - At each SID:
    - Execute behavior
    - Decrement Segments Left
    - Update IPv6 Destination to next SID
    - Forward

## uSID (Micro-SID)
- Compressed SID format
- Purpose
  - Reduce header overhead
  - Enable deeper segment stacks
- Mechanism
  - Pack multiple 16/32-bit micro-SIDs in one IPv6 address
  - Hardware shifts and processes micro-SIDs
- Benefits
  - Closer to MPLS label size
  - Better for high-scale cores, AI, 5G backbones

## SRv6 Traffic Engineering (TE)
- Path encoded as segment list in SRH
- Components
  - Headend node (steers traffic)
  - Segment List (explicit path)
  - Controller / PCE (computes optimal paths)
- Constraints
  - Latency
  - Bandwidth
  - Affinity / admin groups
- Features
  - No RSVP-TE signaling
  - Stateless core
  - TI-LFA for fast reroute
  - Integration with BGP-LS for topology

## SRv6 Services
- L3VPN
  - End.DX4, End.DX6, End.DT46
  - Replaces MPLS VPN labels with SIDs
  - BGP distributes VPN routes + service SIDs
- EVPN (L2/L3)
  - End.DX2, End.DX2V
  - Type-2 (MAC), Type-3 (Multicast), Type-5 (IP Prefix)
  - Multi-homing with ESI + DF election
- Service Chaining
  - Sequence of service SIDs
  - Firewall → IDS → Load Balancer → VPN
  - No separate NSH or MPLS needed in many designs
- Network Slicing
  - Different SIDs / behaviors per tenant or service
  - Low-latency slice, high-bandwidth slice, etc.

## SRv6 Control Plane
- IGP (IS-IS / OSPFv3)
  - Distributes node locators
  - Distributes SR capabilities (SRGB-like, functions)
- BGP-LS
  - Collects topology from IGP
  - Sends to SDN controller / PCE
- BGP SRv6
  - Distributes SRv6 policies
  - Distributes L3VPN/EVPN routes with SRv6 SIDs
- PCE / SDN Controller
  - Computes SRv6 TE paths
  - Programs SRv6 Policies via PCEP or BGP

## SRv6 Interworking
- SRv6 ↔ SR-MPLS
  - Gateway maps SIDs ↔ labels
  - Supports gradual migration
- SRv6 ↔ MPLS
  - Interworking for transport and service
  - VPN over SRv6 core ↔ VPN over MPLS core
- Mixed deployments
  - Some regions SRv6, some SR-MPLS
  - Interworking at domain boundaries

## SRv6 Deployment Scenarios
- 5G Transport
  - Network slicing
  - Mobile backhaul
  - UPF anchoring
- Cloud & Data Center
  - Multi-tenant connectivity
  - EVPN + SRv6 for microservices
  - Service insertion (firewall, LB)
- AI / HPC Fabrics
  - GPU cluster interconnect
  - Low-latency TE paths
  - Fast failure recovery with TI-LFA
- Service Provider Core
  - IPv6-only backbone
  - L3VPN, EVPN over SRv6
  - Gradual migration from MPLS

## SRv6 vs SR-MPLS
- SR-MPLS
  - Uses MPLS label stack
  - Existing MPLS infrastructure
  - 20-bit labels, low overhead
  - Familiar operations
- SRv6
  - Uses IPv6 + SRH
  - Native IPv6, future-proof
  - 128-bit SIDs or uSID
  - Network programming, functions
- Decision factors
  - Existing network: MPLS → SR-MPLS
  - New IPv6 network: SRv6
  - 5G, cloud, AI: SRv6 preferred
  - Migration strategy: interworking at boundaries

## SRv6 Advantages
- Removes LDP, RSVP-TE, BGP-LU
- Single IPv6 data plane
- Native network programming
- Built-in support for services (L3VPN, EVPN, chaining)
- Aligns with 5G, cloud, AI requirements
- Better telemetry and flow-level conditioning

## SRv6 Challenges
- Larger header (128-bit SID)
  - Mitigated by uSID
- Hardware requirements
  - SRH parsing, large SID tables
- Operational complexity
  - New behavior model
  - New tools and troubleshooting methods
- Mixed environments
  - Planning for interworking
  - Gradual migration instead of big-bang

## SRv6 Tie to TI-LFA, Telemetry, Flow Label
- TI-LFA (Fast Reroute)
  - Pre-computed backup segments
  - Sub-50ms failover
- Telemetry
  - Flow label for per-flow entropy
  - MORP, IOAM, in-band OAM
- Load balancing
  - Flow label used by ECMP
  - Better distribution than raw IPv6 5-tuple in some hardware
```

