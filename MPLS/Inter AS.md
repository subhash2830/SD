   !
==Inter-AS MPLS VPN option A==
!
Simple method
VRF are configured on ASBR as well Interface or sub interface connected between ASBR belongs to customer vrf configured

Protocol running between ASBR per vrf can be any IGP or BGP(BGP is prefered over IGP)

Routing information exchanged between ASBR is IPV4(ASBR treats other ASBR as CE)

There will be 2 LSP between PEAS1 to ASBR1 and another between PEAS2 to ASBR2

Traffic on the link between ASBR is IPV4

There would be N interface if N vrf are there and per vrf EBGP peering between ASBR

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
==Inter-AS MPLS VPN option C==

> MP-EBGP is configured between RR of 2 AS  and Need to use next-hop-unchanged command on MP-EBGP peer
> Normal IPV4 EBGP is configured between ASBR of 2 AS
> Need to use send-label command on IPv4 EBGP
> Need to announce all PE loopback IP over ASBR EBGP and need to redistribute that in IGP of other AS
   Scalable but complex
      per vpn qos not possible secure info leak from one AS to another
resource are saved by not duplicating vpnv4 info on ASBR

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

==Inter-AS MPLS VPN option B==
ASBR-to-ASBR 
use of MP-eBGP between ASBRs to transport VPNv4 prefixes.

There is no requirement of TDP/LDP or any IGP to be enabled on the link connecting the two ASBRs. The MP-eBGP session between directly connected interfaces on the ASBRs enables the interfaces to forward labeled packets.

no bgp default route-target filter needs to be configured on an ASBR that does not have any VRFs configured or is functioning as a RR
		-   The command ensures that the ASBR accepts the BGP VPNv4 prefixes from other PE routers inside the AS. 
		- The default behavior is to deny incoming VPNv4 prefixes that are not otherwise imported into any local VRF.

Two method :
	-   Option 2a— Next-hop-self method 
	-                       3 Vpn lable  generated so 3 LSP 
							 V1 ----  IN AS 2   Originator PE 
							 V2  ---  ASBR2 in AS2 (Advertised to ASBR1)
							 V3  --- ASBR1 generated in AS1 
							- 
         Option 2b— Redistribute connected method
				          2 Lable created i.e 2 LSP 
				          V1 -- IN AS 2   Originator PE  
				          V2 --- Either ASBR1 / 2 created While doing redistribute connected 



