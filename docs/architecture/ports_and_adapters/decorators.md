# Decorators

## What are decorators?
Decorators is an elegant syntactic way that Python provides for extending what other functions do. Using the `@` keyword one can annotate another function using such decorators. 

`Minos` uses decorators to extend the functionality of certain methods and eliminate some boilerplate code that other way would distract the developer from its actual goal: building and application. Mainly, those decorators are used in the Command and Query services to define the means through which they are called.

Let's consider the following example:

```python
    @enroute.broker.command("CreateProduct")
    @enroute.rest.command("/products", "POST")
    async def create_product(self, request: Request) -> Response:
        ...
```
Here we have a `create_product()` command, which deals with the business logic of creating a product within our system. By using the `@enroute` decorator, `Minos` extends its functionality and attaches that function to a REST endpoint as well as to an event.

## `Minos` decorators
The `@enroute` decorator is divided in the following categories:
- `@enroute.broker` deals with event subscription
  - `@enroute.broker.command(event: str)` attaches the method to a command invoked through the broker.
  - `@enroute.broker.query(event: str)` attaches the method to a query invoked through the broker. 
  - `@enroute.broker.event(event: str)` attaches the method to an event published by a microservice.
- `@enroute.rest` deals with REST endpoint definition
  - `@enroute.rest.command(endpoint: str, verb: str)` attaches the method to a REST endpoint in the command service.
  - `@enroute.rest.query(endpoint: str, verb: str)` attaches the method to a REST endpoint in the query service.
