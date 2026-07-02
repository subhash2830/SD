

## Prefix SID

### Definition

A Prefix SID is the SR identifier attached to a prefix, usually a loopback, that the ingress uses to reach the node or prefix through the SR label stack.[^1][^2]

### Why it exists

It solves the problem of scalable transport to a destination without LDP per-prefix label distribution or RSVP-TE per-tunnel signaling.[^2][^1]

### How it works

The prefix is advertised with a SID index, and the network converts that index into a usable MPLS label using the SRGB. The same prefix can also be tied to an SR algorithm, such as the normal SPF behavior or a strict SPF behavior, depending on the protocol and design.[^1][^2]

### Real-world use case

Used to reach loopbacks in an SR core, reach service anchors, and steer traffic to an ingress/egress node in DC or SP transport.[^2][^1]

### Failure scenario

If the SRGB is inconsistent across the domain, the same SID index maps to different labels and forwarding breaks even though routing still looks up.[^1][^2]

### Design insight

Treat Prefix SID as the global node identity of SR. Keep the SRGB consistent, keep the SID plan simple, and use algorithms only when you truly need different forwarding intent.[^2][^1]

### Interview-ready answer

Prefix SID is the SR label identity of a prefix. The ingress resolves it through the SRGB, so traffic can be steered without LDP or RSVP-TE state.[^1][^2]

## SRGB

### Definition

The Segment Routing Global Block is the global label range from which SID indexes are translated into MPLS labels.[^2][^1]

### Why it exists

It creates a common label mapping model so every router interprets a global SID index the same way.[^1][^2]

### How it works

A router advertises its SRGB base and range, and all other routers calculate label = base + SID index. If the base is 16000 and the index is 1, the resulting label is 16001.[^2][^1]

### Real-world use case

SP cores, inter-area SR designs, and SR-enabled DC fabrics that need a uniform label namespace.[^1][^2]

### Failure scenario

A mismatched SRGB during migration or between vendors can blackhole traffic because label meaning is no longer uniform.[^2][^1]

### Design insight

SRGB stability is a design invariant. Change it only with a controlled migration and label validation plan.[^1][^2]

### Interview-ready answer

SRGB is the mapping window for SR labels. It makes SID indexes portable across the network, but only if the block is consistent everywhere.[^2][^1]

## ISIS Router Capabilities

### Definition

ISIS Router Capabilities advertise SR support, SRGB, algorithms, max SID depth, and related behavior through the IGP.[^1]

### Why it exists

Other routers need to know whether the node can support SR forwarding and how much label-stack depth it can handle.[^1]

### How it works

The TLV carries SR capability sub-TLVs, algorithm info, and max SID depth. The router capability RID can be derived from MPLS TE RID, ISIS RID, loopback, or physical interface depending on what is configured.[^1]

### Real-world use case

Used in SR core discovery and path computation so ingress routers know which nodes can really support a chosen segment list.[^1]

### Failure scenario

If the stack depth exceeds hardware limits, the path may be computed but not forwardable in the data plane.[^1]

### Design insight

Capability advertisement is only half the story; hardware forwarding limits are the true constraint. Keep repair and TE stacks short whenever possible.[^1]

### Interview-ready answer

ISIS Router Capabilities tell the domain what SR features a node supports and how much label stack it can process. That information is essential for safe SR design.[^1]

## OSPF SR LSAs

### Definition

OSPF carries SR information in opaque LSAs that extend the protocol without changing SPF behavior.[^2]

### Why it exists

OSPF needed a way to advertise SR metadata separately from classic route calculation.[^2]

### How it works

The lab notes describe SR-related data in type 10 opaque LSAs, with extended prefixes and links carrying prefix SIDs and adjacency SIDs respectively.[^2]

### Real-world use case

OSPF-based SR underlays in SP and DC environments where SR is desired without switching to ISIS.[^2]

### Failure scenario

If the SR metadata does not flood correctly, reachability may exist but the SID database is incomplete, so SR steering fails.[^2]

### Design insight

In OSPF SR, route reachability and SR metadata are separate operational checks. Validate both.[^2]

