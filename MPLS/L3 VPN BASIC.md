!
what is VPN

VPN is a network that emulates private network over common infrastructure of SP

In an MPLS VPN network, ==packet forwarding takes place only if the router specified as the BGP next hop in the incoming BGP update is the same as the router that assigned the VPN labe==l in the MPLS VPN label stack.

 BGP next-hop IP must be the same router that assigned the VPN label.

**SO in MPLS vpn whenever next hop changes a new vpn lable generated for cust prefix** 

!
RD format < Type | Administrative value | Custom value >

RD is use to make overlapping prefixes between vrf unique


Type0 || 2byte ASN || 4byte value      
Type1 || 4byte IP || 2byte value
Type2 || 4byte ASN || 2byte value

RD should be unique for each vrf/VPN/customer

1 > Same RD used then
      in case of Multihome CE -- ( PE1--CE--PE2)  
      Both PE advertised same vpnv4 routes to RR
      RR Share only single prefix to Remote PE ( PE3 )
      in case Link failure from PE1
       Processing delay at PE1 / RR / and PE3 ( withdrawn and processing )
       PE3 --- dont have anyvpnv4/ipv4 pefix -- in transit stare and findaly have routes from PE2 
       
2 > Different RD used ACtive/ACTive
    in case of Multihome CE -- ( PE1--CE--PE2)  
    Both PE advertised unique  vpnv4 routes to RR .. RR have 2 Prefix : And PE3 have 2 different Prefix in vpnv4 table and 2 in ipv4 table
     in case failute from PE1 withdrawn routes from PE1 to RR 
     RR will share Withdrawn routers to PE3 and PE2 
     PE3 already have PE2 routes in vpnv4 table so only local processing after RR withdrawl msg
 Different RD used ACtive/backup 
    case of Multihome CE -- ( PE1--CE--PE2)  
    Both PE advertised unique  vpnv4 routes to RR .. RR have 2 Prefix ( transiant ): And PE3 have1 different Prefix in vpnv4 table and 1 in ipv4 table also PE2 also have PE1 prefere routes from PE1
     In case failute from PE1 , withdrawn routes from PE1 to RR  and RR  to PE3 and PE2 
     now RR send second best prefux (PE2) toward PE3 
[ ]  

Imp Notes :  RD same On All Nodes   wih Same cluster ID  or Diff Cluster ID on Dual RR 

 ==RD IS same on All PES  then no of vpnv4 routes received equal to no of RRs with lowest of Sending RID in case All Active and Active Backup case== 

>>> RD is Same Active active -- 2 Vpnv4 and 1 vrf table

>>> RD is Same Active Backup -- 2 Vpnv4 and 1 vrf table

==RD is Different then  No of vpnv4 routes received On Destination Node is 2   ( RR with lowest RID  is elected )
But if we made Active Backup then only one route of Lowest RID -PE send by Lowest RID RR

>>> RD is Different  Active active -- 2 Vpnv4 and 1 vrf table but

>>> RD is Different  Active Backup -- 1 Vpnv4 and 1 vrf table


       

> 
Globally significant
> if per/vrf RD load balancing is not possible with RR in picture
> if per/PE RD load balancing will be possible with RR in picture but additional memory will be used

Route Target
	Size is 64 bits / 8 bytes
	use to define membership of vrf
	it has same structure as RD

Basic CP  working
> CE advertised Customer local  routes to PE via (CE -- PE protocol)
> Localy Attached PE Store Customer routes in associated vrf 
> PE advertised Customer routes toward remote PE via MP-BGP (attached RD and make it vpnv4 unique routes)
> VPNV4 routes also has one or more RT and VPN lable 
> Remote PE  accept VPNV4 routes , RD is striped off and routes assigned to appropriate vrf based  on attached RT / and import RT membership
> Remote PE then advertised routes toward CE (through vrf  ) via any PE CE protocol 

DP working
	CE send the data to the PE device as unlabeled.
	PE router get this packet and add a pre-determined VPN (Inner) Label to each packet. This learn via MP-BGP
	PE router also add a Transport (Outer) Label to the packet. This is learned via LDP.
	P router will do either lable Swap /pop 
	egress router  other lable removed with VPN Label, the destination L3 VPN is determined

