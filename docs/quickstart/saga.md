# Building Interactions: Sagas

Implement each business transaction that spans multiple services is a saga. A saga is a sequence of local transactions.
Each local transaction updates the database and publishes a message or event to trigger the next local transaction in 
the saga. If a local transaction fails because it violates a business rule then the saga executes a series of 
compensating transactions that undo the changes that were made by the preceding local transactions.

In this section we will define a fictitious microservice `Attempt` and how to compose and call a SAGA.

## Introduction
The Exam aggregate can be viewed at :doc:`/quickstart/aggregate`

In order to make an example with SAGA we will need another fictitious microservice to simulate a communication.

In this case we are going to create one called attempts, in order to control for example that a user does not exceed 
the number of attempts of an exam.

```python
"""attempt/src/aggregates.py"""

from __future__ import (
    annotations,
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


class Attempt(Aggregate):
    user: ModelRef[User]
    exam: ModelRef[Exam]
    quantity: int
    
    def validate_quantity(self, quantity: int) -> bool:
        return isinstance(quantity, int) and len(quantity) >= 6

class User(AggregateRef):
    name: str


class Exam(AggregateRef):
    """Exam reference"""

```

## `Attempt`microservice

We need to define the Command to return if the number of attempts to perform the test for a user has been exceeded:

```python
"""attempt/src/commands.py"""

from minos.common import (
    EntitySet,
    Condition,
    MinosException,
)
from minos.cqrs import (
    CommandService,
)
from minos.networks import (
    Request,
    Response,
    enroute,
)
from minos.saga import (
    SagaContext,
)

from ..aggregates import (
    Attempt,
)
from .sagas import (
    ADD_CART_ITEM,
    DELETE_CART,
    REMOVE_CART_ITEM,
    UPDATE_CART_ITEM,
)

class AttemptCommandService(CommandService):
    """Cart Command Service class"""

    @staticmethod
    @enroute.broker.command("GetUserAttempts")
    async def get_user_attempts(request: Request) -> Response:
        """Create a new cart.
        :param request: A request instance containing the information to build a payment instance.
        :return: A response containing the newly created payment instance.
        """
        content = await request.content()
        user = content["user"]
        exam = content["exam"]
        
        condition = Condition.AND(
            Condition.EQUAL("user", user),
            Condition.EQUAL("exam", exam),
        )

        async for attempt in Attempt.find(condition, limit=1):
            if (attempt.quantity == 3):
                raise MinosException("The maximum number of attempts has been exceeded.")
            
            return Response(attempt.quantity)
        
        return Response(0)

```


## SAGA creation in `Exam`microservice
The saga consists of steps and are those that define the operations to be performed in an orderly way to communicate with other 
services. These steps normally contain the call to the corresponding service (`.invoke_participant("CreateOrder")`) and 
optionally a compensation (`.with_compensation("DeleteOrder", delete_order_callback)`) in case an error occurs or the 
operation cannot be performed by the business logic. It also optionally has a function in charge of 
receiving the response from the invoked service (which is used in case the response has to be processed).

For a SAGA instance to be valid, a `step`, `invoke_participant` and `commit` are required. So the minimum structure of 
a SAGA is:
```python
Saga()
.step()
.invoke_participant("GetUserAttempts")
.commit()
```

Now we can create the SAGA that would call the `GetUserAttempts` command of the Attempt microservice:

```python
"""exam/src/commands.py"""

from minos.saga import (
    Saga,
    SagaContext,
)
from src.aggregates import (
    Exam,
)
from minos.common import (
    Aggregate,
    Model,
    ModelType,
)
from uuid import (
    UUID,
)

AttemptQuery = ModelType.build("PaymentQuery", {"user": UUID, "exam": UUID})

def _prepare_data(context: SagaContext) -> Model:
    return AttemptQuery(context["user_uuid"], context["exam_uuid"])

async def commit_result(context: SagaContext) -> SagaContext:
    exam = await Exam.create(...)
    
    return SagaContext(exam=exam)
    
GET_USER_ATTEMPTS = (
    Saga()
    .step()
    .invoke_participant("GetUserAttempts", _prepare_data)
    .commit(commit_result)
)

```

1. Normally if you want to send information together with the command, ModelType.build is used:

    ```python
    AttemptQuery = ModelType.build("PaymentQuery", {"user": UUID, "exam": UUID})
    ```

2. To see the structure of the SAGA in detail visit :doc:`/architecture/saga`

## SAGA execution in `Exam`microservice
When sending the test results we will check that the user has not exceeded the limit of attempts, if so, an exception 
is thrown.

```python
"""exam/src/commands/services.py"""

from minos.cqrs import (
    CommandService,
)
from minos.networks import (
    Request,
    Response,
    ResponseException,
    enroute,
)
from minos.saga import (
    SagaContext,
    SagaStatus,
)

from ..aggregates import (
    PaymentDetail,
    ShipmentDetail,
)
from .sagas import (
    GET_USER_ATTEMPTS,
)

class ExamCommandService(CommandService):
    """Exam Service class"""

    @enroute.rest.command("/exams", "POST")
    @enroute.broker.command("CreateExam")
    async def submit_exam(self, request: Request) -> Response:

        content = await request.content()
        user_uuid = content["user_uuid"]
        exam_uuid = content["exam_uuid"]

        saga = await self.saga_manager.run(
            GET_USER_ATTEMPTS,
            context=SagaContext(
                user_uuid=user_uuid,
                exam_uuid=exam_uuid,
            ),
        )

        if saga.status == SagaStatus.Finished:
            return Response(dict(saga.context["exam"]))
        else:
            raise ResponseException("An error occurred during exam submit.")

```