### Interview-ready answer

OSPF uses opaque LSAs for SR information. That lets the protocol keep normal SPF behavior while still distributing SID data.[^2]

## Adjacency SID

### Definition

An Adjacency SID is a local segment that identifies a specific adjacency, usually one exact link out of a router.[^5][^1]

### Why it exists

Prefix SIDs tell you how to reach a node, but not how to force a packet out a specific link. Adjacency SIDs solve that link-steering problem.[^5]

### How it works

The router allocates a local label for the adjacency and advertises it as a local SID. In SR, the ingress pushes that label to force the packet out the intended interface.[^5][^1]

### Real-world use case

Used for strict link steering, traffic engineering, and deterministic repair paths in the core.[^5]

### Failure scenario

If the wrong adjacency SID is selected, or the SID disappears with the adjacency, the path can fail immediately.[^5]

### Design insight

Use adjacency SIDs only when you truly need link-level control. They reduce abstraction and make the design more topology sensitive.[^5]

### Interview-ready answer

Adjacency SID is the SR tool for exact link steering. It is more precise than a Prefix SID, but also more sensitive to topology changes.[^5]

## LAN Adjacency SID

### Definition

LAN Adjacency SID is the adjacency SID model for broadcast segments, where the adjacency is to a pseudonode rather than a simple point-to-point neighbor.[^5]

### Why it exists

Broadcast links hide individual next hops behind a shared LAN. SR still needs a way to steer traffic to a specific node on that LAN.[^5]

### How it works

Each router on the LAN advertises its own LAN adjacency SID, tied to the shared segment model. The DIS/DR behavior matters because the pseudonode creates different advertisement semantics than a p2p link.[^5]

### Real-world use case

Used on shared Ethernet segments where point-to-point design is not possible but deterministic steering is still needed.[^5]

### Failure scenario

A wrong assumption about DIS/DR or pseudonode behavior can make the operator think a path is targetable when it is not.[^5]

### Design insight

LAN segments are inherently more complex than p2p links. Prefer p2p where possible; use LAN adjacency SIDs only when the design requires a multi-access segment.[^5]

### Interview-ready answer

LAN Adjacency SID restores link-level steering on broadcast networks. It exists because a LAN hides individual adjacencies behind a pseudonode.[^5]

## LFA

### Definition

Loop-Free Alternate is a local backup next-hop that can forward traffic around a failed link without looping the packet back to the PLR.[^6]

### Why it exists

It solves the need for immediate protection with almost no extra control-plane state.[^6]

### How it works

The PLR checks whether a neighbor can reach the destination without sending traffic back into the protected side of the topology. If yes, that neighbor becomes the backup next-hop.[^6]

### Real-world use case

Basic IGP fast reroute in SP cores when simple, low-state protection is acceptable.[^6]

### Failure scenario

Many destinations may not have a valid LFA, so coverage is incomplete.[^6]

### Design insight

LFA is a useful baseline, but it is topology dependent and not universally available.[^6]

### Interview-ready answer

LFA is the simplest fast-reroute method. It precomputes a loop-free alternate neighbor so packets can bypass a failure immediately.[^6]

## LFA tiebreakers

### Definition

These are the rules used to choose one valid LFA over another.[^6]

### Why it exists

Multiple neighbors may satisfy the loop-free rule, and the router needs a deterministic way to choose the better one.[^6]

### How it works

In the ISIS lab, the preference order is node-protecting LFA, then SRLG-disjoint LFA, then lowest-backup-metric LFA.[^6]

### Real-world use case

Useful when a topology has several backup choices and the operator wants the router to prefer stronger survivability.[^6]

### Design insight

Backup cost and backup survivability are not the same thing. The best repair path is the one most likely to survive the real outage.[^6]

### Interview-ready answer

LFA tiebreakers let you rank backup candidates. In practice, node protection and SRLG disjointness matter more than lowest metric.[^6]

## Remote LFA

### Definition

Remote LFA uses a remote PQ node and an MPLS tunnel when no local LFA exists.[^5]

### Why it exists

It expands repair coverage beyond what plain LFA can deliver.[^5]

