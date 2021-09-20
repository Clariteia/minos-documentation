# Aggregate

## Introduction

TODO


```python
"""src/aggregates.py"""

from typing import (
    Optional,
)
from uuid import (
    uuid4,
)
from minos.common import (
    Aggregate
)

class Product(Aggregate):
    """Product class."""

    code: str
    title: str
    description: str
    price: float

    def __init__(self, *args, code: Optional[str] = None, **kwargs):
        if code is None:
            code = uuid4().hex.upper()[0:6]
        super().__init__(code, *args, **kwargs)
```
