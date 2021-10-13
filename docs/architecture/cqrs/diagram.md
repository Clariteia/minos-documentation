# Diagram

.. uml::
   :align: center

   @startuml
   actor User
   box "Microservice Cart" #FFFFF3
   participant "add_to_cart()"
   participant SAGA
   end box
   
   box "Microservice Product" #F3FFF6
   participant Product
   end box
   
   autonumber
   User --> "add_to_cart()": Add item to cart
   "add_to_cart()" -> SAGA: Execute
   SAGA -> Product: Try to reserve the product
   Product --> SAGA: Product reserved
   SAGA --> SAGA: Add product to the cart
   note left
   In the commit method,
   add item to cart
   end note
   SAGA --> "add_to_cart()": Success
   "add_to_cart()" --> User: Response
   @enduml