#R1, R3, R5
ethernet mac limit action flooding disable
!
bridge-domain 100
 mac aging-time 150
 mac limit maximum addresses 100