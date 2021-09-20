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
    Product,
)


class ProductCommandService(CommandService):
    """Product Command Service class"""

    @staticmethod
    @enroute.rest.command("/products", "POST")
    @enroute.broker.command("CreateProduct")
    async def create_product(request: Request) -> Response:
        """Create a product.

        :param request: A request instance containing the information to build a product instance.
        :return: A response containing the newly created product instance.
        """
        content = await request.content()
        
        title = content["title"]
        description = content["description"]
        price = content["price"]

        product = await Product.create(title, description, price)

        return Response(product)
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
    Product,
)


class ProductQueryService(QueryService):
    """Product Query Service class"""

    pass
```