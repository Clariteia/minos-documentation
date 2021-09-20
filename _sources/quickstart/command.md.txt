# Exposing Operations: Command Service

## Introduction

TODO

## Example

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