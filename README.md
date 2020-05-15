# kubernetes-servicemesh

Do we need a service mesh ?  

few years ago i star to evaluate this feature fitting in an existing infrastructure  

There are many concept to consider and many mistake the people usually think  
Better to start with what is NOT a work for a service mesh   

- is not an apigw (even if could share some components)  
- is not the place to put firewall rules  


So what is ...   
well short answer   
is the missing link in the infrastructure observability  
is a way to handle in a structured way the application routing  
is an internal ratelimit / anti ddos / infrastructure layer (be careful)  
  
  
Anyway is this something that we can add in our infrastructure ?  

There no YES and NO , however we can evaluate the company and the maturity of microservices  
internal rate limit , is usually a feature that could safe the infrastructure during snowball effects   
however means that if the infrastructure is SYNC (no decoupling) have the rate limit can just   
stop the application to serve requests , and this will propagated to the others below.  
  
result: no answer  
threads safe  
   
Sometimes it's better to have a strategic business login using the a circuitbraker  that 
bring a huge complexity in configuration.  
  
The other point related to rate limit is .. who will maintain those values ?  
Should be part of deployment pipeline and directly correlated with the application scope  
In a 200+ micro services infrastructure this could be a huge problem:  
-project that lost ownership  
-projects not well maintained  
-new legacy projects  
  
So my idea about rate limit is to use it in a specific "strategic" applications 
and should not indiscriminately added to the whole infrastructure






