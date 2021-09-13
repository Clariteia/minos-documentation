Minos being a microservices' framework, it's very important to support a REST port to access it. Thus, Minos REST
Handler keeps a REST server active for clients to consume its endpoints.

# Configured through decorators

REST endpoints' handlers are bind to endpoints using Python decorators, as in Flask, FastAPI and the like.

```python
@enroute.rest.query("/login", "GET")
async def get_token(self, request: Request) -> Response:
    pass
```

# Request parameter
The `request` parameter is a Minos Request object that contains the data sent through the network.

```python
content = await request.content()
username = content["username"]
```

