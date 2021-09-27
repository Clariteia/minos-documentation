# Exposing Operations: Command Service

## Introduction

After being learned how to setup a `minos` microservice and knowing the `Aggregate` basics and being able to start working on the domain model, the next step is to expose the implemented functionality outside, to be used both by another microservices and external clients! One of the main advantages of using `minos` is that it is able to expose endpoints over different interfaces transparently, so that the infrastructure details (that in many cases require a big effort) are handled by the framework and the business logic becomes into the leading actor of the microservice. 

The `minos` framework is based on the [CQRS](https://martinfowler.com/bliki/CQRS.html) pattern, which is characterised by the partition of the system into two main parts: the one focused into `Command` processing and the one focused into `Query` resolving. Following the event driven ideas, the `CommandService` is the one who processes requests that modify the internal state of the microservice, with changes on the `Aggregate`, that generate events, that can be handled by the `QueryService`, who updates its own query-oriented database. Then, the requests that will not change the state of the microservice (so that, the ones that are read-only) should be resolved by the `QueryService` who asks its own database.

The rest of the section is focused into the `CommandService` and the basic syntax to expose endpoints. Next, the :doc:`/quickstart/query` guide is fully focused into the `QueryService` and how to read and update the user-defined query database. 

## Setting up the service!

The setup process of the services is mostly equivalent both for the `CommandService` and the `QueryService` as they share the same interfaces configuration. Currently, the currently supported ones are the *HTTP REST* and the *Kafka*'s broker, but there are plans to increase the list with *gRPC*, *RabbitMQ*, etc.

The second important part in the configuration file is the `commands` and `queries`, in which the `CommandService` an `QueryService` classes are being setup.

```yaml
# config.yml

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
commands:
  service: src.commands.ExamCommandService
queries:
  service: src.queries.ExamQueryService
...
```

After being set up the services' configuration, the next part is to start writing the code itself! In this case, the command service is defined into the `src/commands.py` file:   

```python
from minos.cqrs import (
    CommandService,
)


class ExamCommandService(CommandService):
    pass
```

The most important part here is the `minos.cqrs.CommandService` import. In `minos`, the `cqrs` module is the one who implements the abstract service classes, both for commands and for queries.

Another important part is how to expose and set up the endpoints. In this case, `minos.networks` provides the `@enroute` decorator, that can be seen as a hierarchical decorator composed by the following parts:
* `@enroute.rest.command(url, method)`: Exposes a *Command* request over the *REST* interface.
* `@enroute.rest.query(url, method)`: Exposes a *Query* request over the *REST* interface.
* `@enroute.broker.command(topic)`:Exposes a *Command* request over the *Broker* interface.
* `@enroute.broker.query(topic)`:Exposes a *Query* request over the *Broker* interface.
* `@enroute.broker.event(topic)`: Exposes an *Event* request over the *Broker* interface.

After being learnt how to setup the services on the configuration file and take an overview over the `CommandService` base class and the `@enroute` decorator, it's time to introduce the expected function's prototype for each exposed endpoint:

```python
from minos.networks import (
    Request,
    Response,
)

class MyService(...):
    
    @enroute.rest.command(...)
    @enroute.broker.command(...)
    async def make_foo(self, request: Request) -> Optional[Response]:
        ...

```

The `minos.networks.Request` and `minos.networks.Response` are the classes with the responsibility to provide access to the received request itself and to build the given response. The main advantage of these two classes are that they are able to fully isolate the surrounding network interface that received the request, so that the code keeps much more simple. One thing to notice is that the endpoint does not need to necessarily return any value because there are some cases in which has no sense from the business logic perspective. Also, it's important to notice that the `minos.networks.ResponseException` provides a way to notify that there was some kind of situation that does not allow resolve the request successfully.

## Exposing `Exam` creation...

```python
from minos.common import (
    EntitySet,
)
from minos.networks import (
    enroute,
    Response,
    Request,
)
from .aggregates import (
    Exam,
)


class ExamCommandService(CommandService):

    @enroute.rest.command("/exams", "POST")
    @enroute.broker.command("CreateExam")
    async def create_exam(self, request: Request) -> Response:
        content = await request.content()

        exam = await Exam.create(content["subject"], content["title"], EntitySet())

        return Response(exam.uuid)

```

## Adding `Exam` questions!

TODO

## Deleting `Exam`...

TODO

## Why not to implement `get` commands?

TODO

## Summary

```python
"""src/commands.py"""

from minos.common import (
    EntitySet,
)
from minos.cqrs import (
    CommandService,
)
from minos.networks import (
    Request,
    Response,
    enroute,
)

from ..aggregates import (
    Exam,
)


class ExamCommandService(CommandService):
    
    @enroute.rest.command("/exams", "POST")
    @enroute.broker.command("CreateExam")
    async def create_exam(self, request: Request) -> Response:
        content = await request.content()

        exam = await Exam.create(content["subject"], content["title"], EntitySet())

        return Response(exam.uuid)
```