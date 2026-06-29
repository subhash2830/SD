ospf by default don’t do any IP sumarisation
sumarisatipn can be done @ ABR for type 3 LSA  or @ ASBR for type 5 LSA 

Summarization of type 3 summary LSAs means we are creating a summary of all the interarea routes. This is why we call it interarea route summarization. 

Sumilaraly External route sumarrisation for type 5

A summary route will only be advertised if you have at least one subnet that falls within the summary range.  

A summary route will have the cost of the subnet with the lowest cost that falls within the summary range.  

Your ABR that creates the summary route will create a null0 interface to prevent loops.