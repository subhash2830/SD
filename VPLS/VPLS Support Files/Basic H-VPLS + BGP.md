#
# U-PEs
#
#R1
l2vpn vfi context VPLS1
 vpn id 10
 member 2.2.2.2 encapsulation mpls
!
int gi4
 service instance 10 ethernet
  encapsulation default
 exit
!
bridge-domain 10
 member gi4 service-instance 10
 member vfi VPLS1

#R3
l2vpn vfi context VPLS1
 vpn id 10
 member 4.4.4.4 encapsulation mpls
!
int gi6
 service instance 10 ethernet
  encapsulation default
 exit
!
bridge-domain 10
 member gi6 service-instance 10
 member vfi VPLS1

#R5
l2vpn vfi context VPLS1
 vpn id 10
 member 6.6.6.6 encapsulation mpls
!
int gi6
 service instance 10 ethernet
  encapsulation default
 exit
!
bridge-domain 10
 member gi6 service-instance 10
 member vfi VPLS1

#
# N-PEs
#
#R2
l2vpn vfi context VPLS1
 vpn id 10
 autodiscovery bgp signaling ldp
 !
bridge-domain 10
 member vfi VPLS1
 member 1.1.1.1 10 encapsulation mpls

#R4
l2vpn vfi context VPLS1
 vpn id 10
 autodiscovery bgp signaling ldp
 !
bridge-domain 10
 member vfi VPLS1
 member 3.3.3.3 10 encapsulation mpls

#R6
l2vpn vfi context VPLS1
 vpn id 10
 autodiscovery bgp signaling ldp
 !
bridge-domain 10
 member vfi VPLS1
 member 5.5.5.5 10 encapsulation mpls