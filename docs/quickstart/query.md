# Exposing Information: Query Service

## Introduction

TODO

## Example

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