### How it works

The PLR computes P-space and Q-space. A node in both is the PQ node, and traffic is tunneled to it so the packet can continue safely past the failed area.[^5]

### Real-world use case

Older LDP-based networks that need broader coverage than LFA alone can provide.[^5]

### Failure scenario

Remote LFA depends on LDP transport behavior, and the repair path can break service label transparency if the stack is not handled correctly.[^5]

### Design insight

Remote LFA is useful, but it is a transitional solution. In SR networks, TI-LFA is usually the cleaner long-term approach.[^5]

### Interview-ready answer

Remote LFA protects cases where no local LFA exists by tunneling to a PQ node. It improves coverage, but it still depends on LDP behavior.[^5]

## RLFA tiebreakers

### Definition

These are the selection rules used among remote LFA candidates.[^8]

### Why it exists

More than one PQ candidate can be valid, so the router needs a priority order.[^8]

### How it works

The lab emphasizes that RLFA still follows the remote-LFA model and does not provide the stronger node or SRLG protection that TI-LFA can.[^8]

### Failure scenario

Using RLFA for node protection goals is a design mistake because the backup may still share the same failure domain.[^8]

### Design insight

RLFA is a coverage tool, not a full survivability framework.[^8]

### Interview-ready answer

RLFA improves coverage but remains limited. It is not the right tool for true node or SRLG protection.[^8]

## TI-LFA

### Definition

Topology Independent LFA is an SR-based fast-reroute mechanism that computes the repair path after removing the failed resource from the topology.[^5]

### Why it exists

It solves LFA and RLFA weaknesses: incomplete coverage and imperfect repair path modeling.[^5]

### How it works

The router removes the failed link, node, or SRLG from the topology, recomputes the shortest path, and programs the SID stack required to follow that repair path.[^5]

### Real-world use case

Modern SR cores and underlays where scalable, deterministic fast reroute is required.[^5]

### Failure scenario

If the needed SID is missing or the label stack exceeds platform capability, the repair path may not be installable.[^5]

### Design insight

TI-LFA is generally the preferred protection strategy in SR designs because it is both scalable and topology-aware.[^5]

### Interview-ready answer

TI-LFA is SR fast reroute done properly. It computes the post-failure shortest path and builds the label stack to reach it immediately.[^5]

## TI-LFA node protection

### Definition

Node protection means the backup path avoids the failed next-hop router entirely.[^5]

### Why it exists

A node failure is a stronger failure case than a single link failure.[^5]

### How it works

The failed node is removed from the topology before computing the repair path. If a node-protecting path exists, it is preferred.[^5]

### Design insight

Node protection is usually more important than the lowest backup metric because it protects the actual failure domain.[^5]

### Interview-ready answer

TI-LFA node protection keeps traffic off the failed router, not just off the failed link. That is a stronger and more realistic protection goal.[^5]

## TI-LFA SRLG protection

### Definition

SRLG protection avoids paths that share the same shared-risk link group as the protected resource.[^5]

### Why it exists

Physical diversity matters; logical topology alone is not enough.[^5]

### How it works

Candidate paths sharing the same SRLG as the protected link are pruned before choosing the repair path.[^5]

### Failure scenario

If SRLG information is inaccurate or incomplete, the selected path may still fail with the same physical event.[^5]

### Design insight

SRLG protection is only as good as the physical risk model.[^5]

### Interview-ready answer

SRLG protection avoids shared physical risk, not just shared nodes or links. It is the right choice when survivability depends on infrastructure diversity.[^5]

## TI-LFA protection priorities, ISIS

### Definition

These priorities decide whether ISIS TI-LFA prefers node protection, SRLG protection, or fallback link protection.[^5]

### Why it exists

The router needs deterministic repair selection when several backup candidates exist.[^5]

### How it works

If both node and SRLG protection are configured, the router tries to satisfy both first. If not possible, it uses the strongest available option and then falls back to link protection.[^5]

### Design insight

Lowest-backup-metric is an LFA concept, not the main TI-LFA selector.[^5]

### Interview-ready answer

