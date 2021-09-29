# Modeling the Domain: Aggregates

## Introduction

In opposite to monolithic systems who use relational database model to represent and store the business logic, microservice-based systems requires applying slightly different techniques. One of the main reasons to start thinking different is based on the splitting of a single but big information schema into multiple smaller ones, as each microservice has its own dedicated database. 

The `minos` proposal is based on the *Domain Drive Design (DDD)* ideas, supported by *Entities*, *Value-Objects* and an *Aggregate* (or *Root Entity*) that represents the main concept of the microservice. These set of concepts allows us to model information at microservice level, but at some cases it's needed to relate information from one microservice with information from another one. The way to do that in `minos` is over *Aggregate References*. To understand these concepts in more detail, the :doc:`/architecture/data_model` section provides a more detailed explanation about them.

## Defining the `Exam` aggregate...

As it is advanced at the beginning of the :doc:`/quickstart/_toc` guide, the study case will be to define an `exam` microservice. This one will be able to store all the needed information related with a common *exam* composed of questions with selectable answers. For the sake of simplicity, all of them must be multiple answer questions, but it's a good practising exercise to continue working on these case and add another functionalities.

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
Exam("Mid-term", timedelta(hours=1, minutes=30))
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
...
```

After being introduced how aggregate persistence is implemented in `minos`, the next sections provide a reference guide about how to use the storage operations step by step. One important thing to notice is that all of them are implemented using *awaitables*, so it's needed to know the [asyncio](https://docs.python.org/3/library/asyncio.html) basis to get the most of them. 

#### Create

To create new aggregate instances, the best choice is to use the `create()` class method, which is similar to creating an instance directly calling the class constructor, but also stores a *creation event* into the *Repository*, so that the persistence is guaranteed. Then, the *Broker* will publish the *update event* on the `{$AGGREGATE_NAME}Created` topic. In addition to that, the retrieved instance has already set the auto-generated fields (`uuid`, `version`, etc.).

For example, creating an `Exam` aggregate can be done with:

```python
exam = await Exam.create("Mid-term", timedelta(hours=1))
```

The `exam` instance will be like:

```python
Exam(
    uuid=..., # generated uuid
    version=1, 
    created_at=..., # generated datetime 
    updated_at=..., # generated datetime
    name="Mid-term", 
    duration=timedelta(hours=1),
)
```

And the corresponding *creation event* will be like:

```python
AggregateDiff(
    action=Action.CREATE,
    name="src.aggregates.Exam",
    uuid=..., # generated aggregate uuid
    version=1,
    created_at=..., # generated aggregate datetime
    fields_diff=FieldDiffContainer(
        [
            FieldDiff(
                name="name", 
                type_=str, 
                value="Mid-term"
            ),
            FieldDiff(
                name="duration", 
                type_=timedelta, 
                value=timedelta(hours=1)
            ),
        ]
    )
)
```


#### Update

The update operation modifies the value of some fields that are composing the aggregate. The way to do that is through the `update()` method, which get the set of values to be updated as named arguments, in which the given name matches with the corresponding field name. Internally, the method computes the fields difference between the previous and new fields and then stores an *update event* into the *Repository*, so that the persistence is guaranteed. Then, the *Broker* will publish the *update event* on the `{$AGGREGATE_NAME}Updated` topic.  In this case, also the `version` and `updated_at` fields are updated according to the new changes.

For example, updating the `duration` field from the previously created `Exam` aggregate can be done with:

```python
await exam.update(duration=exam.duration + timedelta(minutes=30))
```

The `exam` instance will be like:

```python
Exam(
    uuid=...,
    version=2, # updated
    created_at=..., 
    updated_at=..., # updated
    name="Mid-term",
    duration=timedelta(hours=1, minutes=30), # updated
)
```

And the corresponding *update event* will be like:

```python
AggregateDiff(
    action=Action.UPDATE,
    name="src.aggregates.Exam",
    uuid=...,
    version=2,
    created_at=..., # generated datetime
    fields_diff=FieldDiffContainer(
        [
            FieldDiff(
                name="duration", 
                type_=timedelta, 
                value=timedelta(hours=1, minutes=30)
            ),
        ]
    )
)
```


Additionally, the `Aggregate` class provides a `save()` method that automatically *creates* or *updates* the instance depending on if it's a new one or an already exising. Here is an example:

```python
exam = Exam("Mid-term", timedelta(hours=1))
await exam.save()

