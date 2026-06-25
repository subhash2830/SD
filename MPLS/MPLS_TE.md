!
What is traditionally approach of link provisioning 
		A conservative capacity-planning rule that upgrade links when utilization reaches to 50%, ensuring at least twice as much capacity. Following this simple rule has helped them in achieving tight SLAs for delay, jitter, and loss. .

WHY MPLS TE Needed
		Minimize maximum utilization in normal working case.
		Minimize propagation delay for delay sensitive traffic.
		Avoid situations where certain parts of network are congested and other parts are underutilized. Improving the utilization of existing resources lowers the investment.
		Certain traffic gets priority in the event of a resource crunch like a link or node failures and Fast Re-Route.

When Congestion normally occurs normally
	1. When network resources do not have enough capacity to contain the offered load. Increse rsource capacity as only option
	2. When traffic streams are inefficiently allocated to available resources, resulting in subsets of network resources becoming over-utilized while other subsets remain underutilized  MPLS TE as solution

TE deployment model
		Tactical TE: to address specific performance problems (such as hot spots) that occur in the network in an improvised and reactive manner. They are selectively deployed in the network areas.
		Strategic TE: Strategic TE tackles the congestion problem from a more systematic , taking into consideration the immediate and longer-term outcomes of specific policies and actions. Strategic TEs are deployed throughout out the network.

MPLS TE Terminology
	Head end router 
	Tail end router
	TE tunnels

Building blocks of MPLS TE
		Link constraints
		OSPF / ISIS protocol with TE extension
		Constrained SPF-CSPF)
		Resource reservation protocol (RSVP) to signal a tunnel.
		Forward traffic into tunnel.

IGP with TE extension
 TE metric; Maximum BW; Maximum reservable   BW; Unreserved BW; and Administrative group

Link attributes :
  > Maximum reservable BW
  > Attribute flags
  > TE metric
  > Shared risk link groups (SLRG)
  > Maximum reservable sub-pool bandwidth


MPLE TE Tunnel attribute
>	Tunnel destination
>	Desired BW
>	Path setup options  explicit or dynamic
>	Set-up and holding priority 
>				Setup priority
					High-important tunnel has to be configured with less priority.
					Setup priority of a tunnel cannot be lower than the holding priority.
					Holding priority
					Indicates whether the tunnel can be dropped when high importance tunnel has to be establish.


Re-optimization in TE tunnel 
> Periodic re-optimization   > By default every one hou
> Event reoptimization;          >
> 				When a link comes up or more BW is configured for TE, it will not trigger any reoptimization of tunnel, by default.
   Manual reoptimization    > 
   Using “mpls traffic-eng reoptimize” on router prompt on head end LSR. 
   “mpls traffic-eng reoptimize tunnel <number>” for particular tunnel.


CSPF 
		SPF algorithm which gives the path to be taken for a TE tunnel.
		Calculation based on TE-IGP database and takes into account all link attributes.
		If there are multiple paths with same cost and same constraints;
		Path with largest minimum BW is chosen
		If tie, the path with few hops
		Still tie, IOS picks one.

Basic operation of signaling
		RSVP PATH message is sent from head end to tail end LSR and carries a request for Label. It has Explicit Route Object (ERO) which has the information(IP address) about how the PATH message to travel.
		
		Each router on the path when receives the PATH message, it extracts it IP address from the ERO and send it to next-hop specified in ERO.
		
		On reaching Tail end router, it sends RSVP RESV message which carries a label. Each LSR on the path reserves the bandwidth specified and returns a label with RESV message, which finally reaches head-end LSR.