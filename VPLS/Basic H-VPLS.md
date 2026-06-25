![[Pasted image 20260625152348.png]]
[[Basic H-VPLS.txt]]
[[Basic H-VPLS + BGP ]]
[[Basic H-VPLS + QnQ]]
[[Basic H-VPLS with Redundancy]]
# 📘 H‑VPLS (Hierarchical VPLS) — Design Notes

## 🔹 What is H‑VPLS?

- **Definition****:** A hierarchical extension of VPLS designed to improve scalability.
    
- **Core idea****:** Split the PE roles into **U‑PEs (User‑facing)** and **N‑PEs (Network‑facing)**.
    
- **Benefit****:** Reduces the number of pseudowires and replication overhead in large deployments.
    

## 🔹 Why H‑VPLS?

- **Scalability:** Flat VPLS requires a full mesh → grows as O(n²). With 100 PEs, ~5,000 pseudowires.
    
- **Efficiency:** Less replication of BUM traffic.
    
- **Operational simplicity:** U‑PEs only connect to one N‑PE, not to every other PE.
    

## 🔹 How H‑VPLS Works

- **U‑PEs****:** Connect directly to CEs. Form a single PW to an N‑PE. Split horizon stays ON.
    
- **N‑PEs****:** Form a full mesh with other N‑PEs. Treat U‑PEs like ACs. Split horizon is turned OFF for PWs facing U‑PEs.
    
- **Core VPLS:** Exists only between N‑PEs. U‑PEs are “attached” into this core via their N‑PE.
    
- **Hierarchy:**
    
    - U‑PE ↔ N‑PE = single PW.
        
    - N‑PE ↔ N‑PE = full mesh.
        

## 🔹 Data Plane Behavior

- **Flat VPLS:** Ingress PE pushes two labels → transport + service label → direct to egress PE.
    
- **H‑VPLS:** Traffic stitched across multiple LSPs:
    
    - CE1 → U‑PE R1 → LSP to N‑PE R2.
        
    - R2 bridges → forwards via LSP to R4.
        
    - R4 bridges → forwards via LSP to R3.
        
    - R3 → CE2.
        
- **Trade‑off:** Control plane is more scalable, but data plane is more complex (multiple bridging hops).
    

## 🔹 MAC Scalability

- **Flat VPLS:** Each PE learns MACs only for domains it participates in.
    
- **H‑VPLS:** N‑PEs participate in all domains → must learn all CE MACs. CAM table load shifts toward N‑PEs.
    

## 🔹 Interview‑Ready Summary

> “H‑VPLS is a scalability technique for VPLS. Instead of every PE forming a full mesh of pseudowires, we introduce hierarchy: U‑PEs connect to customers and only form a pseudowire to one N‑PE, while N‑PEs form a full mesh among themselves. From the N‑PE’s perspective, a U‑PE looks like an access circuit, so split horizon is disabled on those links. This reduces the number of pseudowires and replication overhead, but adds complexity in the data plane because traffic is stitched across multiple LSPs and N‑PEs must perform bridging. The trade‑off is clear: simpler control plane, more complex data plane.”

## 🔹 Memory Map (Quick Recall)

Code

```
H-VPLS
   ├─ U-PE → user-facing, connects to CE
   │    └─ PW to one N-PE (split horizon ON)
   ├─ N-PE → network-facing, full mesh with other N-PEs
   │    └─ Treat U-PE PW like AC (split horizon OFF)
   ├─ Core VPLS → only N-PEs
   ├─ Control plane → scalable (fewer PWs)
   ├─ Data plane → stitched LSPs, multiple bridging hops
   └─ MAC learning → N-PEs learn all CE MACs
Verification → N-PE shows full mesh + U-PE PWs with split horizon disabled
```
======================================================================================================================
# 📘 H‑VPLS with BGP Autodiscovery — Design Notes


- **N‑PE domain****:** N‑PEs form a full mesh. Here, **BGP autodiscovery** can be used to automatically discover N‑PE ↔ N‑PE pseudowires.
    
- **U‑PE ↔ N‑PE links****:** Cannot use BGP autodiscovery here, because it forces split horizon ON.
    
    - For U‑PE ↔ N‑PE, split horizon must be OFF (otherwise BUM traffic loops).
        
    - This can only be done with **manual pseudowire neighbors under the bridge domain**.
        

## 🔹 Why Not Pure BGP Signaling?

