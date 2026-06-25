# 📘 VPLS with LDP and BGP

## 🔹 What Problem Does It Solve?

- **Challenge****:** Traditional VPLS using LDP requires a full mesh of pseudowires between all PEs — operationally heavy and hard to scale.
    
- **Solution****:** Use **BGP autodiscovery** to dynamically identify participating PEs, while still using **LDP for data‑plane signaling**.
    
- This hybrid model keeps the simplicity of LDP for label exchange but removes manual neighbor configuration.
    

## 🔹 Why This Design Matters

- **Operational simplicity****:** No need to manually list every PE under the VFI.
    
- **Scalability****:** Adding a new PE automatically updates all others via BGP advertisements.
    
- **Consistency****:** The VPN ID (from AGI) ensures all PEs belong to the same broadcast domain.
    
- **Flexibility****:** You can still mix manual and autodiscovered neighbors if needed (FEC 128 vs FEC 129).
    

## 🔹 How It Works (Design View)

- **BGP L2VPN/VPLS AFI/SAFI****:** Used for autodiscovery — each PE advertises a route identifying its participation in a given VPLS instance.
    
- **Route Reflector (RR)****:** Central point (R4 in your lab) that reflects these advertisements to all other PEs.
    
- **VFI context****:** Defines the VPLS instance (`vpn id 100`) and specifies `autodiscovery bgp signaling ldp`.
    
- **Data plane****:** Still uses LDP pseudowires for forwarding — identical to traditional VPLS.
    
- **FEC type****:**
    
    - Manual config → FEC 128 (basic pseudowire).
        
    - BGP autodiscovery → FEC 129 (includes source/target info).
        

## 🔹 Key Design Components

| **Component**                 | **Purpose**              | **Design Insight**                       |
| ----------------------------- | ------------------------ | ---------------------------------------- |
| **RD (Route Distinguisher)**  | Uniqueness per PE        | Prevents duplicate BGP updates           |
| **RT (Route Target)**         | Membership control       | Ensures only matching VFIs import routes |
| **AGI (Attachment Group ID)** | Identifies VPLS instance | Format `<ASN>:<VPN ID>`                  |
| **L2VPN RID**                 | Router identifier        | Used in NLRI prefix, not as endpoint     |
| **Next‑hop**                  | Endpoint for pseudowire  | Derived from BGP update                  |
| **VPN ID**                    | VCID equivalence         | Must match across all PEs                |
 [[VPLS+BGP-AD+LDP_SIgnalling.txt]]

## 🔹 Interview‑Ready Summary

> “In a BGP‑only VPLS design, BGP handles both autodiscovery and service label signaling. Each PE advertises its participation in the VPLS instance using L2VPN/VPLS routes, and the Route Reflector distributes this information. The RD ensures uniqueness, the RT controls membership, and the AGI identifies the VPLS instance. Unlike the hybrid model, there’s no LDP — BGP itself carries the service label. The data plane remains the same: bridging, MAC learning, and split horizon. This design is chosen when scalability and operational simplicity are priorities, especially in networks already running large‑scale iBGP.


## Memory Map (Quick Recall)

Code

```
VPLS → L2VPN → SP core = distributed switch
   ├─ Hybrid (BGP AD + LDP signaling)
   └─ BGP-only
        ├─ BGP handles autodiscovery + signaling
        ├─ RD → uniqueness
        ├─ RT → membership
        ├─ AGI → VPLS instance ID
        └─ Service label signaled via BGP
Data plane unchanged → bridging, MAC learning, split horizon
Verification → EIGRP neighbors across VPLS
```
======================================================