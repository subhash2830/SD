!
 - [ ] BGP uses tcp port 179 for reliable transport.
         
| **Parameter**                                                                                                                                                                                                                                              | **Active Peer (Lower RID)** | **Passive Peer (Higher RID)** |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- | ----------------------------- |
| **Listens on 179?**                                                                                                                                                                                                                                        | Yes                         | Yes                           |
| **Initiates TCP?**                                                                                                                                                                                                                                         | Yes                         | No                            |
| **Source Port**                                                                                                                                                                                                                                            | Random (e.g., 49152)        | 179 (in reply)                |
| **Destination Port**                                                                                                                                                                                                                                       | 179                         | Random (reply to initiator)   |
| **Decided By**                                                                                                                                                                                                                                             | **Router-ID (Lowest wins)** |                               |
| 1. R1 (RID 1.1.1.1) → Active → Sends: SYN<br>   Src: 10.0.0.1:49152 → Dst: 10.0.0.2:179<br><br>2. R2 (RID 2.2.2.2) → Passive → Listens on :179 → Replies:<br>   SYN-ACK → Src: 10.0.0.2:179 → Dst: 10.0.0.1:49152<br><br>3. R1 → ACK → Session Established |                             |                               |
 - [ ] BGP is path Vector : neighbor is consider as whole world for updates 

BGP "update source"
	 when BGP tries establish neighbor ship By default it uses outgoing interface IP as source
	 We can change this with command update-source

BGP different tables
	 Adj-RIB-in : The unedited routing information sent by neighbouring routers.
	 Adj-RIB-out : The information the router chooses to send to neighbouring routers.
	 Loc-RIB : T he actual routing information the router uses, developed from Adj-RIBs-In

BGP Loop prevention
		 EBGP - As path list
		 IBGP - Split horizon rule and with RR Cluster iD and Originator ID 

BGP "Route refresh"-------

   Hard Reset
   Dynamic soft reset - Route refresh capability  -- cli needed clear ip bgp 1.1.1.1 soft in 
   Soft** reset using stored information - soft-reconfiguration inbound >                                                                                     cli needed clear ip bgp 1.1.1.1 soft in no RR message 
		 

Router A and Router B are eBGP peers.
Router A adds a new prefix-list to filter incoming routes.
Router A issues clear ip bgp <Router B IP> soft in.
Router A sends a ROUTE-REFRESH message to Router B.
 Router B resends its IPv4 Unicast routes.
 Router A applies the new prefix-list, updating its BGP table\
       
- **Why Needed**: Enables policy updates without session resets, ensuring stability.
- **How It Works**: Negotiates capability, sends ROUTE-REFRESH message, peer resends routes, and new policies are applied.
- **Benefits**: No downtime, low memory usage, widely supported.
		 ******

BGP "Backdoor" -----

	 network 10.10.0.0 0.0.255.255 backdoor
	 purpose  is to change the AD of a particaulr eBGP prefix from 20 to 200
	 not to advertise a new network. the command applies to non-local prefixes

Default route routing
	 >  Default route can be redistributed from other protocol in BGP but we need to use command "==default-information originate" along with redistribute== command
	 >  If we have default route in RIB then we can use network command to announce  route
	 >  To unconditionally announce default route to particular Peer use
	   per peer default originate command (to make it conditional use route-map with this command)

==BGP "Dampening==

 limit the propagation of flapping routes
  to reduce load on router processor caused by flapping routes
  increase route stability
  prevent sustained route oscillations
  command is
  "==bgp dampening half-life reuse-limit suppress-limit maximum-suppress-limit=="
   Max_Penalty = ReuseLimit * 2^(MaximumSuppressTime/Half_Life)

==BGP Maximum Prefix== 
			  neighbor <IP> maximum-prefix <Number> [<Threshold%>] [warning-only]|[restart <minutes>]
       default threshold warning is 75%
       BGP status will be "Idle(PfxCt)"
			 nei <IP> maximum-prefix 8(gives warning first resets BGP if prefix limit exceeds and stuck in Idle(PfxCt))  
			 nei <IP> maximum-prefix 8 restart 1 (attemts to restart session after 1 min)
			 nei <IP> maximum-prefix 8 warning-only (gives warning of prefix limit exceeded by thronging     log message)

BGP Local AS 

[[localasnoprepend replaceas.xlsx]]

 Useful when migrating an autonomous system to a different AS number   R1 --() R2 --- R3
		1 .  router bgp <65000>
			 neighbor <IP> local-as <55410>
			allows neighbor router to peer with 65000
			advertise prefix has AS-path (<65000> <55410>)
			Received prefix has AS-path (<55410> <Neighbor AS>)
			**IMP : local AS command will append local AS in out and in direction while advertisng and receiving prefixes**
        1. neighbor <IP> local-as <55410> no-prepend
			Useful with partial transitions, when part of your AS is using the 55410
			number, and another part is still using the 65000 number
			advertise prefix has AS-path (<65000> <55410>)
			Received prefix has AS-path (<Neighbor AS>)
		      **imp:no repend act in direction  R1 prefix will reach r3 with as path list as 200 and 100 ( 55410 strip off ) ******
        1. neighbor <IP> local-as <55410> no-prepend replace-as
			advertise prefix has AS-path (<55410>)
			Received prefix has AS-path (<Neighbor AS>)  
			  
		2. local-as no-prepend replace-as dual-as
			allow the neighbor to connect to either AS(65000 or 55410)

BGP Remove-Private
 Private AS numbers in range 64512-65535
 neighbor <IP> remove-private-as

 All BGP updates sent over this session are inspected to
 have a sequence of private AS numbers in the beginning of the AS_PATH
 when the private AS sequence is not located in the beginning of the AS_PATH, the stripping will not work
 to make it work use command neighbor <IP> remove-private-as all

			 neighbor <IP> remove-private-as (removes private-as on at the start of as sequence in outbound direction)
			 neighbor <IP> remove-private-as all (removes all private as number from AS-path list
			 neighbor <IP> remove-private-as all replace-as (and replace them with own as number)


![[localasnoprepend replaceas.xlsx]]
