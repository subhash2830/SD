#R4
router bgp 65000
 neighbor IBGP peer-group
 neighbor IBGP remote-as 65000
 neighbor IBGP update-so lo0
 neighbor 1.1.1.1 peer-group IBGP
 neighbor 3.3.3.3 peer-group IBGP
 neighbor 5.5.5.5 peer-group IBGP
 !
 add l2vpn vpls
  neighbor 1.1.1.1 activate
  neighbor 3.3.3.3 activate
  neighbor 5.5.5.5 activate
  neighbor IBGP route-reflector-client
  neighbor IBGP suppress-signaling-protocol ldp

#R1
router bgp 65000
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-so lo0
 !
 add l2vpn vpls
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 suppress-signaling-protocol ldp
!
l2vpn vfi context VPLS1
 vpn id 10
 autodiscovery bgp signaling bgp
  ve id 1
!
int Gi4
 service instance 10 eth
  encapsulation default
 exit
!
bridge-domain 10
 member Gi4 service-instance 10
 member vfi VPLS1

#R3
router bgp 65000
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-so lo0
 !
 add l2vpn vpls
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 suppress-signaling-protocol ldp
!
l2vpn vfi context VPLS1
 vpn id 10
 autodiscovery bgp signaling bgp
  ve id 3
!
int Gi6
 service instance 10 eth
  encapsulation default
 exit
!
bridge-domain 10
 member Gi6 service-instance 10
 member vfi VPLS1

#R5
router bgp 65000
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-so lo0
 !
 add l2vpn vpls
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 suppress-signaling-protocol ldp
!
l2vpn vfi context VPLS1
 vpn id 10
 autodiscovery bgp signaling bgp
  ve id 5
!
int Gi6
 service instance 10 eth
  encapsulation default
 exit
!
bridge-domain 10
 member Gi6 service-instance 10
 member vfi VPLS1