# Domain Modeling

## Introduction

TODO


## Aggregate structure and Storage operations

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
    subject: ...
    questions: ...
```


## External Aggregate References

TODO

```python
from minos.common import (
    AggregateRef,
)


class Subject(AggregateRef):
    title: str
```


## Entities 
TODO

```python
from minos.common import (
    Entity,
    ValueObjectSet,
)

class Question(Entity):
    title: str
    choices: ...
```

## Value Objects
TODO

```python
from minos.common import (
    ValueObject,
)


class Choice(ValueObject):
    text: str
    correct: bool
```


## Full Picture
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