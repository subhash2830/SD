 LSA type 3 is describing how to reach ASBR as a subnet. But the router ID is a label in an IP address format. You can even set the RID as an IP address which is not configured on any interface and OSPF still works and even LSA type 5 will still be reachable, this is because of LSA type 4.

LSA type 4 kind of  name resolver. Because the ==router ID can be any label or name== (in IP format) which doesn’t exist on the router, we do need the LSA type 4 to tell routers in area 0 how to reach ASBRs in area 1.

> in ospf node is identify by its router-id 
> More-ever it is not necessary that router-id should be reachable in ospf ( router-id is 32 bit similar to IP addr but in reality's not a IP add )
> Within Area all nodes in area knowns each others i.e know router-id or loopback ( having same topological info) but other area dont have topological view other area nodes they have only routing view ( type -3 lLA)

to figure-out ASBR by its router-id in other Area we need ABR to advertised type-4 lsa
this external mechnium allow ABR to advertised ASBR's router-id in other area setting self as next-hop for reachability .


> 