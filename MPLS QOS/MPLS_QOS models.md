!
short pipe :
			> On ingress mpls exp bit may or may not be derived(changed by SP) from IP header TOS value
               On egress forwarding treatment done based on mpls exp bits.

pipe  :
			> same as pipe model
             but on egress forwarding treatment is done using IP header TOS bit


uniform :
			> On ingress IP header TOS bits must be copied to imposed mpls exp bits
                On egress mpls bits must be copied to IP header TOS bits