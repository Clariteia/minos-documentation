# Events

## Introduction

An event is a change in state, or an update, like an item being placed in a shopping cart on an e-commerce website.
Events can either carry the state (the item purchased, its price, and a delivery address) or events can be identifiers 
(a notification that an order was shipped).

Key components: event `producers` and event `consumers`. 
A producer publishes an event to the Kafka and the consumer reads them. 
Producer services and consumer services are decoupled, which allows them to be scaled, updated, 
and deployed independently.

Each component and its operation is detailed below.

## Overview

Events are stored in a PostgreSQL database before being sent to Kafka. This plays an important role as it allows
messages to be transactional and fault tolerant (network errors, Kafka crashes...).

Once the messages are successfully published in Kafka, they are removed from PostgreSQL and a notification
is sent (using PostgreSQL [LISTEN](https://www.postgresql.org/docs/9.1/sql-listen.html) / 
[NOTIFY](https://www.postgresql.org/docs/9.1/sql-notify.html) ) to consumers so that they can consume it.
(As you can see, this is done in a reactive way so that there does not have to be
a periodic background process that checks every x seconds for new messages for example. This reduces the overall
consumption of each microservice).

@startuml
actor Producer
database DB
database Kafka
actor Consumer

autonumber
Producer -> DB: Store Event
DB -> Kafka: Publish Event
DB -> Consumer: Notify
Consumer --> DB: Get Event
@enduml

## Producer (Broker)

It is responsible for **storing** the event in the `PostgreSQL` database, **sending** it to `Kafka`, **deleting** it from `PotgreSQL` 
(if it has been successfully sent to Kafka) and **notifying** the consumer that the event can be consumed.

Let's take a step-by-step look at what a Producer does:

1. Stores the `Event` in database\
   The events have several **attempts** to be stored in the database (parameterizable). As an example, we set the number of 
   retries to 3, if on the third attempt the event is not stored in the database because it is not a valid event, it 
   is marked to not be processed any more times.\
    @startuml
    participant Event
    database DB
    
    autonumber
    Event ->x DB: Store Event
   Event ->x DB: Store Event
   Event ->x DB: Store Event
   Event -> Message: Marked for no further processing
    @enduml

2. Publish `Event` to `Kafka`

@startuml
participant Event
database DB
database Kafka

autonumber
Event -> DB: Store Event
DB -> Kafka: Publish Event
DB ->x Consumer: Notify
@enduml

## Consumer
TODO