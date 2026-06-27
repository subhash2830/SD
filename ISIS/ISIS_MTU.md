---
uid:
title:
alias:
topic:
date:
tags:
status:
priority:

---
 - [ ]  MTU for ISIS on ethernet interface is 1497
-        Cisco ISIS uses 802.3 LLC frame format
         so actual header length is increased by 3 bytes and actual maximum payload is 1497

- [ ] isis MTU need to check while bring up neighbor ship  else stuck in init 

How ISIS detect MTU mismatch

 All hellos are padded to full MTU size (Padding TLV 8)
 and even if hello padding is disabled cisco router still sends first 5 hellos fully padded
# IS-IS MTU Handling & Adjacency Issues – Architect Notes + Interview  

  

## 1. IS-IS MTU on Ethernet  

  

- What: IS-IS uses a reduced effective MTU (~1497 bytes) on Ethernet due to protocol overhead.  

- Why: IS-IS runs over **802.3 LLC (Layer-2)**, which adds extra header bytes (approx. 3 bytes compared to IP/Ethernet).  

- How:  

- Ethernet MTU = 1500  

- LLC header overhead → usable payload ≈ 1497 bytes  

- Risk:  

- MTU mismatch leads to LSP drops even if Hello works  

- Example:  

- Interfaces configured with 1500 vs 1492 → adjacency stuck or unstable  

- Takeaway:  

IS-IS MTU is not same as IP MTU; must be validated explicitly for stability  

  

👉 Interview Angle:  

IS-IS runs directly over Layer-2 (LLC), reducing usable MTU and requiring careful MTU alignment for stable operation.  

  

---  

  

## 2. Why MTU Matters for IS-IS  

  

- What: MTU determines maximum size of IS-IS PDUs (especially LSPs)  

- Why:  

- LSPs carry topology info → may exceed smaller MTU  

- How:  

- Hellos may pass, but LSPs fail if MTU mismatch exists  

- Risk:  

- Stuck in INIT state  

- Partial adjacency  

- Routing inconsistency (LSP drops)  

- Example:  

- Neighbor forms partially, but LSP exchange fails → no routes installed  

- Takeaway:  

MTU mismatch is one of the most common hidden IS-IS issues  

  

---  

  

## 3. MTU Detection Mechanism (Critical)  

  

- What: IS-IS uses **Hello padding** to detect MTU mismatch  

- Why:  

- Ensures both routers can handle full-size packets  

- How:  

- Hello packets are padded to full interface MTU using:  

- **TLV 8 (Padding)**  

- If neighbor cannot receive full-size Hello → detected mismatch  

- Special Cisco Behavior:  

- Even if padding disabled:  

- First 5 Hellos are still sent at full MTU  

  

- Risk:  

- Disabling padding may delay detection but not eliminate MTU issues  

- Example:  

- Adjacency forms initially (due to partial packets), fails later during LSP exchange  

- Takeaway:  

Hello padding is proactive MTU validation mechanism  

  

👉 Interview Angle:  

IS-IS uses padded Hellos (TLV 8) to verify MTU compatibility before full LSP exchange.  

  

---  

  

## 4. Adjacency Issue (INIT State Problem)  

  

- What:  

- Adjacency stuck in **INIT state**  

- Why:  

- MTU mismatch prevents proper bidirectional communication  

- How:  

- Router receives Hello but cannot fully process LSPs  

- Risk:  

- No routing exchange  

- Silent failure (hard to detect initially)  

- Example:  

- Router sees neighbor but does not progress to FULL state  

- Takeaway:  

INIT state is a classic symptom of MTU mismatch in IS-IS  

  

---  

  

## 5. Design Thinking (Key Insight)  

  

- IS-IS:  

- Runs on Layer-2 → no IP fragmentation handling  

- Therefore:  

- MTU must be consistent end-to-end  

  

👉 Key Rule:  

- **All IS-IS interfaces in same domain must support same MTU**  

  

---  

  

## 6. Real-World Scenario (STAR Applied)  

  

- S:  

IS-IS adjacency stuck in INIT state across WAN link after migration.  

  

- T:  

Identify and resolve adjacency failure.  

  

- A:  

Checked MTU settings and found mismatch; validated using padded Hello behavior; aligned interface MTU.  

  

- R:  

Adjacency moved to FULL state and routing restored successfully.  

  

Short Explanation:  

Even though Hello packets were exchanged, MTU mismatch blocked proper adjacency formation and LSP exchange.  

  

- Final Line:  

In IS-IS, MTU consistency is critical because the protocol does not rely on IP fragmentation, making early detection via padded Hellos essential.  

  

---  

  

## 7. Design Guidelines (Architect View)  

  

- Always verify MTU across all IS-IS links  

- Do not rely only on Hello success → validate LSP exchange  

- Keep padding enabled (default behavior)  

- Be cautious in:  

- MPLS networks  

- Tunnel interfaces  

- Use consistent MTU in:  

- Data center fabrics  

- WAN links  

  

---  

  

## 8. Quick Recall (Interview)  

  

- IS-IS MTU ≈ 1497 (Ethernet + LLC overhead)  

- Runs over Layer-2 (no IP header)  

- Uses TLV 8 (padding) for MTU detection  

- First 5 Hellos padded even if disabled  

- MTU mismatch → INIT state / LSP failure  

  

---  

  

## 9. Final Architect Insight  

  

MTU handling in IS-IS is critical because the protocol operates directly over Layer-2 without relying on IP fragmentation.  

Designs must ensure consistent MTU across the network, as mismatches can silently break adjacency or LSP exchange, making MTU validation a mandatory step in IS-IS deployments.