ISIS TI-LFA prefers the strongest survivability first, then degrades gracefully. The key is choosing the right failure-domain priority.[^5]

## TI-LFA protection priorities, OSPF

### Definition

OSPF TI-LFA uses a similar priority model to ISIS for choosing among repair candidates.[^7]

### Why it exists

OSPF also needs deterministic repair selection when multiple valid backup paths exist.[^7]

### How it works

The router tries to satisfy both node and SRLG protection if possible, then selects based on configured priority if one cannot be met.[^7]

### Design insight

The concept is the same as ISIS, but protocol-specific behavior still needs validation.[^7]

### Interview-ready answer

OSPF TI-LFA follows the same basic idea as ISIS: prefer the strongest repair path first. The precise priority behavior still needs to be checked in the protocol implementation.[^7]

## Repair path and packet flow

### Definition

This is the actual label-stack forwarding behavior during repair.[^5]

### Why it exists

A protection feature is only useful if the packet is forwarded correctly during failure.[^5]

### How it works

The PLR pushes the repair stack, the packet follows the backup path, and intermediate nodes pop or swap labels as needed. Adj-SIDs force exact link steering, while Prefix SIDs keep the traffic anchored to the destination node.[^5]

### Design insight

Always validate the real packet type, not just the route. A path that works for plain IP can still fail for VPN or service traffic if label context is lost too early.[^5]

### Interview-ready answer

The packet flow matters more than the feature name. Good design means the repair stack preserves forwarding correctness all the way to the destination.[^5]

## BGP EPE

### Definition

BGP Egress Peer Engineering lets the ingress influence which BGP peer the egress uses to forward traffic.[^9]

### Why it exists

Ordinary SR-TE can steer traffic to an egress PE, but not necessarily to the exact eBGP peer chosen by the egress bestpath process. EPE gives that extra hop of control.[^9]

### How it works

BGP EPE uses peering SIDs: peer node SID, peer adj SID, and peer set SID. The node SID means “forward to that peer,” the adj SID means “forward over that exact link,” and peer set SID means “forward to any peer in the set,” though peer set SID is not supported on IOS-XR in the lab.[^9]

### Real-world use case

Used in SR-TE and unified MPLS designs where the controller wants end-to-end path control including the egress peer choice.[^9]

### Failure scenario

If the egress BGP bestpath overrides the intended path, the route may not follow the desired peer without EPE. Multihop eBGP can also create additional peer-adj SID behavior.[^9]

### Design insight

EPE is about controlling the last routing decision at the egress, not just reaching the egress PE. That is why it is powerful for PCE-driven designs.[^9]

### Interview-ready answer

BGP EPE extends SR steering to the egress peer decision. It lets the controller choose not only the egress node, but also the exact BGP peer or link used there.[^9]

## SR-TE autoroute include

### Definition

Autoroute include makes an SR-TE policy appear as a usable forwarding path in the routing table.[^3]

### Why it exists

It gives a familiar RSVP-TE-like behavior where traffic can automatically recurse onto a TE policy.[^3]

### How it works

The router treats the SR-TE policy like a link in the graph and can install routes through it. The lab notes say this can be limited to specific prefixes and that it can also affect MPLS-to-MPLS forwarding entries.[^3]

### Real-world use case

Mostly migration, lab use, or special cases where policy-based recursion is intentionally desired.[^3]

### Failure scenario

The lab warns that this is discouraged because it can cause unlabeled or unintended recursion behavior, and BGP colored routes may override it if a matching SR policy exists.[^3]

### Design insight

Modern design should prefer color-based SR-TE / ODN style steering rather than autoroute include. Autoroute is useful, but it is not the cleanest operational model.[^3]

### Interview-ready answer

Autoroute include makes an SR-TE policy behave like an IGP path. It is convenient, but in modern designs it is usually discouraged in favor of color-driven policy selection.[^3]

## Inter-area, inter-IGP and inter-AS SR

### Definition

These are ways to extend SR beyond a single IGP domain.[^4][^1][^2]

### Why it exists

Real networks have areas, multiple IGP islands, and autonomous systems. SR has to work across all of them if it is going to be used end to end.[^4][^1][^2]