- **Mixing protocols****:** You cannot mix BGP‑signaled pseudowires with LDP‑signaled pseudowires in the same bridge domain.
    
- **Result:** Even though autodiscovery is BGP, the signaling of labels remains **LDP‑based** for all pseudowires.
    

## 🔹 Best Practical Design

- **N‑PE ↔ N‑PE:** Use **BGP autodiscovery** to reduce manual config.
    
- **U‑PE ↔ N‑PE:** Use **manual LDP pseudowires** with split horizon disabled on the N‑PE side.
    
- **Overall:** All pseudowires in the topology are **LDP‑signaled**, but N‑PE membership is discovered via BGP.
    

## 🔹 Verification

- **N‑PEs:** Show full mesh of autodiscovered pseudowires (LDP signaled).
    
- **U‑PEs:** Show manual pseudowires to N‑PEs (split horizon disabled at N‑PE).
    
- **CEs:** Verify EIGRP neighborships across the VPLS domain → proves multipoint L2 adjacency.
    

## 🔹 Interview‑Ready Summary

> “In H‑VPLS, we can use BGP autodiscovery for the N‑PE full mesh, but not for U‑PE ↔ N‑PE pseudowires. That’s because autodiscovery automatically enables split horizon, and we need split horizon disabled on N‑PE ↔ U‑PE links. Therefore, those pseudowires must be manually configured. Also, we cannot mix BGP‑signaled and LDP‑signaled pseudowires in the same bridge domain. So the best design is: N‑PEs use BGP autodiscovery for membership, but all pseudowires are still LDP‑signaled. This balances scalability in the core with correct loop prevention at the edge.”

## 🔹 Memory Map (Quick Recall)

Code

```
H-VPLS with BGP AD
   ├─ N-PE ↔ N-PE
   │    └─ BGP autodiscovery (membership)
   │    └─ LDP signaling (labels)
   ├─ U-PE ↔ N-PE
   │    └─ Manual pseudowires
   │    └─ Split horizon OFF at N-PE
   └─ Rule → cannot mix BGP-signaled + LDP-signaled PWs in same BD
Verification → N-PE full mesh via BGP AD, U-PE manual PWs, CE EIGRP neighbors
```

======================================================================================================================
# H‑VPLS with QinQ Access — Design Notes

## 🔹 Why QinQ in H‑VPLS?

- **Scalability****:** Each U‑PE can support thousands of customers (up to 4094 VLANs) by stacking an outer VLAN tag.
    
- **Separation****:** Outer tag distinguishes customers at the NNI (U‑PE ↔ N‑PE).
    
- **Efficiency****:** U‑PEs do simple L2 bridging, leaving VPLS control plane to N‑PEs.
    

## 🔹 How the Flow Works (Step‑by‑Step)

1. **CE → U‑PE (UNI):**
    
    - CE sends normal Ethernet frame (single VLAN or untagged).
        
    - U‑PE pushes an **outer VLAN tag** (QinQ).
        
    - This outer tag identifies the customer at the NNI.
        
2. **U‑PE → N‑PE (NNI):**
    
    - Traffic arrives **double‑tagged** (inner = customer VLAN, outer = QinQ service VLAN).
        
    - Both UNI and NNI belong to the same bridge domain on the U‑PE.
        
    - U‑PE pops the outer tag on egress toward CE.
        
3. **N‑PE (Core VPLS):**
    
    - Sees traffic as double‑tagged.
        
    - Classifies based on **outer tag**.
        
    - Runs a **normal VPLS domain per customer** (using BGP for autodiscovery + signaling).
        
    - VFIs and pseudowires exist only between N‑PEs.
        
4. **N‑PE ↔ N‑PE:**
    
    - Full mesh of pseudowires between N‑PEs.
        
    - BGP handles both autodiscovery and signaling.
        
    - Each customer gets its own VPLS instance across the N‑PE mesh.
        
5. **CE ↔ CE:**
    
    - CE1 can only reach CE2 if they belong to the same QinQ outer tag / VPLS domain.
        
    - CE3 cannot reach CE12 unless they share the same outer tag.
        
    - This proves **customer isolation**.
        

## 🔹 Interview‑Ready Summary

> “In H‑VPLS with QinQ, the access network does pure Layer‑2 bridging. U‑PEs push an outer VLAN tag per customer, allowing thousands of customers per U‑PE. At the NNI, traffic is classified by the outer tag. The N‑PEs then form a normal VPLS domain per customer, using BGP for both autodiscovery and signaling. This design keeps U‑PEs simple and scalable, while N‑PEs handle all VPLS control plane functions. The trade‑off is that N‑PEs must maintain VFIs for all customers, but the access layer remains lightweight and efficient.”

