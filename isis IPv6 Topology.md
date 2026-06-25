IS-IS can run IPv6 in two main ways:

- **Single topology**: IPv4 and IPv6 share the same IS-IS topology.
    
- **Multi-topology**: IPv4 and IPv6 can be advertised in separate logical topologies.
    

In single topology, IPv6 reachability is advertised using **IPv6 Reachability TLV 236**. The IPv6 prefixes are still carried in IS-IS, but the topology itself is the same as IPv4.

## Single topology explanation

In **single topology**, IS-IS treats IPv4 and IPv6 as part of one common SPF topology.

That means:

- One link-state database.
    
- One shortest path tree.
    
- IPv6 prefixes are carried separately in TLV 236.
    
- The protocol uses the same physical and logical topology for both address families.
    

So if the topology is stable for IPv4, IPv6 usually follows the same path.

## Important operational point

Your statement about adjacency needs one correction.

It is **not exactly true** that if one router does not have IPv6 enabled, _both IPv4 and IPv6 reachability are broken_ in every case. The key point is:

- In **single topology**, the **adjacency capability advertisement** must be compatible between neighbors.
    
- If a router does not advertise the required protocol support consistently for the chosen mode, the adjacency or reachability exchange can fail for that address family.
    

So the practical CCIE-style way to say it is:

**In single-topology IS-IS, all routers on the path must agree on the topology model and supported protocols, otherwise IPv6 reachability may not be exchanged correctly.**

That is a safer and more accurate interview answer.

## Single topology only IPv6

Yes, this is valid.

You can run:

- **IPv6 only**
    
- no IPv4 anywhere in the IS-IS domain
    

In that case, the protocol support information can list only IPv6, and the network can still operate in single-topology mode.

This is a useful special case because single-topology does **not** always mean dual-stack.

## Metrics behavior

This part is also important:

- **Single topology** can work with **narrow or wide metrics**, depending on platform and configuration.
    
- **Multi-topology** requires **wide metrics**.
    

So for CCIE-level remembering:

- **Single topology** = flexible.
    
- **Multi-topology** = wide metrics only.
    

## IOS-XE vs IOS-XR

Your platform note is directionally correct:

- **IOS-XE** commonly uses **single topology** by default.
    
- **IOS-XR** commonly uses **multi-topology** by default.
    

This is a good exam-level distinction to remember, especially when troubleshooting lab behavior.

## CCIE-style answer

If asked in an interview, say this:

**IS-IS supports IPv6 using either single topology or multi-topology. In single topology, IPv6 shares the IPv4 SPF tree and IPv6 prefixes are advertised using TLV 236. In multi-topology, IPv4 and IPv6 are carried in separate logical topologies and wide metrics are required. Single topology is commonly the default on IOS-XE, while multi-topology is commonly the default on IOS-XR.**[[cisco](https://www.cisco.com/c/en/us/support/docs/ip/integrated-intermediate-system-to-intermediate-system-is-is/5739-tlvs-5739.html)]

## Revision summary

- **Single topology**: one shared topology for IPv4 and IPv6.
    
- **Multi-topology**: separate logical topologies.
    
- **TLV 236**: carries IPv6 reachability.
    
- **Single topology** can be IPv4+IPv6 or IPv6-only.
    
- **Multi-topology requires wide metrics**.
    
- **Platform defaults differ** between IOS-XE and IOS-XR.
    


cd ./ine-spv4/lab_configs/XR1