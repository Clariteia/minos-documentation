# Response

# Request 

`Minos` commands and queries might return some values to their callers. Thus, a `Response` class is provided by `Minos` that abstracts protocol specific details. 

The `Response` structure receives an object that is serialized and sent back to the caller.

```python
    @enroute.rest.query("/products", "GET")
    async def get_all_products(self, request: Request) -> Response:
        res = await self.repository.get_all()
        return Response(res)
```