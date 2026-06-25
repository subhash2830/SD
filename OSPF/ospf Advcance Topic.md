[[ospf Forwarding address]]


[[ospf Forwarding metric]]

E1 vs E2  and OSPF route preference 
[[ospf E1 verser E2]]

Type 4 need 
[[Why LSA 4 needed ]]


==Interview topic==

OSPF rule for inter-are loop prevention

  Inter-area routes are announced to other areas only if they are associated with the backbone area

OSPF link-local signalling

 its a bit set in option field

 used only in hello or DBD packet

 used to inform additional capability such as NSF ,graceful restart

==OSPF LSA pacing/grouping

 By default all LSA are flooded every 30 sec(LSA Flooding)

 this will cause lot off traffic and calculation every 30 mins on each router

 to avoid this if we track individual LSA age and flood as soon as half is over

 this also cause fragmentation problem

 so we group LSA which are near to half life and flood them at same time (240 sec default)

 command is "timers pacing [flood/lsa-group/retransmission]"

==OSPF LSA throttling       5sec  10sec  60sec

 command is "timers throttle lsa [start] [hold] [max]"

 LSA will be generated after start time and hold time kicks in

 if same LSA need to be regenerated time in that HOLD time period

 LSA will be regenerated after intial hold time and now new hold time is doubled

 this process is carried till we reaches max time

==OSPF SPF throttling

 same as LSA throttling but this is related to SPF calculation

==When is forward address set in external LSA

 An ASBR in the NSSA area, by default, will always set the FA to be non-zero

 and also set the P-bit; by design/RFC, the NSSA ASBR has to set the FA to

 be non-zero, because when the ABR makes the 7/5 translation it becomes an ASBR as well,

 and we loose track of who the real ASBR

 for LSA 5 forward address is set when OSPF is enabled on the non-passive broadcast interface with

 network command

Demand circuit

 Suppresses both Hello as well as Periodic LSAs.

flood reduction

 Suppresses Periodic LSAs(re flooding every 30mis).

what is LSA7 translation election

 if we have multiple ABR then LSA7 to LSA5 conversion is only done by ABE with highest RID

When P-bit is set

 NSSA ASBR sends LSA7 P bit is set

 when NSSA ASBR and ABR are same P bit is not set

 when ASBR router is attached to another area as well as to NSSA area



