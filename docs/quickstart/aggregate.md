# Modeling the Domain: Aggregates

## Introduction

In opposite to monolithic systems who use relational database model to represent and store the business logic, microservice-based systems requires applying slightly different techniques. One of the main reasons to start thinking different is based on the splitting of a single but big information schema into multiple smaller ones, as each microservice has its own dedicated database. 

The `minos` proposal is based on the *Domain Drive Design (DDD)* ideas, supported by *Entities*, *Value-Objects* and an *Aggregate* (or *Root Entity*) that represents the main concept of the microservice. These set of concepts allows us to model information at microservice level, but at some cases it's needed to relate information from one microservice with information from another one. The way to do that in `minos` is over *Aggregate References*. To understand these concepts in more detail, the :doc:`/architecture/data_model` section provides a more detailed explanation about them.

## Defining the `Exam` aggregate...

As it is advanced at the beginning of the :doc:`/quickstart/_toc` guide, the study case will be to define an `exam` microservice. This one will be able to store all the needed information related with a common *exam* composed of questions with selectable answers. For sake of simplicity, all of them must be multiple answer questions, but it's a good practising exercise to continue working on these case and add another functionalities.

In this case, the `Exam` class will be the root-entity or `Aggregate` of the microservice, so that most of the operations sent to it will be related with the `Exam`. In `minos`, the way to do that is to inherit from the ``minos.common.Aggregate`` class, that is similar to `Python`'s [dataclasses](https://docs.python.org/3/library/dataclasses.html) in the sense that is able to build the class fields based on the typing. The currently supported types are all the simple `Python`'s types (`int`, `float`, `str`, ...) but also another advanced types are supported, like `list`, `dict`, `datetime`, etc. See the full documentation to obtain a detailed description.

With this simple functionality, it's already possible to start building the `Exam` structure, or at least defining the fields based on simpler types, as it can be seen here: 

```python
from datetime import (
    timedelta,
)

from minos.common import (
    Aggregate,
)


class Exam(Aggregate):
    name: str
    duration: timedelta
    # subject: ...
    # questions: ...
```
Then, an exam instance could be created as follows:
```python
Exam("Mid-term Exam", timedelta(hours=1, minutes=30))
```

One important thing to notice is that the `Aggregate` class also provides another additional fields related both with the identification and versioning that must not be provided by the developer (they are computed internally). The concrete fields are the following:
* `uuid: UUID` Unique identifier of the instance.
* `version: int` Version of the instance, computed as an auto-incremental integer starting from `1` during the creation process.
* `created_at: datetime` Creation timestamp of the instance.
* `updated_at: datetime` Timestamp of the last update of the instance.

But the real potential of the `Aggregate` class starts being visible when storage operations are performed:

### Storage Operations

Before starting to describe the available *CRUD* operations provided by the `Aggregate` class, it's important to notice one important part of the `minos` framework and how those operations are implemented. The nature of the framework is highly inspired by *Event Sourcing* thoughts, so that the aggregate is not simply stored on a collection that is being updated progressively without taking history into account, but the aggregate is stored as a sequence of incremental modifications known as `AggregateDiff` entries that acts as *Domain Events*. 

This set of events are stored on a *Repository* who is defined by the `minos.common.MinosRepository` interface, but also are exposed to another microservices over a *Broker*, who is defined by the `minos.common.MinosBroker` interface. The event based strategy has an important caveat, that is how to access the full aggregate as most business logic operations will probably require working with a full `Exam` instance (not only with the sequence of `AggregateDiff` instances). As the reconstruction each time a full instance is needed is a really inefficient operation, `minos` holds on a *Snapshot* who is defined by the `minos.common.MinosSnapshot` interface and provided direct access to the reconstructed instances. To know more about how *Event Sourcing* is implemented in `minos`, the [TODO: link to architecture] sections provides a detailed description.

The configuration file `service.injections` section of the configuration file allows to setup both the `reposiory`, `event_broker` and `snapshot` concrete classes:
```yaml
# config.yml

service:
  injections:
    event_broker: minos.networks.EventBroker
    repository: minos.common.PostgreSqlRepository
    snapshot: minos.common.PostgreSqlSnapshot
...
```


#### Create
TODO

```python
exam = await Exam.create("Mid-term Exam", timedelta(hours=1))
```

#### Update
TODO

```python
await exam.update(duration=exam.duration + timedelta(hours=1))
```


Additionally, the `Aggregate` class provides a `save()` method that automatically creates or updates the instance depending on if it's a new one or an already exising. Here is an example:
```python
exam = Exam("Mid-term Exam", timedelta(hours=1))
await exam.save()

exam.duration += timedelta(minutes=30)
await exam.save()
```

#### Delete
TODO

```python
await exam.delete()
```

#### Get
TODO

```python
identifier: UUID = ... 
exam = await Exam.get(identifier)
```

#### Find
TODO

```python
condition = ...
async for exam in Exam.find(condition):
    print(exam)
```

### Field Parsing
TODO

```python
class Exam(Aggregate):
    ...

    def parse_name(self, value: Any) -> Any:
        if not isinstance(value, str):
            return value
        return value.title()
```

### Field Validation
TODO

```python
class Exam(Aggregate):
    ...

    def validate_name(self, value: Any) -> bool:
        return isinstance(value, str) and len(value) >= 6
```



## Defining the `Subject` reference...
TODO

```python
from minos.common import (
    AggregateRef,
)


class Subject(AggregateRef):
    title: str
```

## Defining the `Question` entity... 
TODO

```python
from minos.common import (
    Entity,
    ValueObjectSet,
)

class Question(Entity):
    title: str
    # choices: ...
```

## Defining the `Choice` value object...
TODO

```python
from minos.common import (
    ValueObject,
)


class Choice(ValueObject):
    text: str
    correct: bool
```


## Full Picture...
TODO

```python
"""src/aggregates.py"""

from __future__ import (
    annotations,
)

from minos.common import (
    Aggregate,
    ModelRef,
    AggregateRef,
    ValueObject,
    ValueObjectSet,
    EntitySet,
    Entity,
)


class Exam(Aggregate):
    subject: ModelRef[Subject]
    questions: EntitySet[Question]


class Subject(AggregateRef):
    title: str


class Question(Entity):
    title: str
    choices: ValueObjectSet[Choice]


class Choice(ValueObject):
    text: str
    correct: bool

```