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
- **Cost Community**:
    - EIGRP metric is not native to BGP → SP encodes it in **BGP cost community** and carries across core.
    - On the remote PE, it is converted back to EIGRP metric.
- **Same AS (CE–CE appears internal)**:
    - Metric is reconstructed using MED (via cost community).
    - Routes treated as **internal EIGRP** → better preference.
- **Different AS**:
    - Treated as **external EIGRP**.
    - Original metric is NOT directly trusted → **seed metric must be configured**.
- **Seed Metric (mandatory)**:
    - Without seed metric → redistributed routes may get metric 0 or invalid → no proper path selection.
- **SoO (Site of Origin)**:
    - Tag applied at PE-CE edge.
    - **Rule**:
        - If incoming route has SAME SoO as interface → DROP (loop prevention).
        - If DIFFERENT SoO → accept and preserve.
    - Carried inside MPLS VPN (via BGP extended community).

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