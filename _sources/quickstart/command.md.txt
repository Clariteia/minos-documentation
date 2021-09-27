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
    ...
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

Now that the basic concepts to expose operations that can be used both by another microservices and also by external clients, the next step is to include some examples that help in the process to understand it better.   

## Exposing `Exam` creation...

A common operation on a microservice is the creation of `Aggregate` instances. Here is an example of `Exam`'s creation, defined on the `ExamCommandService`:

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

In this case, the command is exposed both over the *REST* and *Broker* interfaces. The way to extract the information from a `Request` instance is through its `content()` method, that in this case is assumed to be a `dict` instance containing the `subject` and `title` fields. The `return` value in this case is a `Response` instance, in which the content is composed of an `UUID` instance, but another composed options would have been possible. To know more about that, it's recommended to visit [TODO: link to Request/Response architecture section].  

## Adding `Exam` questions!

Similarly to `Exam`'s creation, another possible command could be the question adding: 

```python
from minos.common import (
    ValueObjectSet,
)
from .aggregates import (
    Question,
    Answer,
)


class ExamCommandService(CommandService):
    ...

    @enroute.rest.command("/exams/{uuid}/questions", "POST")
    @enroute.broker.command("AddExamQuestion")
    async def add_question(self, request: Request) -> Response:
        content = await request.content()
        
        exam = await Exam.get(content["exam"])
        answers = ValueObjectSet([Answer(raw["title"], raw["correct"]) for raw in content["answer"]])
        question = Question(content["title"], answers)
        exam.questions.add(question)
        await exam.save()

        return Response(question.uuid)

```

As it can be seeing, the main structure of the handling method is really similar to the one for `Exam`'s creation. An interesting difference here is that the generated `ExamUpdated` event will contain only the `Question` instance that is being created as it is stored inside an `EntitySet`. This is really important from the `QueryService` perspective, as it will be seeing in future guides. 

## Deleting `Exam`...

`Exam`'s deletion could be another common use case on a `exam` microservice. It can be implemented with a handling method like: 

```python
from minos.common import (
    MinosSnapshotAggregateNotFoundException,
    MinosSnapshotDeletedAggregateException,
)
from minos.networks import (
    ResponseException,
)


class ExamCommandService(CommandService):
    ...

    @enroute.rest.command("/exams/{uuid}", "DELETE")
    @enroute.broker.command("DeleteExam")
    async def delete_exam(self, request: Request) -> None:
        content = await request.content()
        uuid = content["uuid"]
            
        try:
            exam = await Exam.get(uuid)
        except MinosSnapshotAggregateNotFoundException:
            raise ResponseException(f"The exam does not exists: {uuid}")
        except MinosSnapshotDeletedAggregateException:
            raise ResponseException(f"The exam is already deleted: {uuid}")

        await exam.delete()
```

The interesting part of this handling method is that it does not return any `Response`, which is a valid implementation, but it raises a `ResponseException` to notify the caller about the reason why the deletion could not be performed.

## Deleting `Exam` by events...

The `Exam` aggregate contains a `ModelRef` to an external `Subject` aggregate, that it's assumed that is defined into another microservice. In classic monolithic systems that uses relational database schemas, a `ON DELETE` statement defines how instances with references will behave after a deletion of the referred instance. As `minos` follows the event driven ideas, it's also possible to perform operations after an event is published. So, a common use case of that could be to implement the `ON DELETE CASCADE` strategy between `Exam` and `Subject`: 

```python
from minos.common import (
    Condition,
    MinosSnapshotAggregateNotFoundException,
    MinosSnapshotDeletedAggregateException,
)
from minos.networks import (
    ResponseException,
)


class ExamCommandService(CommandService):
    ...

    @enroute.broker.event("SubjectDeleted")
    async def delete_exam_by_subject(self, request: Request) -> None:
        content = await request.content()
        
        exams = {exam async for exam in await Exam.find(Condition("subject", content["uuid"]))}
        
        for exam in exams:
            await exam.delete()
```

One of the main advantages of an event based implementation is that it's very easy to understand and implement. Also, it keeps the dependencies on the right direction and improves isolation of each component in the system. [TODO: Tell the saga based alternative.]  

## Why not to implement `get` operations as commands?

The queries (or get operations) are a special kind of operations characterised by the absence of side effects. As the `minos` framework follow the [CQRS](https://martinfowler.com/bliki/CQRS.html) pattern, it is highly recommended not to implement `get` operations to be resolved by the `CommandService` because its responsibility must be to define operations that change the internal `Aggregate` instances in some sense. 

By cons, the `QueryService` must not use the `Aggregate` class directly, so it cannot change its state, but it's able to know that state through the domain events given as `AggregateDiff` instances. In this way, the `QueryService` is able to have a custom internal representation of the `Aggregate` oriented to the queries that it implements, being easier to have big performance advantages. A more detailed explanation can be seen in :doc:`/quickstart/query`.

## Summary

After being described step by step the main features of the `minos.cqrs.CommandService` class and related classes like `minos.networks.Request`, `minos.networks.Response` and `minos.networks.enroute`, here is a full snapshot of the resulting `src/commands.py` file:

```python
"""src/commands.py"""

from minos.common import (
    Condition,
    EntitySet,
    ValueObjectSet,
    MinosSnapshotAggregateNotFoundException,
    MinosSnapshotDeletedAggregateException,
)
from minos.cqrs import (
    CommandService,
)
from minos.networks import (
    Request,
    Response,
    ResponseException,
    enroute,
)

from .aggregates import (
    Exam,
    Question,
    Answer,
)


class ExamCommandService(CommandService):
    
    @enroute.rest.command("/exams", "POST")
    @enroute.broker.command("CreateExam")
    async def create_exam(self, request: Request) -> Response:
        content = await request.content()

        exam = await Exam.create(content["name"], content["duration"], content["subject"], EntitySet())

        return Response(exam.uuid)

    @enroute.rest.command("/exams/{uuid}/questions", "POST")
    @enroute.broker.command("AddExamQuestion")
    async def add_question(self, request: Request) -> Response:
        content = await request.content()
        
        exam = await Exam.get(content["exam"])
        answers = ValueObjectSet([Answer(raw["text"], raw["correct"]) for raw in content["answer"]])
        question = Question(content["title"], answers)
        exam.questions.add(question)
        await exam.save()

        return Response(question.uuid)

    @enroute.rest.command("/exams", "DELETE")
    @enroute.broker.command("DeleteExam")
    async def delete_exam(self, request: Request) -> None:
        content = await request.content()
        uuid = content["uuid"]
            
        try:
            exam = await Exam.get(uuid)
        except MinosSnapshotAggregateNotFoundException:
            raise ResponseException(f"The exam does not exists: {uuid}")
        except MinosSnapshotDeletedAggregateException:
            raise ResponseException(f"The exam is already deleted: {uuid}")

        await exam.delete()

    @enroute.broker.event("SubjectDeleted")
    async def delete_exam_by_subject(self, request: Request) -> None:
        content = await request.content()
        
        exams = {exam async for exam in await Exam.find(Condition("subject", content["uuid"]))}
        
        for exam in exams:
            await exam.delete()
```