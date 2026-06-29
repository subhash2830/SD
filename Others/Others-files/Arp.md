

---

## 1. Architect Notes 

## What

- ARP is the **translator** between IP address and MAC address inside a Layer 2 broadcast domain.
    
- It lets an IP‑speaking host find the correct MAC so Ethernet can actually forward the frame on the wire.
    

## Why (real problem it solves)

- Routers and hosts forward based on IP, but Ethernet switches deliver based on MAC; without a mapping, packets cannot leave the NIC.
    
- In any Ethernet network, every first‑hop (PC → gateway, server → ToR, etc.) needs an IP→MAC mapping; ARP automates this process.
    

## How

- Host first checks local ARP cache; if entry exists and not expired, it reuses it.
    
- If not, it sends a broadcast “Who has IP X?”; the owner responds unicast with “IP X is at MAC Y”; both then cache that mapping with a timer.
    
- ARP only runs for **locally connected** IPs; for remote prefixes, the host ARPs only for the default gateway’s IP, never for the remote server itself.
    

## Design context

- ARP is **broadcast‑based**, so large flat L2 domains multiply ARP noise and can stress CPU on switches/hosts in dense DCs or WLANs.
    
- Designers contain ARP scope with VLANs/VRFs, or offload/suppress ARP using SDN controllers, ARP proxies, or EVPN control‑plane learning.
    
- IPv6 replaces ARP with NDP (multicast, richer options, and security extensions like SEND), but design problems are similar: neighbor discovery load and security.
    

## Variants (where they show up)

- Gratuitous ARP: host/virtual IP announces “IP X is at MAC Y” without being asked; used in HSRP/VRRP, clustering, and vMotion/VM migration to update switches fast.
    
- Proxy ARP: router answers ARP on behalf of a remote host, making a routed segment look like it’s directly attached – useful in legacy flat or VPN/MPLS scenarios, but can hide design issues.
    
- Reverse ARP/InARP: historical WAN mechanisms (Frame Relay/ATM) to map DLCI/VC to IP – now mostly replaced by DHCP and modern control planes.
    

## Security and failure thinking

- ARP is **unauthenticated and stateless**, so anyone can reply “IP X is at my MAC” → classic spoof/poison → MITM or blackhole.
    
- Attackers can flood ARP to overflow CAM tables, forcing switches to behave like hubs and increasing attack surface.
    
- Defenses: Dynamic ARP Inspection with DHCP snooping bindings, static ARP for critical assets, segmentation to keep blast radius small, and encrypting higher‑layer traffic (HTTPS/SSH) so even MITM sees only ciphertext.
    

## Real‑world design applications

- Every host ARPs for its default gateway before sending off‑subnet traffic; first‑hop stability and ARP convergence directly impact user experience.
    
- FHRP (HSRP/VRRP) failover relies on Gratuitous ARP so endpoints start using the new active router’s MAC immediately, without reconfig.
    
- In virtualized and cloud DCs, VM move events trigger Gratuitous ARP/NDP so ToR and fabric update where to send traffic; SDN fabrics often intercept/reply to ARP to avoid broadcast storms.
    
- Security/monitoring tools track ARP patterns to detect spoofing, duplicated IPs, and anomalous behavior.
    

## Example (short scenario)

- You design a campus with large Wi‑Fi user density and voice endpoints.
    
- You segment users into multiple VLANs per floor, keep ARP domains small, enable DHCP snooping + DAI on access switches, and let the SDN controller proxy ARP for east‑west inside the DC, reducing broadcast while still keeping quick neighbor resolution.
    

## Takeaway (design‑level)

- ARP is simple but **foundational control‑plane glue**; as domains grow you must explicitly design for ARP scale and security, not assume “it just works.”
    

---

## 2. Interview Angle (Crisp + Ready-to-Speak)

“In simple terms, ARP maps an IP address to a MAC address inside a subnet so Ethernet can actually forward frames. IP is the logical locator, MAC is the physical locator; ARP is the translator in between.

In design, I care about three things: scope, scale, and security. ARP is broadcast‑based and unauthenticated, so in large L2 domains it can create noise and is easy to spoof. I typically contain ARP with VLANs/VRFs or ARP proxies, and protect it with Dynamic ARP Inspection and DHCP snooping.

Gratuitous ARP is critical for fast failover and VM mobility, and IPv6’s NDP is essentially the next evolution of the same idea using multicast with better extensibility. As an architect, I treat ARP as a key control‑plane function: if it’s noisy or compromised, the network is logically down even if all links are up.”

---

# ARP – Address Resolution Protocol

## Core Role
- Translator between IP and MAC.
- Connects Layer 3 logic with Layer 2 forwarding.
- Needed for Ethernet communication.

## Why It Exists
- IP is logical, MAC is physical.
- Ethernet needs a MAC to deliver frames.
- ARP bridges that gap.

## How It Works
- Cache first.
  - Check local ARP table.
- Broadcast request.
  - “Who has IP X?”
- Unicast reply.
  - Owner replies with MAC.
- Cache update.
  - Store IP-to-MAC mapping.

## Design Context
- Local subnet only.
- Does not cross routers.
- Large Layer 2 domains create ARP noise.
- VLANs and VRFs help reduce scope.

## Variants
- Gratuitous ARP.
  - Announces own IP-to-MAC.
  - Used in failover and VM migration.
- Proxy ARP.
  - Router replies for another host.
  - Useful in legacy networks.
- Reverse ARP.
  - Old method.
  - Replaced by DHCP.
- InARP.
  - Used in Frame Relay / ATM.

## Security Risks
- ARP spoofing.
  - Fake mapping leads to MITM.
- ARP flooding.
  - Can stress switch CAM tables.
- Defenses.
  - DAI.
  - DHCP snooping.
  - Static ARP for critical systems.

## Real-World Use
- Hosts ARP for default gateway.
- HSRP and VRRP use Gratuitous ARP.
- VM migration updates ARP info.
- Data centers may suppress ARP at scale.

## Key Takeaways
- ARP links IP to MAC.
- Works only inside local subnet.
- Simple but insecure.
- Scale needs segmentation and controls.