!
**##Attribute##** 
- [ ] ==Well -known attributes==  (must be recognized by every BGP implementation)
	      1.  Mandatory   --> must be present in all update
						               ==Next hop , AS path , Origin==
	       2. Discretionary --> could be present in an update, but not required
	                                                  ==LP and atomic aggregator== 
- [ ] ==Optional attributes==-  (not expected to be recognized by all BGP implementations )
          - []  Transitive  --> Must sent in update , Even if not recognized Partials bit set and forward 
    -==Aggregator , Community , Ext Community==
				- [ ] Non Transitive -->Not sent in update 
							==Originator ID Cluster ID and MED== 

==Weight==
	It is locally significant within a router as it is never sent out in updates.
	specified per neighbor with the "neighbor weight" command or within a "route-map".
    
	Weight is applied to new incoming updates to affect OUTBOUND routing decisions.
	To enforce newly-set weight values, re-establish BGP sessions with the neighbors
	If ==no weight value is specified==, the default ==value of 0== is applied to received routes .
	Routes that the router ==originates locally== have a default value of ==32768==

==Local preference==

	Default value - 100
	Well Known discrenatory update ( Need not to be available )
	Applied on ebgp update in direction affect outbound traffic ( Service provider point of view )
	Applied on ebgp update on out direction does not works ( customer point of view or SP )
	Applied on IBGP updates in out direction ( SP applied for master backup for services access )

==AS Path== 
	!
	It is well known mandatory attribute available in all updates
	tell  that how many AS path has travel through
	When ==ordered set of AS listed then it call AS Sequence==  for ==unorder set of AS called AS Set==
	AS Set when route has been aggregated
	for Ebgp updates AS path value changes but not for iBGP updates
	Dual home customer can used it for select one path over other via As-prepend option
	!
==MED==
		1. med is metric default value 0
		2 . med is optional non transitive
		3. Applied outbound in eBGP update then it will not cross the receiving AS boundaries
		4.1 By default compare update from same neighboring AS
	    4.2 for Different peering AS  ==use : bgp always compare med==
	    4. Med derived from either network aggregate or redistribution from neighboring AS will not    cross the receiving AS Boundary
     5. 6. Med applied in IBGP updates will never cross Local AS boundaries. IT will works as
     6. 7. Med is not specify in redistribution then in Default Seed metric is taken
        7.1 for RIP OSPF and ISIS and EIGRP IGP metric is carry in BGP as MED
        7.2 for static and connected Default SEED metric in bgp is 0
- **ATOMIC_AGGREGATE**
    
    - Indicates that more specific routes were aggregated and some information may be hidden.
        
    - Ensures downstream routers know they should not deaggregate blindly.
        
- **AGGREGATOR**
    
    - Optional transitive attribute; identifies **which router and AS performed aggregation**.
        
- **NLRI**
    
    - The actual list of prefixes carried in an UPDATE