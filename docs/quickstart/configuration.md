# Configuration

## Introduction

TODO 

## Configuration file

```yaml
# config.yml
service:
  name: product
  aggregate: src.aggregates.Product
  injections:
    postgresql_pool: minos.common.PostgreSqlPool
    command_broker: minos.networks.CommandBroker
    command_reply_broker: minos.networks.CommandReplyBroker
    event_broker: minos.networks.EventBroker
    consumer: minos.networks.Consumer
    dynamic_handler_pool: minos.networks.DynamicHandlerPool
    repository: minos.common.PostgreSqlRepository
    saga_manager: minos.saga.SagaManager
    snapshot: minos.common.PostgreSqlSnapshot
    discovery: minos.networks.DiscoveryConnector
  services:
    - minos.networks.ConsumerService
    - minos.networks.CommandHandlerService
    - minos.networks.CommandReplyHandlerService
    - minos.networks.EventHandlerService
    - minos.networks.RestService
    - minos.networks.SnapshotService
    - minos.networks.ProducerService
rest:
  host: 0.0.0.0
  port: 8082
broker:
  host: localhost
  port: 9092
  queue:
    database: product_db
    user: minos
    password: min0s
    host: localhost
    port: 5432
    records: 1000
    retry: 2
repository:
  database: product_db
  user: minos
  password: min0s
  host: localhost
  port: 5432
snapshot:
  database: product_db
  user: minos
  password: min0s
  host: localhost
  port: 5432
events:
  service: src.ProductQueryService
queries:
  service: src.ProductQueryService
commands:
  service: src.ProductCommandService
saga:
  storage:
    path: ./product.lmdb
discovery:
  client: minos.networks.MinosDiscoveryClient
  host: localhost
  port: 5567
```