#R4
router bgp 65000
 neighbor IBGP peer-group
 neighbor IBGP remote-as 65000
 neighbor IBGP update-so lo0
 neighbor 1.1.1.1 peer-group IBGP
 neighbor 3.3.3.3 peer-group IBGP
 neighbor 5.5.5.5 peer-group IBGP
 add l2vpn vpls
  neighbor 1.1.1.1 activate
  neighbor 3.3.3.3 activate
  neighbor 5.5.5.5 activate
  neighbor IBGP route-reflector-client

#R1
router bgp 65000
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-so lo0
 add l2vpn vpls
  neighbor 4.4.4.4 act
!
l2vpn vfi context VPLS1
 vpn id 100
 autodiscovery bgp signaling ldp
!
int gi4
 service instance 100 eth
  encapsulation default
 exit
!
bridge-domain 100
 member gi4 service-instance 100
 member vfi VPLS1

#R3, R5
router bgp 65000
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-so lo0
 add l2vpn vpls
  neighbor 4.4.4.4 act
!
l2vpn vfi context VPLS1
 vpn id 100
 autodiscovery bgp signaling ldp
!
int gi6
 service instance 100 eth
  encapsulation default
 exit
!
bridge-domain 100
 member gi6 service-instance 100
 member vfi VPLS1