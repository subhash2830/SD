E1 routes use External metric + Metric to reach ASBR does uses hybrid approach of Cold and Hot potato routing 

E2 uses external metric () Hot potato --- Get rid FROM  nearest exit point

Reason to prefer OSPF E1 route over E2 route is that OSPF E1 route uses lowest redistributed cost + lowest cost to reach ASBR this behaviour is hot potato + cold potato routing so packet will reach to destination as quickly as possible

==ospf Route preference== 
If there is more than one route to the same destination within an OSPF domain, the route preference is defined as follows, regardless of the value of the route metric.

1.Intra-area routes are preferred over inter-area and external routes.

2.Inter-area routes are preferred over external routes.

3.External type 1 routes are preferred over external type 2 routes

==Hot potato routing==  – sent packet out of autonomous system as quickly as possible (consider internal AS cost to reach AS exit point )

==Cold potato routing== – hold on the packet in originating autonomous system until it reaches as near to destination as possible (consider external cost to reach destination from As exit point  and ignore the cost to reach AS exit point )

Consider we have 2 E2 routes for same destination with different redistributed cost on the ASBR then OSPF will only consider external cost (redistributed cost) and ignores the internal cost to reach ASBR this behaviour is same as cold potato routing.

Now consider we have 2 E2 routes for same destination with same redistributed cost on the ASBR then OSPF  will compare internal cost to reach ASBR for both routes and select lowest cost path to reach ASBR this behaviour is same as hot potato routing