[[RD.xlsx]]



| Scenario                                            | Description                                                                                                                                           | Potential Loop Risk                                                                                                                   | Prevention Mechanism                                                                                                                                                                                                                               | Example Configuration Insight                                                                                                                                                                                                                                                         |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Same Customer AS Across Sites (BGP PE-CE)**       | Customer uses identical AS (e.g., AS 65001) on CEs at Site1 and Site2. Routes from Site1 reach Site2 via MPLS but get rejected by eBGP AS_PATH check. | Site2 discards its own prefixes advertised back via PE, causing reachability failure (not a forwarding loop, but suboptimal routing). | - **AllowAS-In**: On PE, permit 1-10 occurrences of own AS in path. - **AS-Override**: On PE, replace CE AS with provider AS in outgoing updates to Site2.                                                                                         | `router bgp 200` `neighbor 192.168.12.1 remote-as 65001` `address-family ipv4 vrf CUST` `neighbor 192.168.12.1 activate` `neighbor 192.168.12.1 as-override` Or: `neighbor 192.168.12.1 allowas-in 1`                                                                                 |
| **Redistribution Loops (OMP or IGP into BGP)**      | In SD-WAN hybrid (MPLS + Overlay), routes redistributed from OMP/BGP into local IGP; dual-homed site learns own route via higher-AD path.             | Route via low-AD local path (e.g., eBGP AD=20) preferred, but backup via OMP (AD=250) causes suboptimal or looped reconvergence.      | - **Administrative Distance Tweaks**: Set OMP higher than local protocols. - **Route Tags/Filters**: Tag redistributed routes; deny on import.                                                                                                     | `! In OMP config` `vpn 0` `redistribute connected` `! Route-map tag` `route-map TAG permit 10` `set tag 100` `! Deny import if tag matches`                                                                                                                                           |
| **Multi-Homed Site with Backdoor Link (BGP PE-CE)** | Site connected to two PEs + direct CE-CE link. Routes loop via backdoor if AS-override is used without checks.                                        | Traffic enters via PE1, exits via backdoor to CE2, re-enters MPLS via PE2 (data plane loop).                                          | - **SoO Community**: Attach unique site ID (e.g., 1:1 for Site1) on PE-CE updates; deny import if SoO matches.                                                                                                                                     | `ip vrf CUST` `rd 1:1` `route-target export 1:1` `route-target import 1:1` `!` `router bgp 200` `address-family ipv4 vrf CUST` `redistribute connected` `neighbor 192.168.12.1 send-community extended` `! Route-map to set SoO` `route-map SOO permit 10` `set extcommunity soo 1:1` |
| **OSPF PE-CE with Backdoor Link**                   | OSPF between PE-CE; backdoor OSPF link between CEs. Backbone routes advertised as external LSAs.                                                      | PE2 learns route from CE2 (via backdoor), re-advertises to core, loops back to PE1.                                                   | - **Down Bit (DN)**: Set in Type 3/5 LSAs from PE to CE; CE ignores DN-marked LSAs. - **Domain Tag**: Tags external routes; CE skips if tag matches domain ID. - **Sham Link**: Creates intra-area path via MPLS to prefer backbone over backdoor. | `router ospf 2 vrf CUST` `domain-id 0.0.0.1 type 0` `! DN bit auto-set for external LSAs` `! Sham link between PE loopbacks` `interface Loopback0` `ip ospf 2 area 0` `ip ospf cost 1` (low cost to prefer MPLS)                                                                      |
| **LDP LSP Loops in Core**                           | Temporary IGP convergence delay causes LDP to bind labels forming a loop.                                                                             | Packets cycle indefinitely until TTL expires (slow detection).                                                                        | - **Path Vector TLV**: In LDP messages, track LSR path; reject if loop detected. - **TTL Propagation**: Label TTL mirrors IP TTL; expires at 0.                                                                                                    | Global LDP config: `mpls ldp` `loop detection` (enables path vector). Verify: `show mpls ldp discovery` (check for stalled bindings).                                                                                                                                                 |