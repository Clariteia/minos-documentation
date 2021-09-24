# Exposing Operations: Command Service

## Setting up the service!

TODO

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
queries:
  service: src.ExamQueryService
commands:
  service: src.ExamCommandService
...
```

TODO

```python
"""src/commands.py"""

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
    """Exam Command Service class"""
```

## Exposing `Exam` creation...

```python
from minos.networks import (
    enroute,
    Response,
    Request,
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
    """Exam Command Service class"""
```