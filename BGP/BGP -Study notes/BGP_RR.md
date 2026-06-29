==why required RR ==

Main reason scaling full mesh iBGP 
EBGP loop prevention by AS number , But but in iBGP update AS path remain same 
Also iBGP update never forward to another iBGP neighbor ( Split horizon )
This may lead to looping  and lead to have logical Full mesh BGP (IBGP )  i.e  n (n-1 )/2  sessions


==How it help scaling network== 

introducing RR reduce the number of peering N-1 only
A route reflector is BGP router that is allowed to break the iBGP split horizon rule. 
Route reflectors can advertise updates received from an iBGP peer to another iBGP peer under specific conditions.
Topology changes from full mesh to Hub and Spoke

 ==What is RR and where to use and how it works==

 What >   Route reflector is BGP speaker who override BGP split horizon ( thus scale large n/w ) and Help in loop prevention in IBGP topology

 where>  prefer only in IBGP networks 
 Route reflector reflects  iBGP learned routes to other iBGP neighbor thus scaling large network. 

  So scale large network we think RR as it offers an alternative to the logical full-mesh requirement of internal border gateway protocol (IBGP).

==Working ::==   
 The internal peers that connect to an RR are classified as RR client peers. 
 Every other iBGP router that is not an RR or an RR client is classified as a non-client peer. 
  An RR along with its client peers form a cluster.
  
	  1 > Routes received from a Route-Reflector-client is reflected to other clients and 
		non-client neighbors.
	  2> Routes received from non-client neighbors are reflected to Route-Reflector-client 
	   neighbors only.
	  3 >Setting the Originator-ID attribute in the reflected update if it is not already
	   set.
	 4 > Adding the Cluster-ID  to the Cluster-list attribute in the reflected update.

   Route Reflector reflects routes considered as best routes only.  If more than one update is received for the same destination,  only the BGP best route is reflected. A Route Reflector is not allowed to change any attributes of the reflected routes including the next-hop attribute.

==how Loop Prevention  Done:==

If a router received an iBGP route with the Originator-ID attribute set to its own router-id, the route is discarded.
If a route reflector receives a route with a cluster-list attribute containing its cluster-id, the route is discarded.



==How many RR ==

 Its depended on types of services  ipv4 , VPN ( L3 and L2 ) or Internet depend on that you have IP RR , VPN RR and INTERNET RR

 If there is an attack towards IP RR, it will affects VPN customers as well. This concept is called ==fate-sharing.==

 this can be avoided by RRs pair per services 
	 IF single : its single point of failure lead to outage
	 Multiple :  two RR for all services provide redundancy but carry all traffic overload like internet and VPN etc. many more
    But can thing separates RR Pair for VPN + Ipv4  and Internets 

 ( independent Vpn and internate traffic provide segregation and thus comparatively less overload  )


==In band or out of band RR==

if in path  Not recommended as RR will carry traffic. Basic Function of RR is Single focal point who reflects routes in control plane

Out of band : As all ISP uses mpls lable switching by means of LDP and RSVP ( Thus avoding loops based on IGP )

RR can be place outof band Avoding forwarding loops 

Next stages like RR function can be virtualised in Vm like nFV.

==Placement of RR ==

 RRs and its client form cluster. so Clients are full mesh with RRs Providing redundancey

 Can Have Hierarchical RRs for large networks

 Means ISP can have multiple POP location depends on geography likes east west north and south pop location ( contain multiple PE at POP )

 Each POP location PEs are full mesh with Respective location RRs in full mesh 

 These RRs are first line RRS

 First stage RRs can connect Central RRs( Second line  RRs )  in full mesh 

==RR Issue==

 ==1 ) Slow convergence :== 

 processing and propagation of routes at RR add additional delays,
 PE 1---------->  PE3
 PE2 --------->  PE3

 In case of full mesh PE3 have routes from PE1 and PE2 but used best one by PE1 only in RIB BGP table have both failures of Route from PE1 only cause commutation on PE2  and PE1 ( withdraw )

 With RR entry  Only PE1 advertised ( selection criteria ) by RR 
 in failure PE1 ( withdraw msg )  RR ( withdraw and add PE2 ) and PE2 advertised 
 One more hop processing and propagation delay introduce ( BGP MRAI timer )  


==2 ) Path hindring==

  only share best route to BGP clints
  BGP Route Reflector , both in IP and MPLS environment, selects only one path for the destination, 
  install only this path in the routing table and advertise this path to Route Reflector Clients.

  MPLS VPN can use per site per vrf unique RD which enable advertising both Routes ( in case of PE1-cE1-PE2 ) to pe3 via ( PE1 and PE2 )

