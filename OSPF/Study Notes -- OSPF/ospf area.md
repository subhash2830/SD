[[ospf_Area.png]]

| Area Type         | LSA Allowed        | 0/0 Allowed in Area | Injected in Area By default | By defult originate | LSA type               | Remarks                                               |
| ----------------- | ------------------ | ------------------- | --------------------------- | ------------------- | ---------------------- | ----------------------------------------------------- |
| Backbone          | 1,2,3,4,5          | Yes                 | no                          | no                  | LSA 5                  | Originate Alwayes                                     |
| Stub              | 1,2,3              | Yes                 | Yes                         | Yes                 | LSA 3                  | area 1 stub                                           |
| Totally Stub      | 1 ,2  0/0 as LSA 3 | Yes                 | ABR injects                 | Yes                 | LSA 3                  | area 1 stub no summary                                |
| NSSA              | 1,2, 3, 7          | Yes                 | Yes                         | No                  | LSA 7                  | area 10 nssa deafult info originated                  |
| NSSA Totally stub | 1,2 , 7            | Yes                 | Yes                         | yes                 | LSA 3                  | area 1 nssa no summary  ( t 7 )                       |
|                   |                    |                     |                             |                     | T3 and  T7 But T3 Wins | area 1 nssa default informmation originate no summary |
|                   |                    |                     |                             |                     |                        |                                                       |

- [ ] why area
  To scale large N\w  area s created 
      if any changes in n\w cause all router to process the updates .. Cause unnecessary CPU and     memory utilisation  (i.e >  link instabilities in one area never pass beyond its boundary)

- [ ] SPF find loop free path to destination  i.e find star topology 
      when we create area , we actualy break the links state property (i.e find loop free path in using SPF )  also between area only prefix information travel  means routers in one area totally depend on ABR for routes in other area 
	  
	 Also routing update between area may cause loop (if area connected in ring formate ) thease hypthetically create loops 

      To break this we need star topo 

     where Area 0 at centre and all information paas through area 0
 ```
 Analogy

flat network scalability issue 

> area created 

> routing update bw area create loops 
  i.e distance vector behaviour this may cause loops 

> avoid these info pass thr only area 0 
  i.e creating star topology
> 
> - Flat network → ❌ Not scalable
- Create areas → ✅ Divide network
- Inter-area routing → behaves like DV
- DV behavior → ⚠️ Loop risk
- Solution → enforce **Area 0 backbone**
- Final design → ✅ **Star topology**
```
 



==Stub Area==
	    To control external routes advertisement in ospf Area stub is created
	    Once define stub in ospf ABR send a 0.0.0.0 routes for external prefix reachibility
	    filter out T4 and T5 lsa ( t3 0.0.0.0/0 )
	    

==NSSA Area==
		To have redistibution of external domain routes to only perticular area 
		External routes accepted as T7
		But Does not accept external routes come from other area

==Totally Stuby and NSSA==
		>> filter out inter domain routes 