## 🔹 Memory Map (Quick Recall)

Code

```
QinQ + H-VPLS
   ├─ CE → U-PE (UNI)
   │    └─ U-PE pushes outer VLAN tag
   ├─ U-PE → N-PE (NNI)
   │    └─ Double-tagged traffic
   │    └─ Same bridge-domain (UNI + NNI)
   ├─ N-PE
   │    └─ Classify by outer tag
   │    └─ Run VPLS per customer
   │    └─ BGP for autodiscovery + signaling
   ├─ N-PE ↔ N-PE
   │    └─ Full mesh pseudowires
   └─ Verification
        ├─ U-PE → MACs on UNI/NNI
        ├─ N-PE → VFIs + bridge tables
        └─ CE → isolation confirmed
```

======================================================================================================================
# 📘 H‑VPLS with Redundancy — Design Notes

## 🔹 The Challenge

- In **flat VPLS**, redundancy is usually handled by the control plane (multiple pseudowires, MAC learning, etc.).
    
- In **H‑VPLS**, when you want **N‑PE redundancy**, you need pseudowire redundancy.
    
- But: pseudowire redundancy is not supported inside a VPLS bridge‑domain.
    
- **Result:** At the U‑PE, you must fall back to **VPWS redundancy** instead of VPLS.
    

## 🔹 How It Works

- **U‑PE (R1):**
    
    - Configured with **VPWS xconnect context**.
        
    - Defines two pseudowire members (to R2 and R4).
        
    - Assigns **priority** (R2 = primary, R4 = backup).
        
    - No MAC learning at R1 — it’s point‑to‑point VPWS.
        
- **N‑PEs (R2, R4):**
    
    - Treat R1 as a U‑PE.
        
    - Configure bridge‑domain membership with R1 as a pseudowire endpoint.
        
    - Continue to run normal VPLS among themselves.
        
- **Failover:**
    
    - If R2 fails (e.g., Lo0 shut), R1 switches pseudowire to R4.
        
    - Traffic resumes with minimal packet loss (typically one ping).
        
    - When R2 comes back, R1 switches back to primary.
        

## 🔹 Optimization

- **Redundancy delay timers:**
    
    - `enable-delay` → how long to wait before activating standby PW.
        
    - `disable-delay` → how long to wait before switching back to primary PW.
        
- Tuning these timers reduces packet loss during switchover and prevents flapping when the primary recovers.
    

## 🔹 Data Plane Behavior

- **Normal VPLS:** U‑PE participates in MAC learning, multipoint bridging.
    
- **With redundancy:** U‑PE runs VPWS → no MAC learning at U‑PE.
    
- **N‑PEs:** Still perform bridging and MAC learning in the VPLS core.
    
- **Trade‑off:** Redundancy achieved, but U‑PE loses multipoint capability (can’t attach multiple CEs).
    

## 🔹 Verification

- Shut down R2’s loopback → R1 fails over to R4.
    
- Ping between CE1 and CE2 → only one ping lost.
    
- Bring R2 back → R1 switches back to primary.
    
- With tuned timers → no packet loss on recovery.
    

## 🔹 Interview‑Ready Summary

> “In H‑VPLS, N‑PE redundancy is achieved using pseudowire redundancy. But since pseudowire redundancy isn’t supported inside a VPLS bridge‑domain, the U‑PE must use VPWS redundancy instead. This means the U‑PE loses multipoint capability and acts as a point‑to‑point device. The N‑PEs continue to run VPLS among themselves. Failover is handled by switching pseudowires between primary and backup N‑PEs, with timers tuned to minimize packet loss. The trade‑off is clear: redundancy is achieved, but at the cost of MAC learning and multipoint capability at the U‑PE.”

## 🔹 Memory Map (Quick Recall)

Code

```
H-VPLS Redundancy
   ├─ U-PE (R1)
   │    └─ VPWS xconnect context
   │    └─ Primary PW → R2
   │    └─ Backup PW → R4
   │    └─ No MAC learning
   ├─ N-PEs (R2, R4)
   │    └─ Normal VPLS bridge-domain
   │    └─ Treat R1 as U-PE
   ├─ Failover
   │    └─ R2 down → R1 switches to R4
   │    └─ Minimal packet loss
   │    └─ Timers optimize recovery
   └─ Trade-off
        ├─ Redundancy achieved
        ├─ Simpler failover
        └─ U-PE loses multipoint capability
```

