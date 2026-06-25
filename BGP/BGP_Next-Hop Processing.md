!


When a packet is passed between ==iBGP peers==, ==NO next-hop processing is done==, unless confederations are used.

When a packet is passed between ==eBGP peers, the next-hop field is modified== to the IP address of the sending eBGP router's interface


- [ ] If the receiving BGP router is in the same subnet as the current next-hop address,

the next-hop field remains unchanged to optimize packet forwarding (typically seen on

multi-access networks).

Next-hop processing could be changed in one of two ways :

"neighbor next-hop-self" command.

Or with a route-map by setting the " ip next-hop".