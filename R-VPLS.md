# 📘 R‑VPLS (Routed VPLS)

## 🔹 Definition

- **R‑VPLS****:** A variation of VPLS where the PE provides **integrated Layer‑3 gateway functionality** inside the VPLS domain.
    
- Instead of being purely Layer‑2 (like classic VPLS), R‑VPLS allows the PE to act as a **default gateway** for the connected CEs.
    
- Think of it as **VPLS + Routed interface** — bridging plus routing in the same service.
    

## 🔹 Why R‑VPLS?

- **Gateway integration****:** Eliminates the need for an external router to provide L3 services.
    
- **Simplified design****:** Clients in the VPLS domain can get IP addresses and default gateway directly from the PE.
    
- **Service provider use case****:** Useful when offering managed services — provider PE can act as DHCP server, firewall, or gateway.
    

## 🔹 How It Works

- **Bridge Domain (BD):** Provides multipoint Layer‑2 connectivity across PEs (classic VPLS).
    
- **BDI (Bridge Domain Interface):** Logical Layer‑3 interface bound to the BD.
    
    - Number must match the BD ID (like SVI = VLAN ID on a switch).
        
    - Provides IP address, routing, and gateway services.
        
- **Control Plane:** Still VPLS (LDP or BGP for autodiscovery/signaling).
    
- **Data Plane:** Same as VPLS for L2 forwarding, but with added L3 gateway at the PE.
    

## 🔹 Example (Your Lab)

- **R1:**
    
    - Runs VPLS (VFI + bridge‑domain).
        
    - Has `bdi1` with IP 100.1.1.1 → acts as default gateway.
        
    - DHCP pool assigns IPs to clients.
        
- **R3/R5:**
    
    - Run VPLS only (no BDI).
        
    - Forward traffic into BD1.
        
- **Result:**
    
    - Clients get IPs from R1.
        
    - Clients can ping each other (VPLS multipoint).
        
    - Clients can ping internet via R1’s gateway.
        

## 🔹 Difference from Classic VPLS

|**Feature**|**Classic VPLS**|**R‑VPLS**|
|---|---|---|
|**Gateway**|External router needed|PE provides gateway via BDI|
|**MAC learning**|Only L2|L2 + L3 (BDI participates)|
|**DHCP/VRF**|External|Integrated at PE|
|**Use case**|Pure L2 multipoint|L2 multipoint + L3 gateway|

## 🔹 Interview‑Ready Summary

> “R‑VPLS is essentially VPLS with an integrated routed interface. The PE provides a default gateway inside the VPLS bridge‑domain using a Bridge Domain Interface (BDI). This allows clients to receive IP addresses, use DHCP, and route out without needing an external router. The control plane is still VPLS — pseudowires, MAC learning, split horizon — but the PE now also participates in Layer‑3. In design terms, R‑VPLS is chosen when you want to combine multipoint L2 connectivity with integrated L3 gateway services at the PE.”

## 🔹 Memory Map (Quick Recall)

Code

```
R-VPLS
   ├─ VFI + Bridge-domain = VPLS control plane
   ├─ BDI = L3 gateway bound to BD
   │    └─ IP address, DHCP, routing
   ├─ Clients → get IPs + default gateway from PE
   ├─ Still multipoint L2 across PEs
   └─ Design trade-off → simpler, integrated gateway at PE
```