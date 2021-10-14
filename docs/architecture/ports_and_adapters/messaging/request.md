# Request 

Interface functions in `Minos` receive the request information through a `Request` object which encapsulates technology specific details and delivers just the needed data.

The structure of the Request has the following attributes:

- `content`: returns the body of the request.
- `user`: provides the user making the request. It's useful for authorization.

The `Request` class has some specializations for certain purposes such as `RestRequest`, which has the specific information provided within a REST request:

- `content` and `user` inherited from Request.
- `url_args`
- `path_args`

```python
    @enroute.broker.command("CreateReview")
    @enroute.rest.command("/reviews", "POST")
    async def create_review(self, request: Request) -> Response:
        content = await request.content()
        product = content["product"]
        user = content["user"]
        title = content["title"]
        description = content["description"]
        score = content["score"]

        product = await Review.create(product=product, user=user, title=title, description=description, score=score)

        return Response(product)
```