# Setup

## Introduction

Before starting to think in multiple microservices, distributed environments and so on, it's easier and better to start
by knowing what components are needed to build a microservice individually. For sake of simplicity, each microservice
can be seen as one small monolithic system that provides as limited or specialized functionality. Then, each
microservice will probably need to interact with a storage system, contain some kind of business logic and expose that
functionality over an external API, to be used by external clients and another microservices.

## Dependencies

To be able to provide all that environment functionalities, `minos` relies on external dedicated libraries. For example,
to read or store data in postgres database `aiopg` is used, to publish or subscribe to kafka topics `aiokafka` is used
and to expose a REST interface `aiohttp` is used. In any case, the `minos` philosophy is that the surrounding
technologies must not be frozen to these and each of project must have its own ones so the list of supported databases
and brokers will be increased in future `minos` releases.

Another important thing to know is that `minos` relies on latest `Python` functionalities so the minimal required
version is `3.9` or greater. One of the reasons are the use of *async* code together with *type hinted* code.

Then, for this quickstart the following dependencies are needed:

* **Python** >= 3.9
* **PostgreSQL** >= 9.3
* **Kafka** >= 2.8

In order to ease the quickstart process, here is a simple `docker-compose.yml` that provides the environment
dependencies:

```yaml
# docker-compose.yml

version: "3.9"
services:
  postgres:
    image: postgres:latest
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: minos
      POSTGRES_PASSWORD: min0s
      POSTGRES_DB: exam_db
  zookeeper:
    image: wurstmeister/zookeeper:latest
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka:latest
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_DELETE_TOPIC_ENABLE: "true"
```

## Modularity

Another important detail about how the `minos` framework is how it is packaged. Instead of providing all the framework
as a single and big component, it relies
on `Python` [namespaces](https://packaging.python.org/guides/packaging-namespace-packages/) to provide a modular but
organized way to install only the exact part of functionality that will be used. The main packages are the following
ones:

* [minos-microservice-common](https://pypi.org/project/minos-microservice-common/): Contains the `minos.common` module.
* [minos-microservice-networks](https://pypi.org/project/minos-microservice-networks/):  Contains the `minos.networks`
  module.
* [minos-microservice-saga](https://pypi.org/project/minos-microservice-saga/): Contains the `minos.saga` module.
* [minos-microservice-cqrs](https://pypi.org/project/minos-microservice-cqrs/): Contains the `minos.cqrs` module.

## Package Managers

Before starting a `Python` project, a package manager is needed to install and manage external dependencies. As it's
expected, `minos` is compatible with any of them, but the recommended one is `Poetry` as it provides simple but powerful
functionalities.

### Poetry

To install poetry, it's recommended to follow its own guide, that can be found [here](https://python-poetry.org/docs/).
After installing `poetry`, everything is ready to build the first microservice!

The first step is to create the project (the project name can be whatever you want, but a `minos` microservice
convention is to name it as `src`):

```bash
poetry new exam-microservice-example --name src
```

The project structure will be similar to:

```bash
exam-microservice-example
├── README.rst
├── pyproject.toml
├── src
│   └── __init__.py
└── tests
    ├── __init__.py
    └── test_src.py

2 directories, 5 files
```

To install the `minos` dependencies, simply exec the following command on the project's root directory:

```bash
poetry add \
  minos-microservice-common \
  minos-microservice-networks \
  minos-microservice-saga \
  minos-microservice-cqrs
```

### PipEnv
[TODO: Include pipenv guide.] 

### Pip
[TODO: Include pip guide.]

After that, every `minos` dependencies are ready to use. Let's move to the configuration step!

## Configuration

The configuration step is the last part of the microservice's setup, which is the process of defining the microservice's name, the classes to be responsible for specific framework's tasks and the location of credentials to interact with external resources like databases and brokers.

As many parts of the configuration requires digging deeper into each of the components of the framework, here is provided a superficial description of the main ones.
* `service.name`: The name of the microservice.
* `service.aggregate`: The qualified `Python` path to the root aggregate.
* `service.injections`: A mapping of instances to be injected around the framework to provide framework's functionality.
* `service.services`: A set of background services whose main purpose that provide some framework's functionality. 
* `rest`: Configuration of the rest's interface.
* `broker`: Configuration of the broker's interface.
* `repository`: Configuration of the repository database.
* `snapshot`: Configuration of the snapshot database.
* `events`: Configuration of the events service.
* `queries`: Configuration of the queries service.
* `commands`: Configuration of the commands service.
* `saga`: Configuration of the saga manager.
* `discovery`: Configuration of the discovery service.

Here is a template for a `config.yml` file, which will be used by the `exam` microservice. 

```yaml
# config.yml

service:
  name: exam
  aggregate: src.aggregates.Exam
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
    database: exam_db
    user: minos
    password: min0s
    host: localhost
    port: 5432
    records: 1000
    retry: 2
repository:
  database: exam_db
  user: minos
  password: min0s
  host: localhost
  port: 5432
snapshot:
  database: exam_db
  user: minos
  password: min0s
  host: localhost
  port: 5432
events:
  service: src.ExamQueryService
queries:
  service: src.ExamQueryService
commands:
  service: src.ExamCommandService
saga:
  storage:
    path: ./exam.lmdb
discovery:
  client: minos.networks.MinosDiscoveryClient
  host: localhost
  port: 5567
```

## API Gateway and Discovery

[TODO: Describe integration with discovery.]