==Multiple solution :==

  BGP Route target constraint RFC, i.e. RFC 4684 

 3 )Suboptimal Routing and major factor in Hot potato Routing ( exit AS in nearest exit pointof own AS)

 ==InterPOP suboptimal Routing :== 

    POP A  ---< 10 >--- RR wst cost <------> RR East Cost -- < 20 > --- Pop B

  if any route P learn on Pop A and Pop B , Both advertised it to RRs ( East and West ) with iBGP cost 10 and 20 for A and B respectively
   assuming all other parameter being same in path selection

  Thus Both RR prefer POP A as best and advertised this next hop ( POP A ) to all client C and D selection lowest IGP cost ( BGP path selection )

  This is useful for west cost Pop like POP C as chooses nearest exit point

  for POP D need to sent traffic throgh AS towared to POP A unecessarly ( Better placement of RR ( at centre location ) may use POP B )

   ==IntraPOP suboptimal Routing== 

   POP A  ------- <cost : 30> -------RRs east cost  ----<cost : 20>------- Pop B ( east side pop)

   POP C

   IF ISP place RR at one of edge location ( ==here RR at East cost and  NO RR West cost==). prefix learn on both edge point likely
   east and west pop namely POPB an POP A . Advertised  nearest igp  to RRS ( east cost ) is 20 and 30 respectively

   SO POPS@ west cost always uses POP B as exit point even though they can use POP A  lead to suboptimal routing in intra area domain

 ==4 )Forwarding Loops==

Though this could be a problem with the introduction of RR’s but is only limited to networks doing IP forwarding.

ISP rely on label forwarding aren’t exposed to forwarding loops.

5 )  Routing Oscillations

 refer problem with MED osilattion https://tools.ietf.org/html/rfc3345 

  Solution '

  1 Uses MED only when needed. By default MED attribute is reset to zero for any path received from an external Peer. 

 2 Intra cluster distance (i.e. Distance between RR and its clients)  is always less than Inter-cluster distance (i.e. distance between RR’s clusters)

 3 )Routers have “deterministic-med” and “always compare-med” knobs turned on to avoid the inconsistent route selection.

 5 }  Route Reflector prevents BGP Fast Reroute

  As said only best path advertised 

 --------------------------------------------------------------------------------------------

 Two RR 

  1) Having different cluster id and RR Peer each others

      Topology as follow 

                       RR 1              RR2 
							
                 Router A                  D

RR-clients namely A and   D have full mesh with RR 1 and RR2

 Client have routers from RR1 and RR2 and Each RRs Having 2 additional routes ( one from client and one from RR peers )

  2 ) Having Same cluster id

   Topology as follow 

                       |RR1                   RR2

                      Router A                  D

 Client have routers from RR1 and RR2 and Each RRs Having only one   routes from client routes from RRs to each other is discarted due to same cluster id )


Analogy

AS loop 
spilt horizone---
Full mesh
Scalability
RR 


# 🧠 Route Reflector —  Notes

---

### 🔹 Core Problem Narrative

iBGP was intentionally designed to **not modify AS_PATH**, which indirectly removes its strongest loop prevention mechanism. To compensate, the protocol enforces **split-horizon behavior**, where routes learned from one iBGP peer are not forwarded to another. This guarantees safety, but at a very high cost—**reachability requires full mesh**.

At small scale, this works.  
At large scale, this becomes a control-plane explosion:

- Sessions grow quadratically → `N(N-1)/2`
- Update churn multiplies
- Convergence becomes unstable
- Operational handling becomes fragile

The real issue is not just session count, but **lack of hierarchy** in the control plane.

👉 Route Reflector is introduced to **inject hierarchy into iBGP**, while **preserving loop safety using attributes instead of topology**.

---

### 🔹 Behavioral Shift Introduced by RR

Once RR is deployed, iBGP stops being a flat topology and becomes a **policy-driven hierarchy**.

- Some routers become **clients**
- One (or more) routers become **reflection points**
- Full mesh requirement is removed

The RR **breaks split-horizon**, but not freely—it does so using strict rules:

- Client-learned routes → widely distributed (clients + non-clients)
- Non-client routes → only sent to clients

This asymmetry is intentional.  
It prevents uncontrolled route propagation and ensures **predictable dissemination patterns**.

---

### 🔹 Loop Prevention Redefined

Since topology no longer guarantees safety, BGP introduces **stateful attributes**:

- **Originator-ID**
    
    - Stamps the original source
    - Prevents a route from coming back to origin
