!

Hello Timer  : 60 Sec
Hold Timer  : 180 Sec 

Periodic keepalives are used to verify TCP connectivity .IF miss 3 consecutive Hello session is torn down BGP session.

KA Selection 
					hold time is sent in updates . Two peers agree on the lowest hold time value between them and then calculate the keepalive value from the hold time value
BGP update interval 
		iBGP -  5 
		eBGP    30  (in sec )
		
   
bgp update-delay (default 120 when neighbor first time comes up; update is sent after this timeout)

 bgp scan-time <seconds> (60 sec default)
                     bgp scanner process track BGP next hop values but this process is periodic


Event driven is next hop trigger

			bgp nexthop trigger enable
			bgp nexthop trigger delay <seconds> <5 sec default>
			
bgp fast-external-fallover
          neighbor <IP> fall-over

timers bgp <keepalive> <holdtime>
neighbor <IP> advertisement-interval <seconds>

!
!
BGP state maching
BGP MTU path discovery
BGP  message formate 
BGP update -- std and exte
timers 
		

  
  
 