!

==< WLLAOMNI > ==
1 - prefer the ==highest weight== ( Cisco specific )

2- Prefer the ==highest local preference== (local to AS).

3- Prefer the routes that the router originated locally.

4- Prefer the ==shortest AS== paths (only the length is compared).

5- Prefer the ==lowest origin code== (IGP-i before EGP-e before Incomplete-?).

6- Prefer the ==lowest MED.==

7- Prefer external (eBGP) paths over internal (iBGP) paths  ==eBGP over iBGP==

			For eBGP paths , prefer the oldest (most stable) path.
			For iBGP paths , prefer the path through the closest IGP neighbor (lowest IGP 
			metric).

			
8- If RR  are configured:

	1>  multiple **iBGP routes** non-reflected routes are preferred above reflected routes.
	2> reflected routes with a shorter cluster -list are preferred above routes with a 
	longer cluster -list.



9- Prefer the path from the router with the l==owest BGP router - ID.==

10- Prefer the path from the ==lowest neighbor interface address.==