1. 224.0.0.5 is the OSPF All-routers multicast and 
2. 224.0.0.6 is the OSPF All-Designated routers multicast
3. IP Protocol 89
4. OSPF uses a default metric of 20 when redistributing from any other   IGP to OSPF 
5. Interface MTU is not checked during formation of neighbor adjacencies. However, mismatched interface MTU’s will prevent the successful exchange of DD packets and prevent the neighbors from reaching the FULL state
6. OSPF uses a default metric of 1 when redistributing from BGP
7. ospf is link state but act as distance vector for inter area communication and link state as intra are communication
  8 . [[ospf Data structure ]]

==Neighbor states==
 Down
 Init       :  Hello is received from other router but my Router-ID is not included in it
 2-way  : OSPF elects DR and DBR in this state and forms adjacency with other router
 Exstart :  Initial DBD packet is exchanged and Master/Slave election takes place in this state.
 Exchange :  Actual DBD packets are exchanged
Loading :  LSR and LSU are sent
Full :  Router formed full adjacency and all link state information is exchanged

==OSPF neighborship formation rule==   ==AACMSTR==
 Area
 Authentication
 Compatible n/w types
 MTU
 Stub flag
 Timers
 Router-ID should be unique

 ==ospf cost ==
Interface cost = Reference BW / Interface BW

- Reference BW = **100 Mbps**
- So:
    - ≥100 Mbps links → cost = **1** (no differentiation)

Different ways to enable OSPF cost
		  auto-cost reference-bandwidth 1000
		  ip ospf cost 31
		  bandwidth 10000 
		  nei 10.0.0.1 cost 2000

 