exam.duration += timedelta(minutes=30)
await exam.save()
```

#### Delete

After being explained who to create and update instances, the remaining operation is the deletion one. In the `minos` framework it's implemented with a `delete` method, that internally stores a *deletion event* in to the *Repository* so that the persistence is guaranteed. Then, the *Broker* will publish the *update event* on the `{$AGGREGATE_NAME}Deleted` topic.

For example, deleting an instance can be done with:
```python
await exam.delete()
```

In this case, does not make any sense to continue working with the `exam` instance anymore, but the corresponding *delete event* will be like:

```python
AggregateDiff(
    action=Action.DELETE,
    name="src.aggregates.Exam",
    uuid=...,
    version=3,
    created_at=..., # generated datetime
    fields_diff=FieldDiffContainer.empty()
)
```

One important thing to notice is that the `create`, `update` and `delete` operations are writing operations, so all of them generates some kind of event to be stored internally and notified to others so that these operations requires to use the *Repository* and *Broker* components. However, the following operations (`get` and `find`) are reading operations, so the execution of them does not generate any events. Also, as it will be explained later, these operations are related with `Aggregate` instances, so it's needed to use the *Snapshot* component, whose purpose is to provide a simple and efficient way to access them.

#### Get

The way to obtain an instance based on its identifier is calling the `get` class method, which returns a single one, failing if it does not exist or is already deleted. In this case, the *Snapshot* guarantees a strong consistency respect to the *Repository* of events. 

```python
original = await Exam.create(...)

identifier = original.uuid

recovered = await Exam.get(identifier)

assert original == recovered
```

As the `get` class method only retrieves one instance at a time, a good option to retrieve multiple instances concurrently, together with the validation checks (existence and not already deletion) is to use `asyncio`'s `gather` function:

```python
from asyncio import (
    gather,
)

uuids: list[UUID] = ...

exams = await gather(*(Exam.get(uuid) for uuid in uuids))
```

If the validation checks are not needed, or can be performed directly at application level, a better option is to use the `find` class method.

#### Find

Previously described operations have one important thing in common, that is all of them need to know the exact aggregate instance to work with, in other words, all of them needed to know the exact identifier of the instance. Is true that in many cases, this is enough to resolve many use cases, but there are some situations in which a more advanced search is needed. A common example is when it's needed to apply operations to a set of instances characterised by special conditions and so on.   

The way to perform this kind of queries is with the `find` class method, which not only filters instances according to a given `Condition`, but can also return them with some specific `Ordering` and `limit` the maximum number of instances. For more details about how to write complex queries is highly recommended reading the [minos.common.queries](https://clariteia.github.io/minos_microservice_common/api/minos.common.queries.html) reference documentation. Another thing to know about the `find` class method is that it returns the obtained instances over an `AsyncIterator` and supports a streaming mode directly from the database if the `streaming_mode` flag is set to `True`.

Here is an example of a relatively complex `find` operation, that will return a ranking of `"Mid-term"` exams with more duration created during the last week: 
```python
from datetime import (
    date,
    timedelta,
)
from minos.common import (
    Condition,
    Ordering,
)


condition = Condition.AND(
    Condition.GREATED_EQUAL("created_at", date.today() - timedelta(days=7)), 
    Condition.EQUAL("name", "Mid-term")
)

ordering = Ordering.DESC("duration")

async for exam in Exam.find(condition, ordering, limit=10):
    print(exam)
```

### Field Parsing

There are some specific cases in which the fields should be transformed according to specific rules before setting them into the aggregate instances. To do that, `minos` is able to automatically recognizes the methods that matches the `parse_${FIELD_NAME}(value: Any) -> Any` and triggers them before setting the corresponding values.

For example, a parsing method can be used to capitalize the `Exam`'s name:

```python
class Exam(Aggregate):
    ...

    def parse_name(self, value: Any) -> Any:
        if not isinstance(value, str):
            return value
        return value.title()
