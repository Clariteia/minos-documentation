# Command Service

Although `Minos` encourages you to create it in `src/commands`, any other location is possible as long as the path is set appropriately within the `commands` sections of the `config.yml` file. 

A `command` is defined through a class that inherits from `CommandService`. Then, command functions can already get created!

```python
class TicketCommandService(CommandService):
    async def create_ticket(self, request: Request) -> Response:
        ...
```

This command defines a business function of the domain, which means it is independent of the technology used to call it. How are available ports to access those functions specified, then? `Minos` provides the `enroute` decorator, which abstracts certain kind of port through which the `command` is accessed. For example, consider the option of creating a REST interface to call the command in the last example:

```python
class TicketCommandService(CommandService):
    @enroute.rest.command("/tickets", "POST")
    async def create_ticket(self, request: Request) -> Response:
        ...
```

One might as well create a port to access the same command thorugh a published event in the broker:
```python
class TicketCommandService(CommandService):
    @enroute.rest.command("/tickets", "POST")
    @enroute.broker.command("CreateTicket")
    async def create_ticket(self, request: Request) -> Response:
        ...
```
