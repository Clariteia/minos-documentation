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
    Vehicle,
)


class VehicleCommandService(CommandService):
    """Vehicle Command Service class"""
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
    Vehicle,
)


class VehicleQueryService(QueryService):
    """Vehicle Query Service class"""
```