```

### Field Validation

Similar to field parsing, `minos` provides also the capabilities to implement custom validation methods if type checking is not enough. Concretely, the framework automatically recognizes the methods that matches the `validate_${FIELD_NAME}(value: Any) -> bool` and triggers them before setting the corresponding values. If the validation result is evaluated to `False`, then a validation exception is raised and the value is not set.

For example, a validation function can be used to require that `Exam`'s name have at least three characters.

```python
class Exam(Aggregate):
    ...

    def validate_name(self, value: Any) -> bool:
        return isinstance(value, str) and len(value) >= 6
```


## Defining the `Subject` reference...

After being described the basics about aggregates and how to perform operations with them, the next natural step to be able to create complex data models is to learn how to create relations with another aggregates. One important thing to notice here is that whereas multiple *Aggregate* classes can coexist within the same microservice, the recommended strategy is to keep only one *Aggregate* per microservice. Then, assuming that it's needed to create a relation from one to another, `minos` provides the concept of *Aggregate References*, which allow to define the schema of the external *Aggregate*, so that the business logic within that microservice will be able to support on its fields.  

For example, if it's needed to relate the `Exam` aggregate with a supposed `Subject` aggregate defined on another microservice then using the `AggregateRef` will be the solution. One of the reasons why we need to have a reference with `Subject` could be to use the `title` field for some kind of query defined in the `QueryService` (which is described on :doc:`/quickstart/query`).

```python
from minos.common import (
    AggregateRef,
)


class Subject(AggregateRef):
    title: str
```

After being defined the `AggregateRef`, the next step is to create the relation with the `Aggregate` one. The way to do that is using the `ModelRef` type hint. 
The reason why it exists is because *Aggregate References* should be stored as references on the database, but in some cases, it's needed to be resolved to use some of their fields.  The `ModelRef` is mostly defined as:
```python
ModelRef[T] = Union[T, UUID] # in which T is an AggregateRef type 
```

Then, the `Exam` aggregate turns into:
```python
from minos.common import (
    ModelRef,
)


class Exam(Aggregate):
    name: str
    duration: timedelta
    subject: ModelRef[Subject]
    # questions: ...
```

## Defining the `Question` entity... 

One of the most important parts of an exam is the set of questions. Here, the `Question` will be modeled as an `Entity` descendant. The *Entity* classes are characterised for being parts of the `Aggregate` one, with the extra capability to be uniquely identified based on an internal identifier. To know more about the `Entity` class, it's recommended to read the [TODO: link to architecture]. 

In this case, for sake or simplicity, the `Question` entity will only have two attributes: a `title`, which will contain the question to be answered, and a set of `answers` to be picked. But initially, `answers` will be ignored:
```python
from minos.common import (
    Entity,
)

class Question(Entity):
    title: str
    # answers: ...
```

After being defined the `Question` entity, the next step is to integrate it into the `Exam` aggregate. In this case, it's needed to have a special feature, which is multiplicity. 

A possible solution could be to use a `list` or something similar so that `questions: list[Question]` resolves the problem, but this approach has a caveat, that is the event publication. Using the `list` class, the `questions` field is treated as a standard field and the generated events will publish the full list of questions also when only one of them has a small change. In some cases this could be the needed behaviour, but another one can be used.

The `EntitySet` class is the best way to store *Entities* in most cases, as it provides *incremental storing capabilities* that provides a big speedup when a big amount of them is stored. In terms of usage, the `EntitySet` inherits from the [collections.abc.MutableSet](https://docs.python.org/3/library/collections.abc.html#collections.abc.MutableSet) base class, so it can be used as a standard Python's `set`, so that includes the common `add`, `remove`, `__contains__`, etc. methods, but are specially adapted for `Entity` instances. 

In any case, the real advantage to use the `EntitySet` are the *incremental storing capabilities*, that is, instead of storing the full entity set after each `Aggregate.update` call, only the creations, updates or deletions are stored (and also published over the event system). 

For example, if the `EntitySet` is chosen for the `questions` field, the `Exam` aggregate turns into:
```python
from minos.common import (
    EntitySet,
)


class Exam(Aggregate):
    name: str
    duration: timedelta
    subject: ModelRef[Subject]
    questions: EntitySet[Question]
