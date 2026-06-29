1    On Broadcast segment , ospf uses DR/BDR to advertised ospf updates  to attached routers  on  Broadcast segment 

2    This will reduce same control plane updates over shared segment thus reduces CPU /Memory   

3      less number of adjancy with compare full mesh adjancy ( n(n-1)/2 )   to (n-1) /2 only

4      DR and BDR works only in Control plane and not in Data plane forwarding 

5      DR is property of interface and not the router 

6       DR also  advertised its connected routers information to rest of ospf networks

- [ ] Election of DR []

1.> Check high ospf priority  ( default = 1 )  Max = 255 ( always DR ) and Min = 0 ( Always  
					DRothers 
2.> Hightes  Router _ID [[ospf RID ]]

3.> Loopback not available Then Available highest int IP 

Basically DR election done by hello protocol , Hello packet contain DR / BDR field alonge with int proiority  field

 - [ ] DR Function : 
                             Reduce CPU and B|w on shared segments 
				 Advertised Connected routers info to rest of ospf domain ( within area )

