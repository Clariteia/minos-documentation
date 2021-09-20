# Service

## Introduction

TODO

## Command Service

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

## Query Service

TODO

```python
"""src/queries.py"""

from minos.cqrs import (
    QueryService,
)
from minos.networks import (
    Request,
    Response,
    enroute,
)

from ..aggregates import (
    Exam,
)


class ExamQueryService(QueryService):
    """Exam Query Service class"""
```