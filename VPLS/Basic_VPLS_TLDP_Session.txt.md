#R1
int gi4
 service instance 100 ethernet
  encapsulation default
 exit
!
l2vpn vfi context VPLS1
 vpn id 100
 member 3.3.3.3 encapsulation mpls
 member 5.5.5.5 encapsulation mpls
!
bridge-domain 100
 member Gi4 service-instance 100
 member vfi VPLS1

#R3
int gi6
 service instance 100 ethernet
  encapsulation default
 exit
!
l2vpn vfi context VPLS1
 vpn id 100
 member 1.1.1.1 encapsulation mpls
 member 5.5.5.5 encapsulation mpls
!
bridge-domain 100
 member Gi6 service-instance 100
 member vfi VPLS1

#R5
int gi6
 service instance 100 ethernet
  encapsulation default
 exit
!
l2vpn vfi context VPLS1
 vpn id 100
 member 1.1.1.1 encapsulation mpls
 member 3.3.3.3 encapsulation mpls
!
bridge-domain 100
 member Gi6 service-instance 100
 member vfi VPLS1