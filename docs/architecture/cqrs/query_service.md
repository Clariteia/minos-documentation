# Query Service

The `Query Service`, on the other hand, is defined in the `queries.service` section within the `config.yml` file. Similar to the `Command Service`, it must inherit from `QueryService`.

```python
class TicketQueryService(QueryService):
    @enroute.broker.query("GetTicketQRS")
    @enroute.rest.query(f"/tickets/{{uuid:{UUID_REGEX.pattern}}}", "GET")
    async def get_ticket(self, request: Request) -> Response:
        ...
```

The actual power of a query service lies in its independent database. Thus, although the command service works as an event sourced platform, the query service can perform queries using any other model, no matter it is a graph, a document oriented model or so.

How does `Minos` help you craft that database? `Minos` encourages the use of a `Repository` that abstracts the creation and access to that model you want to use.

```python
class OrderQueryRepository(MinosSetup):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.engine = create_engine("postgresql+psycopg2://{user}:{password}@{host}:{port}/{database}".format(**kwargs))
        self.session = sessionmaker(bind=self.engine)()

    async def _setup(self) -> None:
        META.create_all(self.engine)

    @classmethod
    def _from_config(cls, *args, config: MinosConfig, **kwargs) -> OrderQueryRepository:
        return cls(*args, **(config.repository._asdict() | {"database": "order_query_db"}) | kwargs)
```

By extending `MinosSetup`, `Minos` handles the creation of the model by extending the `_setup()` method. Then queries can be performed within it the get data from the data source.

```python
class OrderQueryRepository(MinosSetup):
    async def create(self, **kwargs) -> None:
        kwargs = {k: v if not isinstance(v, FieldDiff) else v.value for k, v in kwargs.items()}

        kwargs["ticket_uuid"] = kwargs["ticket"]
        kwargs["payment_uuid"] = kwargs["payment"]
        kwargs["customer_uuid"] = kwargs["customer"]["uuid"]
        kwargs["payment_detail"] = dict(kwargs["payment_detail"])
        kwargs["shipment_detail"] = dict(kwargs["shipment_detail"])

        kwargs.pop("payment")
        kwargs.pop("ticket")
        kwargs.pop("customer")

        query = ORDER_TABLE.insert().values(**kwargs)
        self.engine.execute(query)
```

The repository must be connected to the `QueryService` class so it is 