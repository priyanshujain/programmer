## View: 

The view has few goals. First, 
- it is responsible for getting the inputs that come via HTTP request. 
- Secondly, it should sending this to the process that will process that data and collect the output of the operation.
- thirdly, it is responsible for processing the template and send the response back to client.


The view is not responsible for having the process itself, it is the controller. 

No business logic should live here. In fact, it should be the thinest layer on your application.
