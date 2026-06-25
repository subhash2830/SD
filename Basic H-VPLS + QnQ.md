#R1
int gi4
 service instance 1 ethernet
  encapsulation default
  rewrite ingress tag push dot1q 10 symmetric
 exit
int gi5
 service instance 1 ethernet
  encapsulation default
  rewrite ingress tag push dot1q 20 symmetric
 exit
int gi6
 service instance 10 ethernet
  encapsulation dot1q 10
 service instance 20 ethernet
  encapsulation dot1q 20
 exit
!
bridge-domain 10
 member gi4 service-instance 1
 member gi6 service-instance 10
!
bridge-domain 20
 member gi5 service-instance 1
 member gi6 service-instance 20

#R3
int gi6
 service instance 1 ethernet
  encapsulation default
  rewrite ingress tag push dot1q 10 symmetric
 exit
int gi7
 service instance 10 ethernet
  encapsulation dot1q 10
exit
!
bridge-domain 10
 member gi6 service-instance 1
 member gi7 service-instance 10

#R5
int gi6
 service instance 1 ethernet
  encapsulation default
  rewrite ingress tag push dot1q 20 symmetric
 exit
int gi7
 service instance 20 ethernet
  encapsulation dot1q 20
exit
!
bridge-domain 20
 member gi6 service-instance 1
 member gi7 service-instance 20

## 
## N-PEs
##
#R2
l2vpn vfi context VPLS10
 vpn id 10
 autodiscovery bgp signaling bgp
  ve id 2
!
l2vpn vfi context VPLS20
 vpn id 20
 autodiscovery bgp signaling bgp
  ve id 2
!
int gi6
 service instance 10 eth
  encapsulation dot1q 10
 service instance 20 eth
  encapsulation dot1q 20
 exit
!
bridge-domain 10
 member vfi VPLS10
 member gi6 service-instance 10
!
bridge-domain 20
 member vfi VPLS20
 member gi6 service-instance 20

#R4
l2vpn vfi context VPLS10
 vpn id 10
 autodiscovery bgp signaling bgp
  ve id 4
!
int gi7
 service instance 10 eth
  encapsulation dot1q 10
 exit
!
bridge-domain 10
 member vfi VPLS10
 member gi7 service-instance 10

#R6
l2vpn vfi context VPLS20
 vpn id 20
 autodiscovery bgp signaling bgp
  ve id 6
!
int gi7
 service instance 20 eth
  encapsulation dot1q 20
 exit
!
bridge-domain 20
 member vfi VPLS20
 member gi7 service-instance 20