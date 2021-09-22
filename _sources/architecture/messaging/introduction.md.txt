## Introduction

The message is the minimum unit of communication between different actors or the same. An message is a change in state, 
or an update, like an item being placed in a shopping cart on an e-commerce website.
Messages can either carry the state (the item purchased, its price, and a delivery address) or messages can be identifiers 
(a notification that an order was shipped).

The general operation of the messages is as follows:

.. uml::

   @startuml
   box "Message Publishing - Microservice X" #FFFFF3
   participant Broker
   database DB_A
   participant Producer
   end box
   database Kafka
   box "Message Receiving - Microservice Y" #F3FFF6
   participant Consumer
   database DB_B
   participant Handler
   end box
   
   autonumber
   --> Broker: send
   
   Broker -> DB_A: Store Message
   Broker -> DB_A: Notify
   DB_A -> Producer: Notify
   Producer --> DB_A: Get Message
   Producer --> Kafka: Publish Message
   Kafka <-- Consumer: Receive Message
   Consumer -> DB_B: Notify
   Consumer -> DB_B: Store Message
   DB_B -> Handler: Notify
   Handler --> DB_B: Get Message
   Handler --> :Trigger action
   @enduml

Messages are stored in a PostgreSQL database before being sent to Kafka. This plays an important role as it allows
messages to be transactional and fault tolerant (network errors, Kafka crashes...).

Once the messages are successfully published in Kafka, they are removed from PostgreSQL and a notification
is sent (using PostgreSQL [LISTEN](https://www.postgresql.org/docs/9.1/sql-listen.html) / 
[NOTIFY](https://www.postgresql.org/docs/9.1/sql-notify.html) ) to consumers so that they can consume it.
(As you can see, this is done in a reactive way so that there does not have to be
a periodic background process that checks every x seconds for new messages for example. This reduces the overall
consumption of each microservice).

## Message Publishing

It is responsible for **storing** the event in the `PostgreSQL` database, **sending** it to `Kafka`, **deleting** it from `PotgreSQL` 
(if it has been successfully sent to Kafka) and **notifying** the consumer that the event can be consumed.

Let's take a step-by-step look at what a Producer does:

1. Stores the `Event` in database
   
   The events have several **attempts** to be stored in the database (parameterizable). As an example, we set the number of 
   retries to 3, if on the third attempt the event is not stored in the database because it is not a valid event, it 
   is marked to not be processed any more times. 
   
   @startuml
   participant Event
   database DB
   autonumber
   Event -> DB: Store Event
   @enduml


2. Publish `Event` to `Kafka`
   The events have several **attempts** to be sent to Kafka (parameterizable). As an example, if we set the number of 
   retries to 3, if on the third attempt the event is not sent to Kafka, it 
   is marked to not be processed any more times.
   
   @startuml
   participant Event
   database DB
   database Kafka

   autonumber
   Event -> DB: Store Event
   DB ->x Kafka: Publish Event
   DB ->x Kafka: Publish Event
   DB ->x Kafka: Publish Event
   DB -> DB: Marked for no further processing
   @enduml

## Message Receiving
TODO