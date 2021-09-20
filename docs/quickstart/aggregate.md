# Aggregate

## Introduction

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
    EntityObjectSet,
)


class Exam(Aggregate):
    subject: ModelRef[Subject]
    questions: EntityObjectSet[Question]


class Subject(AggregateRef):
    title: str


class Question(ValueObject):
    title: str
    choices: ValueObjectSet[Choice]


class Choice(ValueObject):
    text: str
    correct: bool


```