- **Cluster-list**
    
    - Tracks RR traversal history
    - Prevents reflection loops across clusters

This is a key architectural shift:

```
Before RR → Loop prevention = topology constraint
After RR  → Loop prevention = attribute validation
```

👉 Every router now actively verifies loop conditions instead of passively relying on topology.

---

### 🔹 What Actually Changes Internally

RR does **not behave like a router in forwarding sense**.

- Does NOT modify NEXT_HOP
- Does NOT sit in forwarding path (ideally)
- Only reflects **best path**

This is critical: 👉 RR is a **control-plane scaler**, not a data-plane optimizer

---

### 🔹 Scaling Impact

Without RR:

```
100 routers → ~4950 iBGP sessions
```

With RR:

```
100 routers → ~99 sessions per RR
```

But the deeper impact is:

- Reduced adjacency state
- Reduced update fan-out
- Controlled route dissemination

👉 This transforms BGP from **fully distributed → semi-centralized system**

---

# 🌍 Real Network Behavior (Where Problems Start)

---

### 🔸 Path Hiding (Most Critical Effect)

RR reflects only its **best path**.

Meaning:

- Alternate paths exist → but are invisible to clients
- Backup paths are not pre-installed

Impact:

- Slower failover
- No immediate reconvergence
- Breaks fast-reroute expectations

👉 Clients depend entirely on RR’s perspective

---

### 🔸 Convergence Delay

In full mesh:

```
PE1 → PE3 (direct alternative already known)
```

With RR:

```
PE1 → RR → PE3
Failure → RR re-evaluates → sends new path
```

Extra steps:

- Withdraw processing
- Best-path recalculation
- Propagation delay (MRAI impact)

👉 RR introduces **serialization of convergence**

---

### 🔸 RR Placement Bias (Hidden Design Risk)

Route selection depends on:

- IGP distance to NEXT_HOP
- RR visibility

If RR is poorly placed:

- It prefers a geographically closer path
- Entire network inherits that decision

Result:

- Traffic may cross backbone unnecessarily
- POP-local exits may be ignored

👉 This creates **systematic suboptimal routing**

---

### 🔸 Inter-POP vs Intra-POP Impact

- Inter-POP:
    
    - Wrong exit selection across regions
    - Backbone utilization increases
- Intra-POP:
    
    - Edge-placed RR pulls traffic toward itself
    - Local exits ignored

👉 RR position effectively influences **traffic engineering**

---

### 🔸 Fate Sharing

Single RR handling:

- Internet
- VPNv4
- L2VPN

Failure ⇒ All services impacted

Solution:

- Service-specific RR
    - IP RR
    - VPN RR
    - Internet RR

👉 Separation = isolation of failure domains

---

### 🔸 Hierarchical RR (At Scale)

Large ISP design:

```
          Core RR
        /         \
   Regional RR   Regional RR
     /   \         /     \
    PE   PE       PE     PE
```

Benefits:

- Limits update scope
- Reduces churn impact
- Scales geographically

Tradeoff:

- Adds convergence layers
- Increases path hiding probability

---

# ⚖️ Design Trade-Off Summary

|Benefit|Cost|
|---|---|
|Scales iBGP|Path hiding|
|Reduces sessions|Slower convergence|
|Enables hierarchy|Suboptimal routing|
|Simplifies config|RR placement critical|

👉 RR is always a **controlled compromise**, never a perfect solution

---

# 🧪 Design Thinking (CCDE Insight)

An architect does NOT ask:  
“Should I use RR?”

Instead asks:

- Where should RR sit in IGP topology?
- How many clusters are required?
- Should services share RR or be isolated?
- Do I need Add-Path to mitigate path hiding?
- What failure behavior is acceptable?

👉 RR design is about **controlling side effects**, not just enabling scaling

---

# 🧠 Memory Map (Fast Recall)

```
RR
│
├── Problem
│   ├─ No AS_PATH change in iBGP
│   ├─ Split horizon
│   └─ Full mesh explosion
│
├── Concept
│   ├─ Break split horizon
│   ├─ Introduce hierarchy
│   └─ Reflect best path
│
├── Safety
│   ├─ Originator-ID
│   └─ Cluster-list
│
├── Behavior
│   ├─ Client vs Non-client rules
│   ├─ Best path only
│   └─ Control-plane only role
│
├── Risks
│   ├─ Path hiding
│   ├─ Slow convergence
│   ├─ Suboptimal routing
│   └─ Fate sharing
│
└── Design
    ├─ Out-of-band RR
    ├─ Hierarchical
    ├─ Multi-RR
    └─ Service separation
```