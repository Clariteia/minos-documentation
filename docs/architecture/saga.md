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

