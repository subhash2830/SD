
# 📘 VPLS  with full mesh LDP 

## 🔹 Definition

- **VPLS****:** Virtual Private LAN Service is a Layer‑2 VPN that makes remote sites appear as if they are on the same Ethernet LAN.
    
- **Core concept****:** The SP MPLS core acts like a distributed Ethernet switch.
    
- **Customer view****:** All sites share one broadcast domain.
    

## 🔹 Why VPLS?

- **Business driver****:** Multipoint L2 connectivity for legacy apps, DC interconnect, broadcast‑heavy workloads.
    
- **Design driver****:** Transparent LAN extension without changing CE routing.
    
- **Alternative****:** VPWS = point‑to‑point; VPLS = multipoint.

- **Business driver****Examples:**
    
    - **Legacy ERP systems** that rely on Layer‑2 adjacency between servers in different sites.
        
    - **Data center interconnect (DCI)** between Mumbai and Pune sites for VM migration or storage replication.
        
    - **Broadcast‑intensive applications** like DHCP, ARP, or multicast video streaming across multiple branches.
        
    - **Financial trading networks** needing low‑latency L2 connectivity between exchange gateways.
        
    - **Industrial automation** systems where controllers and sensors must appear on the same VLAN across locations.

## 🔹 How VPLS Works

- **Virtual Forwarding Instance (VFI)****:** Logical construct inside PE representing pseudowire mesh.
    
- **MAC learning****:** PE learns MACs per pseudowire or AC.
    
- **BUM traffic****:** Broadcast, Unknown unicast, Multicast flooded via ingress replication.
    
- **Loop prevention****:** Split horizon stops BUM loops in MPLS core.
    
- **Control plane****:**
    
    - Manual config → full mesh of LDP pseudowires.
        
    - BGP auto‑discovery → scalable membership.
        
    - BGP signaling → service label distribution.
        

## 🔹 Design Considerations

- **Scalability****:** LDP full mesh = O(n²). Large deployments → BGP auto‑discovery.
    
- **Resiliency****:** Depends on MPLS core convergence.
    
- **MTU****:** Must match across VFI.
    
- **Operational impact****:** Manual config requires touching all PEs when adding new members.
    

## 🔹 Interview‑Ready Explanation

> “VPLS is a Layer‑2 VPN where the provider core acts like a distributed Ethernet switch. Each PE runs a **VFI**, which is a virtual bridge domain connecting pseudowires to other PEs. MACs are learned per pseudowire, and BUM traffic is handled with ingress replication and split horizon to prevent loops. From a design perspective, scalability is the key challenge — full mesh LDP pseudowires are fine for small deployments, but for larger networks we use BGP auto‑discovery and signaling. The design choice depends on customer scale, operational model, and whether they need multipoint L2 connectivity versus simpler point‑to‑point VPWS.”

## 🔹 Memory Map (Quick Recall)

Code

```
VPLS → L2VPN → SP core = distributed switch
   ├─ VFI = virtual bridge domain
   │    ├─ Pseudowires (full mesh or BGP auto-discovery)
   │    └─ MAC learning per pseudowire
   ├─ BUM traffic → ingress replication
   │    └─ Split horizon prevents loops
   ├─ Control plane
   │    ├─ LDP (manual, small scale)
   │    ├─ BGP auto-discovery (scalable)
   │    └─ BGP signaling (labels)
   └─ Design considerations
        ├─ Scalability (O(n²) with LDP)
        ├─ MTU consistency
        ├─ Operational overhead (manual config)
        └─ Resiliency depends on MPLS core
```

## 🔹 Comparison Table

|**Service**|**Use Case**|**Control Plane**|**Scalability**|**Loop Handling**|
|---|---|---|---|---|
|**VPWS**|Point‑to‑point L2VPN|LDP tLDP|High (linear)|Not needed (p2p)|
|**VPLS**|Multipoint L2VPN|LDP full mesh / BGP AD|Limited with LDP, scalable with BGP|Split horizon|
|**EVPN**|Modern multipoint L2/L3VPN|BGP|Highly scalable|Built‑in loop prevention (control plane learning)|

[[Pasted image 20260625134733.png]]

[[Basic_VPLS_TLDP_Session.txt 1]]

