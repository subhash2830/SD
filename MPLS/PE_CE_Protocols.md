!
1 > PE CE protocol BGP 
			no mutual redistribution is requried as BGP is multiprotocol
			Use different AS on each site
			no special trick is required
			Use same AS on each site
			we need to use as-override on PE end(works in outbound direction towards CE)
			if site is multihomed we need to use SOO along with as-override to avoid loop
			or we need to use allowas-in on CE (works in inbound direction)

2  > PE CE protocol EIGRP
			Cost-community - carries EIGRP route metric value
			rule for SOO
			if the route is sent or received on the interface has the same SOO value as configured on the interface the route is discarded
			if route is sent or received on the interface has different SOO value as configured on the interface value is preserved
			soo value is carried in EIGRP as well
			Same soo on PE interface facing CE
			Different soo on PE interface facing CE
			Soo configured on CE interface as well
			Same AS number    > routes consider internal  metric is recovered from MED value 
			Different AS number > routes consider external  med value is not copied as metric but seed metric is used
			in both cases we need to mention seed metric

3 > PE CE protocol OSPF
		mutual redistribution is required between OSPF and BGP
		OSPF route cost is carried in BGP MED
		Extended community used are  < ==Route-type ; router-id ; domain-id== >
		by default domain-id is same ospf process id if not set explicitly
	    Same domain-id inter-area routes
		==LSA 1 2 3 converts to LSA 3==
		Different domain-id external routes
		==LSA 1 2 3 converts to LSA 5==
		Down bit is used for loop prevention for summary route LS3
		Domain tag is used for loop prevention for external routes LSA5
		If we have backdoor link still inter-area routes will not be preferred so we need sham-link

4 > PE CE protocol RIP
		mutual redistribution is requried
		RIP metric is carried in BGP MED
		use metric transparent to recover metric from MED
		we always need to mention metric while doing redistribution else route will not forward