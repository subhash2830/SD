!
Three deployment scenarios are possible with CSC architecture:

-   Customer carrier is not running MPLS inside its POP sites.
			- Customer carrier routers CSC_CEs connected to backbone carrier are configured as RR and IBGP is formed between them
               LSP will be formed between 2 RR
-   Customer carrier is running MPLS inside its POP sites.
			- customer carrier edge router can directly form IBGP or we can use same approach used when customer carrier not running MPLS 
               but LSP will be formed between customer carrier PE to PE
-   Customer carrier is providing MPLS VPN services to user sites.
			-MPIBGP is formed between customers carriers RR
              in this case LSP will be end to end only thing different is 3 labels when packet travelling in backbone carrier.

why we need to run LDP between interfaces connecting backbone carrier and customer carrier
          AS we are not exchanging customer routes learned by customer carrier with backbone carrier


---------------------------------------------------------------------
# Carrier Supporting Carrier (CSC) – 

  

## 1. What is CSC (Carrier Supporting Carrier)  

  

- What: CSC is an MPLS architecture where one service provider (backbone carrier) transports the MPLS traffic of another service provider (customer carrier).  

- Why: Allows smaller or regional providers to use a larger provider’s backbone without exposing their customer routes.  

- How:  

- Customer carrier runs MPLS/VPN  

- Backbone carrier only transports labeled traffic (does not learn customer routes)  

- Risk:  

- Label complexity, troubleshooting difficulty  

- Dependency on backbone provider  

- Example:  

- Tier-2 ISP uses Tier-1 ISP backbone to carry its MPLS VPN traffic  

- Takeaway:  

CSC separates **transport (backbone)** from **services (customer carrier)**  

  

👉 Interview Angle:  

CSC allows one provider to carry another provider’s MPLS traffic without exchanging customer routes, using label stacking.  

  

---  

  

## 2. Why CSC is Needed  

  

- Smaller SP cannot build nationwide backbone  

- Need:  

- Scalability  

- Isolation of routing domains  

- Secure transport of VPN traffic  

  

👉 Key Idea:  

> Backbone carries traffic, but does NOT participate in customer routing  

  

---  

  

## 3. Important Building Block  

  

- MPLS Label Stack:  

- Outer label → Backbone transport  

- Inner label(s) → Customer carrier routing  

  

👉 No route exchange between carriers required  

  

---  

  

## 4. Why LDP is Required Between Carriers  

  

### Problem:  

- Backbone must forward packets using labels  

- But:  

- Customer routes are NOT shared with backbone  

  

### Solution:  

- Use LDP between CSC-CE and PE  

- Creates LSP for transport only  

  

👉 Conclusion:  

> LDP builds transport path, not routing knowledge  

  

---  

  

## 5. CSC Deployment Scenarios  

  

---  

  

## 🔷 Scenario 1: Customer Carrier NOT Running MPLS in POP  

  

- What:  

- Customer carrier routers act like IP routers (no MPLS internally)  

  

- How:  

- CSC-CE routers:  

- Act as Route Reflectors (RR)  

- Run IBGP between them  

- MPLS LSP:  

- Built between backbone PEs (RR-like behavior)  

  

- Key Behavior:  

- Backbone carries labeled traffic  

- Customer uses IP-based routing internally  

  

- Risk:  

- Limited scalability in customer network  

  

- Takeaway:  

MPLS only used in backbone, not in customer internal network  

  

---  

  

## 🔷 Scenario 2: Customer Carrier Running MPLS in POP  

  

- What:  

- Customer has its own MPLS network  

  

- How:  

- Customer PEs form IBGP directly  

- MPLS LSP:  

- Formed between customer PEs  

  

- Key Behavior:  

- Full MPLS end-to-end inside customer network  

- Backbone acts only as transport  

  

- Risk:  

- More complexity but better scalability  

  

- Takeaway:  

Customer maintains MPLS control; backbone provides connectivity  

  

---  

  

## 🔷 Scenario 3: Customer Carrier Providing MPLS VPN Services  

  

- What:  

- Customer carrier itself offers MPLS VPN services  

  

- How:  

- MP-BGP between customer RRs  

- Label stacking occurs:  

- Inner label → VPN route  

- Middle label → customer MPLS transport  

- Outer label → backbone transport  

  

👉 Total:  

- **3 labels used**  

  

---  

  

### Label Stack Example  

  

| Label | Purpose |  

|------|--------|  

| Outer | Backbone LSP |  

| Middle | Customer MPLS transport |  

| Inner | VPN label |  

  

---  

  

- Key Behavior:  

- End-to-end MPLS VPN across two providers  

  

- Risk:  

- Increased complexity in troubleshooting  

- Label stack management  

  

- Takeaway:  

CSC enables **multi-provider MPLS VPN services**  

  

---  

  

## 6. Key Design Differences Across Scenarios  

  

| Scenario | MPLS in Customer | LSP Formation | Labels |  

|--------|----------------|--------------|--------|  

| 1 | No | Backbone only | 1–2 |  

| 2 | Yes | Customer PE-PE | 2 |  

| 3 | Yes (VPN) | End-to-end | 3 |  

  

---  

  

## 7. Core Design Thinking  

  

👉 Backbone Provider:  

- Only transports traffic  

- Does NOT learn customer routes  

  

👉 Customer Carrier:  

- Controls routing  

- Maintains MPLS/VPN logic  

  

---  

  

## 8. Common Interview Points  

  

- CSC = MPLS over MPLS (provider over provider)  

- Backbone does NOT carry customer routes  

- Label stacking enables separation  

- LDP used only for transport LSP  

- 3-label stack in VPN scenario  

  

---  

  

## 9. Final Simple Understanding  

  

👉 Without CSC:  

- Each provider builds full backbone  

  

👉 With CSC:  

- One provider uses another provider’s backbone  

  

👉 Final Insight:  

CSC enables **scalable multi-provider MPLS architecture** by separating transport and routing using label stacking.

---
