!

MPLS QOS Rule1  <ip2label >
			> First 3 bits of Tos field of IP header are copied into all imposed labels at ingress LSR


MPLS QOS Rule2   <label2label ; swap & push >
			> Exp bits of top incoming label are copied to swaped or pushed labels

MPLS QOS Rule3  <label2label ; pop>
			> Exp bit of incoming top label are not copied to newly exposed label when incoming label is popped

MPLS QOS Rule4  <label2ip) >
			> Exp bits of incoming top label are not copied to tos bits of IP header when label stack is removed

MPLS QOS Rule5   <config change to exp bits>
			> when we change exp bits value through configuration value of exp bits of top or swap or push label is changed 
               but exp bits id labels underneeth and ip header tos bits are not changed