```
Inserting a `Question`, is as simple as inserting it into any `set`:

```python
exam.questions.add(Question("What is 1 + 1?"))
```

The generated event when `await exam.save()` is called in this case is similar to:

```python
AggregateDiff(
    action=Action.UPDATE,
    name="src.aggregates.Exam",
    uuid=...,
    version=3,
    created_at=..., # generated datetime
    fields_diff=FieldDiffContainer(
        [
            IncrementalFieldDiff(
                name="questions", 
                type_=Question, 
                value=Question("What is 1 + 1?"), 
                action=Action.CREATE
            ),    
        ]
    )
)
```


## Defining the `Answer` value object...

The last part before the `Exam` aggregate is ready is to define the available answers for the questions. In this case, the `Answer` class will be defined as a `ValueObject` descendant. *Value Object* classes are characterised by both qualities: They are immutable and the way to identify them is through their field values. To know more about this concept it's recommended to read the [TODO: link to architecture].

In this case, the `Answer` class will be composed of two simple attributes, the `text` containing the answer itself and a `correct` boolean flag. This strategy will allow to extend the implementation to multiple-answer questions without so much effort.

```python
from minos.common import (
    ValueObject,
)


class Answer(ValueObject):
    text: str
    correct: bool
```

Similarly to the `Entity` case, to store a collection of `Value Object` instances, it's possible to use the `list`, but the best choice for most cases is the `ValueObjectSet` class, that also heirs from the [collections.abc.MutableSet](https://docs.python.org/3/library/collections.abc.html#collections.abc.MutableSet) base class. The main difference between `ValueObjectSet` and `EntitySet` is that in this case the hashing is performed from the field values of each `ValueObject` instance, instead of simply using the `Entity`'s unique identifier for obvious reasons.


Then, if the `ValueObjectSet` is chosen as the collection of answers, the `Question` class will turns on:

```python
from minos.common import (
    Entity,
    ValueObjectSet,
)


class Question(Entity):
    title: str
    answers: ValueObject[Answer]
```

Now, the way to create `Answer` instances is as follows:
```python
question_identifier: UUID = ...
question = exam.questions.get(question_identifier)

question.answers.add(Answer("2", True))
question.answers.add(Answer("5", False))

await exam.save()
```

One important thing to not in this case is how `AggregateDiff` works in cases like this one. Here, a `ValueObjectSet` (with incremental capabilities) is used inside `Entity` instances stored inside an `EntitySet`, that is one of the fields of the `Aggregate` class. So, this is a special case because the incremental capabilities only work at first field level respect to the `Aggregate`. 

In this case, the generated `AggregateDiff` will be like:
```python
AggregateDiff(
    action=Action.UPDATE,
    name="src.aggregates.Exam",
    uuid=...,
    version=3,
    created_at=..., # generated datetime
    fields_diff=FieldDiffContainer(
        [
            IncrementalFieldDiff(
                name="questions", 
                type_=Question, 
                value=Question("What is 1 + 1?", ValueObjectSet([Answer("2", True), Answer("5", False)])), 
                action=Action.UPDATE
            ),    
        ]
    )
)
```

## Summary

After being described step by step the main features of the `Aggregate` class, and also the commonly used classes that relate with, here is a full snapshot of the resulting `src/aggregates.py` file:

```python
"""src/aggregates.py"""

from __future__ import (
    annotations,
)
from datetime import (
    timedelta,
)
from typing import (
    Any,
)

from minos.common import (
    Aggregate,
    AggregateRef,
    Entity,
    EntitySet,
    ModelRef,
    ValueObject,
    ValueObjectSet,
)


class Exam(Aggregate):
    name: str
    duration: timedelta
    subject: ModelRef[Subject]
    questions: EntitySet[Question]
    
    def parse_name(self, value: Any) -> Any:
        if not isinstance(value, str):
            return value
        return value.title()
    
    def validate_name(self, value: Any) -> bool:
        return isinstance(value, str) and len(value) >= 6


class Subject(AggregateRef):
    title: str


class Question(Entity):
    title: str
    answers: ValueObjectSet[Answer]


class Answer(ValueObject):
    text: str
    correct: bool

```