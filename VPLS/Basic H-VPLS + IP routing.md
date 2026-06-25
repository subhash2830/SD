#R1
int Gi4
 service instance 1 eth
  encapsulation default
!
l2vpn vfi context VPLS1
 vpn id 1
 autodiscovery bgp signaling bgp
  ve id 1
!
bridge-domain 1
 member gi4 service-instance 1
 member vfi VPLS1
!
ip dhcp pool CLIENTS
 vrf INET
 network 100.1.1.0 255.255.255.0
 default-router 100.1.1.1
!
int bdi1
 vrf forwarding INET
 ip add 100.1.1.1 255.255.255.0
 no shut

#R3
int Gi6
 service instance 1 eth
  encapsulation default
!
l2vpn vfi context VPLS1
 vpn id 1
 autodiscovery bgp signaling bgp
  ve id 3
!
bridge-domain 1
 member gi6 service-instance 1
 member vfi VPLS1

#R5
int Gi6
 service instance 1 eth
  encapsulation default
!
l2vpn vfi context VPLS1
 vpn id 1
 autodiscovery bgp signaling bgp
  ve id 5
!
bridge-domain 1
 member gi6 service-instance 1
 member vfi VPLS1