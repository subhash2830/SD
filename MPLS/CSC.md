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
Public key anytype
:: mango floor quantum state knee robot recycle above bird laugh pull kind


![[Pasted image 20260406110311.png]]
![[Pasted image 20260406110347.png]]

![[Pasted image 20260406110441.png]]