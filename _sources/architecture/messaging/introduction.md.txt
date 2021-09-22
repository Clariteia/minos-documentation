## Introduction

The message is the minimum unit of communication between different actors or the same. An message is a change in state, 
or an update, like an item being placed in a shopping cart on an e-commerce website.
Messages can either carry the state (the item purchased, its price, and a delivery address) or messages can be identifiers 
(a notification that an order was shipped).

The general operation of the messages (`Events`, `Commands` and `Command Replies`) is as follows:

.. uml::
   :align: center

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

Before the message is finally published in Kafka, it goes through a series of operations involving the following actors,
which are necessary to know before going deeper into the flow:
+ **Broker**: In charge of storing the received message in the database and launching a 
  [NOTIFY](https://www.postgresql.org/docs/9.1/sql-notify.html).
+ **Database**: It is the database where the message is stored to preserve transactionality and to be tolerant to system 
  failures.
+ **Producer**: It is in charge of [LISTENING](https://www.postgresql.org/docs/9.1/sql-listen.html) for new messages in the
  database and publishing them in Kafka.

Now that we know the components involved, let's look at the complete flow:

.. uml::
   :align: center

   @startuml
   box "Message Publishing - Microservice X" #FFFFF3
   participant Broker
   database DB_A
   participant Producer
   end box
   database Kafka
   
   autonumber
   --> Broker: send
   
   Broker -> DB_A: Store Message
   Broker -> DB_A: Notify
   DB_A -> Producer: Notify
   Producer --> DB_A: Get Message
   Producer --> Kafka: Publish Message
   @enduml


1. The `Broker` receive the message and add it to the database 
   
   The received message is stored in the `PostgreSQL` database of the corresponding microservice.
   The table where the message is stored has the following structure:
   ```SQL
   CREATE TABLE IF NOT EXISTS producer_queue (
   id BIGSERIAL NOT NULL PRIMARY KEY,
   topic VARCHAR(255) NOT NULL,
   data BYTEA NOT NULL,
   action VARCHAR(255) NOT NULL,
   retry INTEGER NOT NULL DEFAULT 0,
   created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
   updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW())
   ```

2. The `Broker` trigger notify action to the database
   
   The Broker triggers a notification to let the Producer know that new messages are available.
   ```SQL
   NOTIFY producer_queue
   ```
   
3. The `Producer` listens to new messages
   
   The Producer is actively listening for new messages using the following SQL:

   ```SQL
   LISTEN producer_queue
   ```
   
4. The `Producer` is notified of new messages

   The producer reads from the database table (mentioned in step 1) obtaining the new messages.

5. The `Producer` publishes the messages in `Kafka`

   The message have several **attempts** to be stored in the database (parameterizable). As an example, we set the
   number of retries to 3, if on the third attempt the message is not stored in the database, it is marked to not be
   processed any more times. 
   
.. uml::
   :align: center

   @startuml
   database DB_A
   participant Producer
   database Kafka
   
   autonumber
   Producer ->x Kafka: Publish Message
   Producer ->x Kafka: Publish Message
   Producer ->x Kafka: Publish Message
   Producer -> DB_A: Message marked for no further processing
   @enduml


6. If the message has been published, it is deleted

   The `producer` takes care of deleting the message from the local database once it has been successfully published in
   Kafka.

## Message Receiving
TODO