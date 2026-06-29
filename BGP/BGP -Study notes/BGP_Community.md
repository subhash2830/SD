!
The community attribute is an optional transitive attribute.

BGP communities are a means of tagging routes to ensure consistent filtering or route selection policy in incoming/outgoing routing updates , or with redistribution.

By default, communities are stripped in outgoing BGP updates. Sending them must be manually enabled.

!
! 
==Standard Community==   : 32 bit 

Is an extension used by BGP to pass additional information between BGP peers
used to aid policy administration and reduce the complexity of route management
```
set policy-options community R1_PREFERRED members 64511:1

set policy-options community R3_PREFERRED members 64511:3
```

Example :
no-advertise - Do not advertise routes to any BGP peer (local to a router).
no-export - Do not advertise routes to REAL eBGP peers (local to an AS).
local-as - Do not advertise routes to any eBGP peers (local to a confed-AS).
internet - Used to match all communities . (Not RFC standard, but Cisco implementation

==Extended Community==   :: 64 bit






Difference

  one can filter out all communities of a ==particular type==, or
   allow only ==certain values== for a particular type of community.  
   It also
   allows one to specify whether a particular community is ==transitive or
   non-transitive across an Autonomous System (AS) boundary==.
     
   Without structure, this can only be accomplished by explicitly enumerating

|                           |                                                                        |                                                                                     |
| ------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Feature                   | Standard BGP Community                                                 | Extended BGP Community                                                              |
| Size                      | 4 bytes (32 bits)                                                      | 8 bytes (64 bits)                                                                   |
| Structure                 | Simple, 1 field: <AS number>:<value>                                   | More complex, 2 fields: <Type> and <Global Value> <local Value >                    |
| Use Case                  | Primarily used for simple routing decisions (e.g., filtering, tagging) | Used for more complex routing policies (e.g., MPLS, VPNs, Traffic Engineering)      |
| Community Type            | One fixed type                                                         | Multiple types (e.g., Route Target, Traffic Engineering, OAM)                       |
| Fields in Community       | AS number : community value                                            |  2: 4:2  i,e Type : G Admin Value : Local value                                     |
| Transitivity              | Transitive by default                                                  | Can be either transitive or non-transitive based on the application                 |
| Common Examples           | - 65000:100 (to tag a route with an ASN)                               | - 0x00 65000:100 (Route Target for MPLS VPN)                                        |
| Flexibility               | Limited, simple filtering and policy control                           | Highly flexible, allows for more granular policy enforcement and filtering          |
| Scope of Use              | Primarily within a single AS or between directly connected peers       | Used across multiple ASes and in complex network setups (e.g., MPLS networks, VPNs) |
| Supported by BGP Versions | Supported by all BGP versions (i.e., BGP-4)                            | Requires extended BGP support (BGP-4 or later)                                      |
| Common Applications       | - Routing policy enforcement, traffic engineering (basic)              | - MPLS VPN Route Target, Traffic Engineering, Route Maps                            |

#### **1. Route Target (RT) Community**

- **Type/Sub-Type**: 0x02:0x02 (Transitive, Target).
- **Purpose**: Controls VPN route import/export (e.g., tags routes for VRF membership).
- **Format**: RT:AS:NN (e.g., RT:65001:1000) or RT:IP:NN.
- **Example**: Tag a route to import into VRF "CUST_A".
    
    bash
    
    ```
    route-map SET_RT permit 10
     set extcommunity rt 65001:1000  ! AS-based RT
    ```
    

#### **2. Route Origin (SOO) Community**

- **Type/Sub-Type**: 0x03:0x03 (Transitive, Origin).
- **Purpose**: Prevents route loops in multi-homed VPN sites by identifying the route's origin.
- **Format**: SOO:AS:NN (e.g., SOO:65001:2000) or SOO:IP:NN.
- **Example**: Mark a site to avoid re-advertising back to the same site.
    
    bash
    
    ```
    route-map SET_SOO permit 10
     set extcommunity soo 65001:2000
    ```
    

#### **3. Site-of-Origin (SOO) Community**

- **Type/Sub-Type**: 0x05:0x00 (Non-transitive, Site-ID).
- **Purpose**: Identifies the originating site in inter-AS VPNs to prevent loops.
- **Format**: SOO:AS:Site-ID (e.g., SOO:65001:500).
- **Example**: Use in inter-AS MPLS to block backdoor routes.
    
    bash
    
    ```
    neighbor <PE-IP> send-extended-community-ebgp
    route-map SET_SOO permit 10
     set extcommunity soo 65001:500
    ```
    

#### **4. Link Bandwidth Extended Community**

- **Type/Sub-Type**: 0x40:0x80 (Transitive, Numeric).
- **Purpose**: Signals link bandwidth for load-balancing in multi-path scenarios.
- **Format**: BW:AS:BW-value (e.g., BW:65001:1000000 for 1 Mbps).
- **Example**: Advertise bandwidth to influence ECMP in iBGP.
    
    bash
    
    ```
    route-map SET_BW permit 10
     set extcommunity bandwidth 1000000  ! 1 Mbps
    ```
    

#### **5. IPv4 Address-Specific Extended Community**

- **Type/Sub-Type**: 0x00:0x01 (Transitive, Target).
- **Purpose**: Encodes IPv4-specific policies, e.g., for flowspec or VPN targets with IP admin.
- **Format**: IP:IPv4-address:NN (e.g., IP:192.0.2.1:1000).
- **Example**: Tag routes with source IP for policy matching.
    
    bash
    
    ```
    route-map SET_IP_EXT permit 10
     set extcommunity target 192.0.2.1:1000  ! IP-based RT
    ```
    

#### **6. Cost Community**

- **Type/Sub-Type**: 0x08:0x80 (Transitive, Opaque).
- **Purpose**: Propagates IGP costs across AS boundaries for better path selection.
- **Format**: Cost:AS:Cost-value (e.g., Cost:65001:50).
- **Example**: Set cost to prefer lower-metric paths in inter-AS scenarios.
    
    bash
    
    ```
    route-map SET_COST permit 10
     set extcommunity cost 50
    ```
    

#### **7. Four-Octet AS-Specific Extended Community**

- **Type/Sub-Type**: 0x02:0x01 (Transitive, Target, for 4-byte ASNs).
- **Purpose**: Supports 4-byte ASNs in RT/SOO (RFC 5668).
- **Format**: RT:4-byte-AS:NN (e.g., RT:65535:1000).
- **Example**: Use for large ASNs in VPNs.
    
    bash
    
    ```
    route-map SET_4BYTE_RT permit 10
     set extcommunity rt 65535:1000  ! 4-byte AS
    ```
    

#### **8. No-Export or No-Advertise (as Extended)**

- **Type/Sub-Type**: 0x03:0x00 (Transitive, No-Export) or 0x03:0x02 (No-Advertise).
- **Purpose**: Restricts route advertisement (extended version for VPNs).
- **Format**: No-Export:AS:0 or fixed value.
- **Example**: Prevent VPN routes from leaking to global table.
    
    bash
    
    ```
    route-map NO_EXPORT permit 10
     set extcommunity no-export
    ```



![[BGP_Communities_Deep_Dive_Notes.docx]]

![[BGP_Communities_Deep_Dive_Notes.docx]]






