=================================================
[[Basic H-VPLS + IP routing]]
# 📘 VPLS with BDI (Bridge Domain Interface)



## 🔹 Explanation (Design Point of View)

- **BDI (Bridge Domain Interface):**
    
    - Acts like an SVI on a switch.
        
    - Number must match the bridge‑domain ID.
        
    - Provides L3 gateway for clients inside the VPLS.
        
- **Operation:**
    
    - CE sends traffic → enters U‑PE via Gi4 service instance.
        
    - Traffic mapped into BD1 → forwarded into VFI VPLS1.
        
    - VFI builds pseudowires to other PEs (R3, R5) using BGP autodiscovery + signaling.
        
    - Remote CEs receive traffic via their service instances (Gi6).
        
    - BDI1 provides default gateway + DHCP → clients get IPs automatically.
        
- **Result:**
    
    - All CEs in BD1 form a full mesh VPLS.
        
    - Clients get IPs from DHCP pool.
        
    - Clients can ping each other (L2 adjacency) and reach internet via default gateway (100.1.1.1).



===================================================================
[[MAC Protection]]
# 📘 VPLS MAC Limit & Aging — Design Notes

## 🔹 CLI Breakdown

### **Global Setting**

bash

```
ethernet mac limit action flooding disable
```

- **Default behavior:** When MAC limit is reached, router stops learning new MACs but floods unknown unicast traffic → keeps connectivity but wastes bandwidth.
    
- **This command:** Disables unknown unicast flooding globally once MAC limit is hit → prevents bandwidth waste.
    

### **Per Bridge‑Domain Settings**

bash

```
bridge-domain 100
 mac aging-time 150
 mac limit maximum addresses 100
```

- **MAC aging‑time:** Default = 300s (5 min). Here reduced to 150s (2.5 min). Faster cleanup of stale MACs.
    
- **MAC limit:** Max 100 MACs in BD100. Protects SP core from resource exhaustion.
    

### **Optional: Disable MAC Learning**

bash

```
bridge-domain 100
 no mac learning
```

- Turns BD into **hub‑like behavior**.
    
- All traffic flooded out all ports except ingress.
    
- No MAC table entries → no selective forwarding.
    

## 🔹 Why This Design?

- **Protection****:** Prevents customers from exhausting SP router resources with too many MACs.
    
- **Efficiency****:** Faster aging reduces stale entries, keeps CAM table lean.
    
- **Control****:** Disabling unknown unicast flooding avoids bandwidth waste when MAC limit is exceeded.
    

## 🔹 Verification & Behavior

1. **Normal operation:**
    
    - MACs learned, aged out after 150s.
        
    - Up to 100 MACs stored in BD100.
        
2. **Limit test:**
    
    bash
    
    ```
    bridge-domain 100
     mac limit maximum addresses 2
    ```
    
    - Only 2 MACs allowed.
        
    - When exceeded → log message on R1.
        
    - CE3 cannot ping CE1 (unicast fails).
        
    - EIGRP neighborship stays up (multicast still floods).
        
3. **Remove global flooding disable:**
    
    bash
    
    ```
    no ethernet mac limit action flooding disable
    ```
    
    - Unknown unicast flooding resumes.
        
    - Even with only 2 MACs stored, unicast traffic works (flooded).
        
    - Bandwidth wasted, but connectivity restored.
        

## 🔹 Interview‑Ready Summary

> “In VPLS, every customer MAC consumes resources on the SP core. To protect scalability, we can set per‑bridge‑domain MAC limits and aging timers. By default, when the limit is reached, unknown unicast traffic is flooded to preserve connectivity, but this wastes bandwidth. A better design is to globally disable flooding when the limit is hit, so excess MACs don’t impact the core. We can also tune aging timers to clean stale entries faster. If MAC learning is disabled entirely, the BD behaves like a hub, flooding all traffic. This design balances scalability, efficiency, and customer experience.”

## 🔹 Memory Map (Quick Recall)

Code

```
MAC Control in VPLS
   ├─ Global → disable flooding when limit hit
   ├─ Per-BD → set MAC limit + aging time
   ├─ Default → flood unknown unicast if limit exceeded
   ├─ Option → disable MAC learning (hub mode)
   ├─ Verification → logs, ping tests, EIGRP neighbors
   └─ Trade-off → bandwidth vs connectivity
```