[ ]  ISIS PDU is directely carried in Ethernet Frame 
[ ]  ISIS is link-state protocla; uses LSP ( link starte protocal data units)
[ ]  ISIS  ad value 115 
[ ]  integrated ISIS deve;oped for TCP/IP
[ ]  Dijkstra Algoritham to fin SPF

# IS-IS Route Tagging & Policy (RFC 5130)  

  

## 1. Architect Notes (Clear + Practical)  

  

- What: Route tagging allows adding metadata (tags) to routes for policy control.  

- Why: Needed to control redistribution, filtering, and route leaking decisions.  

- How:  

- Tags added during redistribution or leaking  

- Used in route-maps for filtering and policy enforcement  

- Risk:  

- Misconfiguration can cause routing loops or blackholing  

- Example:  

- Tag external routes to prevent re-redistribution into same domain  

- Takeaway:  

Tags provide control to manage routing policies safely.  

  

👉 Interview Angle:  

Route tags help control routing policies and prevent loops during redistribution.  

  

---  

  

## 2. Interview Answer  

  

- Route tagging enables policy-based routing decisions using metadata.  

- It helps control route leaking and redistribution safely.  

  

- S: Routing loop due to multiple redistribution points.  

- T: Prevent route re-injection.  

- A: Applied route tags and filtering.  

- R: Eliminated routing loops.  

  

- Final Line:  

Route tagging is critical for safe and controlled route redistribution.


# IS-IS 3-Way Handshake  

  

## 1. Architect Notes (Clear + Practical)  

  

- What: 3-way handshake ensures bidirectional communication before forming adjacency.  

- Why: Prevents false adjacency in point-to-point links (one-way connectivity issue).  

- How:  

- Routers exchange 3-step hello confirmation  

- Adjacency formed only after bidirectional validation  

- Risk:  

- Without it, routers may think adjacency is FULL but traffic fails  

- Example:  

- Fiber unidirectional failure → adjacency UP but traffic broken  

- Takeaway:  

Ensures reliable adjacency by validating two-way communication.  

  

👉 Interview Angle:  

3-way handshake prevents false adjacency by verifying bidirectional connectivity.  

  

---  

  

## 2. Interview Answer  

  

- IS-IS 3-way handshake ensures both routers can send and receive traffic before adjacency is established.  

- Prevents one-way connectivity issues common in P2P links.  

  

- S: Adjacency formed but traffic failed due to unidirectional link.  

- T: Ensure reliable adjacency validation.  

- A: Enabled 3-way handshake.  

- R: Only valid adjacencies formed.  

  

- Final Line:  

3-way handshake improves reliability by eliminating false adjacencies.