### How it works

In OSPF, prefix SID mappings are propagated inter-area. In an inter-IGP design, SR information can cross between different IGP domains with a border node. In inter-AS designs, BGP carries the inter-domain reachability and steering context.[^4][^1][^2]

### Real-world use case

Large SP backbones, unified MPLS, DC fabrics, and multi-domain service provider designs.[^4][^1][^2]

### Failure scenario

If SR metadata is not preserved across the boundary, plain reachability may exist but SR steering may not.[^4][^1][^2]

### Design insight

Boundary design matters more than the SR feature itself. Cross-domain SR needs clear ownership of labels, routing intent, and failure domains.[^4][^1][^2]

### Interview-ready answer

Inter-area, inter-IGP, and inter-AS SR extend segment routing across boundaries without collapsing the control-plane model. The challenge is preserving SR meaning while crossing those boundaries.[^4][^1][^2]

## SR BGP Data Center eBGP

### Definition

This is a fabric design where the underlay uses only eBGP and SR-MPLS for transport.[^1]

### Why it exists

It simplifies the DC control plane by removing IGP complexity and keeping policy explicit.[^1]

### How it works

Each router has its own ASN, eBGP exchanges reachability, and SR labels provide the transport behavior.[^1]

### Real-world use case

Routed leaf-spine fabrics where you want a clean eBGP underlay with SR transport.[^1]

### Failure scenario

ASN or next-hop policy mistakes can break the underlay graph, which means SR labels no longer have a valid forwarding substrate.[^1]

### Design insight

This model is clean if the entire fabric is designed intentionally. It is less forgiving if the BGP policy is inconsistent.[^1]

### Interview-ready answer

In an SR eBGP fabric, BGP builds reachability and SR builds transport. It is a good fit when you want explicit policy and low IGP complexity.[^1]

## SR BGP Data Center iBGP

### Definition

This is a fabric design where routers use iBGP inside one AS and SR-MPLS for transport.[^1]

### Why it exists

It keeps the fabric in a single administrative domain while still benefiting from SR transport.[^1]

### How it works

iBGP carries reachability, SR forwards the packets, and route reflection or full mesh becomes part of the scaling design.[^1]

### Real-world use case

Single-AS leaf-spine designs where operational simplicity is preferred over per-device AS separation.[^1]

### Failure scenario

Poor route-reflector design or next-hop handling can create suboptimal paths even though SR itself is working correctly.[^1]

### Design insight

SR does not remove the need for good BGP architecture. It only changes the transport substrate.[^1]

### Interview-ready answer

SR with iBGP keeps the fabric in one AS while using SR for transport. The design challenge is BGP scaling and policy consistency, not SR itself.[^1]

## Memory map

- Prefix SID → global destination identity.[^2][^1]
- SRGB → global label mapping base.[^2][^1]
- Router Capabilities / OSPF SR LSAs → how SR metadata is advertised.[^2][^1]
- Adjacency SID → exact link steering.[^1][^5]
- LAN Adjacency SID → broadcast-segment steering.[^5]
- LFA → first-line local repair.[^6]
- LFA tiebreakers → choose the best local alternate.[^6]
- Remote LFA → coverage extension using PQ node and LDP tunnel.[^5]
- RLFA tiebreakers → legacy remote selection logic.[^8]
- TI-LFA → SR-native repair.[^5]
- Node protection → avoid the failed router.[^5]
- SRLG protection → avoid shared physical risk.[^5]
- Protection priorities → strongest survivability first.[^7][^5]
- Packet flow → label stack correctness is the real test.[^5]
- BGP EPE → control egress peer choice.[^9]
- SR-TE autoroute include → convenient but usually discouraged.[^3]
- Inter-area / inter-IGP / inter-AS SR → extend SR across boundaries.[^4][^2][^1]

If you want, I can next convert this into a **GitBook-ready version** with cleaner heading hierarchy and shorter line lengths for direct copy-paste.
<span style="display:none">[^10][^11][^12][^13]</span>

<div align="center">⁂</div>


