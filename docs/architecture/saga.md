# SAGA

## Context
Minos framework apply **Database per Service** Pattern. Each service has its own database. Some business transactions,
however, span multiple service so we need a mechanism to implement transactions that span services. 
For example, let’s imagine that we have an e-commerce store where customers have a credit limit. 
The application must ensure that a new order will not exceed the customer’s credit limit. Since `Orders` and `Customers` 
are in different databases owned by different services the application cannot simply use a local ACID transaction.

## Problem
How do we implement transactions that affect several services?

In our case, how do we make Order and Customer communicate?

## Solution
Implement each business transaction that spans multiple services is a saga. A saga is a sequence of local transactions.
Each local transaction updates the database and publishes a message or event to trigger the next local transaction in 
the saga. If a local transaction fails because it violates a business rule then the saga executes a series of 
compensating transactions that undo the changes that were made by the preceding local transactions.

## Example
In the following example we are going to see how the saga is for the case of adding a `Product` to the `Cart`, we want 
this product to be reserved for us.

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
   
## Parameters
TODO

## SAGA definition

## Steps
TODO
###  `step()`
TODO
### `invoke_participant()`
TODO
### `with_compensation()`
TODO
### `on_reply()`
TODO
### `commit()`
TODO

## Query building
TODO
