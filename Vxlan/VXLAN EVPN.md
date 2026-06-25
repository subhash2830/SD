 

Type 2 
> Only MAC
> MAC + IP

Type 3
To identify vtep switch per VNI  in VXLAN fabric ( Overlay nw advertised )
Announce property as ingress replication 
IR / Multicast 
VTEP IP information ( used as next Hop ) {{ NH reachability via underlay}}

EVPN unable for overlay CP 
Type 1 -- Auto Discovery ( Remote end forwarding LB and alliasing) 

> Server1 is multihomed to vTEP 1 and vTEP 2
> Same ESI value is assigned on both VTEP
> Bothe VTEP advertised AD routes with ESI value in VXLAN fabric 
> All VTep come to know Server1 is multihome to VTEP1 and VTEP2
> This is useful for Remote forwarding ( from Other server to Server1)

use case 
Alliasing : Here MAC route (Server1) are bind with ESI and ESI reachable via 2 Vtep ( 1 and 2 )
Similar Hirarical FIB -- EVPN control Plane help to achive back up path in case of failure case 
No DP forward + NO Withdrwan message == this will reduce downtime
!
!
Type 4 
VTEP with same ESI value need to discover each other ( they are multihome )
DF -- Who will forward traffic in vXlan 

FroM DF to Non DF ( looping possible )
Traffic not forwarded on same ESI value link 

Broadcast From Non DF to DF ( Multiple copies of packet )
> Traffic not forwarded on same ESI value link 
> DF only forward traffic on LAN link 





[[vxlan.png]]