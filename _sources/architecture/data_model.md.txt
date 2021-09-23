# Data Model

## Introduction

As every software system, `Minos` encourages the use of a particular data model that restricts how the system
works. `Minos` follows the analysis patterns proposed in Domain Driven Design (DDD), which have more than 20 years of
proven validity to tackle software complexity.

## Entity

An `entity` is the minimum level of abstraction possible. It represents a simple business concept that owns its
attributes and the functions that transform them. It can be something like a wheel of a car or the car itself.

`Entities` are responsible for making the system speak the ubiquitous language of the business. This means the terms
used for both operations and attributes must convey meaning of the system.

```python
class TicketEntry(Entity):
    title: str
    unit_price: float
    quantity: int
    product: ModelRef[Product]
```

### EntitySet

TODO

## ValueObject

A `value object` is a concept of the domain whose identity is given by the whole set of its attributes' values. For
example, an address does not have an identity other than its street, number and so, and if any of them are modified, a
new address is actually being created. Thus, `Minos` encourages `value object`'s immutability

```python
class Address(ValueObject):
    street: str
    street_no: int
```

### ValueObjectSet

TODO

## Aggregate

An Aggregate is a composition of entities that work together in order to reach a complex goal. One of the main
objectives of such constructions is to encapsulate business complexity behind an `aggregate` root, which is an `entity`
taking that role. Thus, no message will be sent to each particular entity within the `aggregate`, but rather the
aggregate root will handle and route each request.

Because of this reason, there is a one to one correspondence among and `aggregate` and a microservice. This means a
microservice will encapsulate all the logic related to a business concept and provide the API for all the `aggregate`
components.

```python
class Ticket(Aggregate):
    code: str
    total_price: float
    entries: EntitySet[TicketEntry]
```

### AggregateRef

No software system consists just of one microservice. Microservices always come in groups and collaborate to provide the
expected behaviour. For that reason, `Minos` provides an `AggregateRef` concept, which defines the dependencies that a
microservice has with other microservices. 

### ModelRef

TODO

### Migrations

TODO