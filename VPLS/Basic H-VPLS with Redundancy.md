!
! N-PEs
!
#R2
l2vpn vfi context VPLS1
 vpn id 100
 autodiscovery bgp signaling ldp
!
bridge-domain 100
 member vfi VPLS1
 member 1.1.1.1 100 encap mpls

#R4
l2vpn vfi context VPLS1
 vpn id 100
 autodiscovery bgp signaling ldp
!
bridge-domain 100
 member vfi VPLS1
 member 3.3.3.3 100 encap mpls
 member 1.1.1.1 100 encap mpls

#R6
l2vpn vfi context VPLS1
 vpn id 100
 autodiscovery bgp signaling ldp
!
bridge-domain 100
 member vfi VPLS1
 member 5.5.5.5 100 encap mpls

!
! U-PEs
!
#R1
int Gi4
 service instance 1 eth
  encap default
 exit
!
l2vpn xconnect context N_PE
 member gi4 service-instance 1
 member 2.2.2.2 100 encap mpls group N_PE priority 1
 member 4.4.4.4 100 encap mpls group N_PE priority 2

#R3
int Gi6
 service instance 1 eth
  encap default
 exit
!
l2vpn vfi context VPLS1
 vpn id 100
 member 4.4.4.4 encap mpls
bridge-domain 100
 member gi6 service-instance 1
 member vfi VPLS1

#R5
int Gi6
 service instance 1 eth
  encap default
 exit
!
l2vpn vfi context VPLS1
 vpn id 100
 member 6.6.6.6 encap mpls
bridge-domain 100
 member gi6 service-instance 1
 member vfi VPLS1