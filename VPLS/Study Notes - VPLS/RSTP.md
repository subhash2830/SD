
port State 
          
**==Discarding==**  
No data sent/received 
BPDU received 
No mac learning  :: Replace Blocking in STP

**==learning==**        
No data sent/received 
BPDU sent / received  
mac learning    :: combine learning and listing  in STP

**==Forwarding==**    
data sent/received
BPDU sent / received  
mac learning 


port Role 

Root > received bpdu with lowest root path  cost

Designated > port who transmit best BPDU on segment 
AlterNet  > port having alternate path toward root  
BAckup  port on same sw connected to share segment but less desirable



Handshake  Mechanism 

> SW send its own Proposal and mention its port as Designated port 
> Remote SW have 2 option
   > Send Own proposal ( better BPDU )
   > Send Agreement 
   After that it perform syncronisation 

Process happened only on pt-to-pt interface which full duplex 
 because : Process happened bw 2 device only and half duplex comm may have hub indicating more than 2 devices 

Sync Process
1 Non edge port move into Discarding
2 Agreement sent on port on which Proposal was received 
3 Root port immediately start forwarding
4  On each Non edge port Proposal is now sent
5  Repeat same process if Superior BPDU sent
6 Expect Agreements on Same port 
7 After receiving Agreements move port into forwarding 