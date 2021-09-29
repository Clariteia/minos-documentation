# Exposing Information: Query Service

## Introduction

After being described how to define the aggregate in the :doc:`/quickstart/aggregate` section and being exposed some operations that change its state in the :doc:`/quickstart/command` section, so the next step is to describe how to expose queries. As said in previous sections, the `minos` framework implements the [CQRS](https://martinfowler.com/bliki/CQRS.html) pattern, in which the *Query Model* is decoupled from the *Command Model*, allowing to orient each one to its specific purpose and together giving the most of themselves. 

So, the `QueryService` can be seen as an independent component with respect to the rest of the microservice, allowing to externalize it without big effort. So, it should use a dedicated database, different from the one that stores the `AggregateDiff` events, the `Aggregate` snapshots and so on. Then, at this point the main question is how to populate the `QueryService` database without using another components of the microservice nor accessing the main database. Here, the answer is through event subscription, that provides a decoupled way to stay informed about any change produced over the `Aggregate`s defined both on the target microservice and in external ones. 

The `QueryService` workflow can be seen as a set of writing operations that updates the database, that are launched when specific `AggregateDiff` events are published, and a set of read operations that queries the database, that are launched when another microservices or external clients perform query operations.

One important thing to notice at this point is the consistency degree of the `QueryService` respect to the `Aggregate` state and their stored events: As the way to update the query database is based on event subscription, which generates a certain amount of delay between the publication of the event and the reception, a strong consistency cannot be guaranteed. Alternatively, eventual consistency is the offered guarantee, that means that the `QueryService` is consistent respect to the received `AggregateDiff` events.

## Setting up the environment!

After being described the basic concepts about how the `QueryService` works, the next step is start setting up the one for the `exam` microservice. The first part is to add the configuration parameters into the `config.yml` file (note that the ones related with interfaces are shared with the `CommandService`):

```yaml
# config.yml
service:
  injections:
    query_repository: src.queries.ExamQueryRepository
rest:
  host: 0.0.0.0
  port: 8082
broker:
  host: localhost
  port: 9092
  queue:
    database: exam_db
    user: minos
    password: min0s
    host: localhost
    port: 5432
    records: 1000
    retry: 2
queries:
  service: src.queries.ExamQueryService
...
```

After being updated the configuration file, the next step is to create the structure of the query service, which will be defined into `src/queries.py`. To keep the code organized, the chosen pattern in this case is to split all the logic into two parts: One related with input and output parsing, performed by the `ExamQueryService`, and a second one related with database manipulation, performed by the `ExamQueryRepository`. 

The `ExamQueryService` inherits the `minos.cqrs.QueryService` that is a must condition to implement a query service, and the `ExamQueryRepository` inherits from the `minos.common.MinosSetup` class, who provides some interesting utility methods, like the `setup`, and `destroy`, really useful for creating and destroying database connections together with the repository instance. Another important detail is that the `ExamQueryRepository` will be injected as a dependency injection (as it is defined in the configuration file).

So, the `src/queries.py` file skeleton will be similar to:

```python
from minos.common import (
    MinosSetup,
)
from minos.cqrs import (
    QueryService,
)


class ExamQueryRepository(MinosSetup):
    ...
        

class ExamQueryService(QueryService):
    ...
```

## Configuring query classes...

After being defined the `ExamQueryService` and `ExamQueryRepository` classes, the next step is to start writing the initialization code. The `ExamQueryService` is the easier one, as it only needs to store the repository instance given from the dependency injection (note that the `dependency_injector` package is being used here, so it must be added as a dependency: e.g. `poetry add dependency-injector`): 

```python
from dependency_injector.wiring import (
    Provide,
    inject,
)

class ExamQueryService(QueryService):
    
    @inject
    def __init__(self, repository: ExamQueryRepository = Provide["query_repository"], *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.repository = repository

    ...
```

The case of the `ExamQueryRepository` requires a bit more attention as it's needed to setup the database connections and also provide the initialization logic composed of table creations, etc. Here a `sqlite3` database is chosen to store a simple `exams` table composed by three columns: `uuid`, `duration` and `subject`.

An interesting detail here is the use of the `_setup` and `_destroy` methods to create and clean the database connection, and also create the database table. These methods are called by the `minos.common.DependencyInjector` before and after performing the injection itself.

```python
class ExamQueryRepository(MinosSetup):
    
    def __init__(self, database: str = "exam.sqlite", *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.database = database
        self.connection = None
    
    async def _setup(self) -> None:
        self.connection = sqlite3.connect(self.database)
        cursor = self.connection.cursor()
        cursor.execute("CREATE TABLE IF NOT EXISTS exams (uuid TEXT PRIMARY KEY, duration INT, subject TEXT)")
        cursor.close()
        
    async def _destroy(self) -> None:
        if self.connection is not None:
            self.connection.close()
            self.connection = None
    
    ...
```

**Important**: For the sake of simplicity, here a `sqlite` database is being used, but in real environments another database systems are highly recommended.

TODO

## Handling `Exam` events...

```python
class ExamQueryService(QueryService):
    ...

    @enroute.broker.event("ExamCreated")
    async def exam_created(self, request: Request) -> None:
        diff = await request.content(resolve_references=False)
        
        try:
            uuid, duration, subject = diff.uuid, diff.get_one("duration"), diff.get_one("subject")
        except Exception:
            raise ResponseException("Some fields could not be extracted.")
        
        self.repository.add(uuid, duration, subject)

    @enroute.broker.event("ExamUpdated.duration")
    async def exam_duration_updated(self, request: Request) -> None:
        diff = await request.content()
        
        try:
            uuid, duration = diff.uuid, diff.get_one("duration")
        except Exception as exc:
            raise ResponseException(f"Some fields could not be extracted: {exc}")
        
        self.repository.update_duration(uuid, duration)
        
    @enroute.broker.event("ExamUpdated.subject")
    async def exam_subject_updated(self, request: Request) -> None:
        diff = await request.content(resolve_references=False)
        
        try:
            uuid, subject = diff.uuid, diff.get_one("subject")
        except Exception as exc:
            raise ResponseException(f"Some fields could not be extracted: {exc}")
        
        self.repository.update_subject(uuid, subject)

    @enroute.broker.event("ExamDeleted")
    async def exam_deleted(self, request: Request) -> None:
        diff = await request.content()
        
        try:
            uuid = diff.uuid
        except Exception as exc:
            raise ResponseException(f"Some fields could not be extracted: {exc}")
        
        self.repository.delete(uuid)
```

TODO

```python
class ExamQueryRepository(MinosSetup):
    
    ...
            
    def add(self, uuid: UUID, duration: timedelta, subject: UUID)-> None:
        params = (str(uuid), duration // timedelta(microseconds=1), str(subject))
        cursor = self.connection.cursor()
        cursor.execute("INSERT INTO exams (uuid, duration, subject) VALUES (?, ?, ?)", params)
        cursor.close()
        self.connection.commit()
        
    def update_duration(self, uuid: UUID, duration: timedelta)-> None:
        params = (duration // timedelta(microseconds=1), str(uuid))
        cursor = self.connection.cursor()
        cursor.execute("UPDATE exams SET duration = ? WHERE uuid = ?", params)
        self.connection.commit()

    def update_subject(self, uuid: UUID, subject: UUID)-> None:
        params = (str(subject), str(uuid))
        cursor = self.connection.cursor()
        cursor.execute("UPDATE exams SET subject = ? WHERE uuid = ?", params)
        cursor.close()
        self.connection.commit()

    def delete(self, uuid: UUID) -> None:
        params = (str(uuid), )
        cursor = self.connection.cursor()
        cursor.execute("DELETE FROM table WHERE uuid = ?", params)
        cursor.close()
        self.connection.commit()
```

TODO

## Defining `Exam`-related queries...

TODO

```python
class ExamQueryService(QueryService):
    ...
    
    @enroute.rest.query("/exams/q/exams-with-more-duration", "GET")
    @enroute.broker.query("GetExamsWithMoreDuration")
    async def exams_with_more_duration(self, request: Request) -> Response:
        content = await request.content()
        result = self.repository.get_longest_exams(content["limit"])
        return Response(result)

    @enroute.rest.query("/exams/q/subjects-with-more-exams", "GET")
    @enroute.broker.query("GetSubjectsWithMoreExams")
    async def subjects_with_more_exams(self, request: Request) -> Response:
        content = await request.content()
        result = self.repository.get_subjects_with_more_exams(content["limit"])
        return Response(result)

    ...
```

TODO

```python
ExamDurationDTO = ModelType.build("ExamDurationDTO", {"exam": UUID, "duration": timedelta})
SubjectExamsDTO = ModelType.build("SubjectExamsDTO", {"subject": UUID, "exams": int})
```

TODO

```python
class ExamQueryRepository(MinosSetup):
    
    ...
            
    def get_longest_exams(self, limit: int = 100) -> list[ExamDurationDTO]:
        params = (limit, )
        cursor = self.connection.cursor()
        cursor.execute("SELECT uuid, duration FROM exams ORDER BY duration DESC LIMIT ?", params)
        rows = cursor.fetchall()
        cursor.close()
            
        return [ExamDurationDTO(*row) for row in rows]
            
    def get_subjects_with_more_exams(self, limit: int = 100) -> list[SubjectExamsDTO]:
        params = (limit, )
        cursor = self.connection.cursor()
        cursor.execute(
            "SELECT subject, COUNT(uuid) AS exams FROM exams GROUP BY subject ORDER BY exams DESC LIMIT ?", params
        )
        rows = cursor.fetchall()      
        cursor.close()
        return [SubjectExamsDTO(*row) for row in rows]
    
    ...
```

## Summary

TODO

```python
"""src/queries.py"""

import sqlite3
from datetime import (
    timedelta
)
from uuid import (
    UUID,
)
from dependency_injector.wiring import (
    inject, 
    Provide,
)
from minos.common import (
    ModelType,
    MinosSetup,
)
from minos.cqrs import (
    QueryService,
)
from minos.networks import (
    Request,
    Response,
    ResponseException,
    enroute,
)


ExamDurationDTO = ModelType.build("ExamDurationDTO", {"exam": UUID, "duration": timedelta})
SubjectExamsDTO = ModelType.build("SubjectExamsDTO", {"subject": UUID, "exams": int})


class ExamQueryRepository(MinosSetup):
    
    def __init__(self, database: str = "exam.sqlite", *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.database = database
        self.connection = None
    
    async def _setup(self) -> None:
        self.connection = sqlite3.connect(self.database)
        cursor = self.connection.cursor()
        cursor.execute("CREATE TABLE IF NOT EXISTS exams (uuid TEXT PRIMARY KEY, duration INT, subject TEXT)")
        cursor.close()
        
    async def _destroy(self) -> None:
        if self.connection is not None:
            self.connection.close()
            self.connection = None
            
    def get_longest_exams(self, limit: int = 100) -> list[ExamDurationDTO]:
        params = (limit, )
        cursor = self.connection.cursor()
        cursor.execute("SELECT uuid, duration FROM exams ORDER BY duration DESC LIMIT ?", params)
        rows = cursor.fetchall()
        cursor.close()
            
        return [ExamDurationDTO(*row) for row in rows]
            
    def get_subjects_with_more_exams(self, limit: int = 100) -> list[SubjectExamsDTO]:
        params = (limit, )
        cursor = self.connection.cursor()
        cursor.execute(
            "SELECT subject, COUNT(uuid) AS exams FROM exams GROUP BY subject ORDER BY exams DESC LIMIT ?", params
        )
        rows = cursor.fetchall()      
        cursor.close()
        return [SubjectExamsDTO(*row) for row in rows]
            
    def add(self, uuid: UUID, duration: timedelta, subject: UUID)-> None:
        params = (str(uuid), duration // timedelta(microseconds=1), str(subject))
        cursor = self.connection.cursor()
        cursor.execute("INSERT INTO exams (uuid, duration, subject) VALUES (?, ?, ?)", params)
        cursor.close()
        self.connection.commit()
        
    def update_duration(self, uuid: UUID, duration: timedelta)-> None:
        params = (duration // timedelta(microseconds=1), str(uuid))
        cursor = self.connection.cursor()
        cursor.execute("UPDATE exams SET duration = ? WHERE uuid = ?", params)
        self.connection.commit()

    def update_subject(self, uuid: UUID, subject: UUID)-> None:
        params = (str(subject), str(uuid))
        cursor = self.connection.cursor()
        cursor.execute("UPDATE exams SET subject = ? WHERE uuid = ?", params)
        cursor.close()
        self.connection.commit()

    def delete(self, uuid: UUID) -> None:
        params = (str(uuid), )
        cursor = self.connection.cursor()
        cursor.execute("DELETE FROM table WHERE uuid = ?", params)
        cursor.close()
        self.connection.commit()


class ExamQueryService(QueryService):
    
    @inject
    def __init__(self, repository: ExamQueryRepository = Provide["query_repository"], *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.repository = repository

    @enroute.rest.query("/exams/q/exams-with-more-duration", "GET")
    @enroute.broker.query("GetExamsWithMoreDuration")
    async def exams_with_more_duration(self, request: Request) -> Response:
        content = await request.content()
        limit = content["limit"]
        result = self.repository.get_longest_exams(limit)
        return Response(result)

    @enroute.rest.query("/exams/q/subjects-with-more-exams", "GET")
    @enroute.broker.query("GetSubjectsWithMoreExams")
    async def subjects_with_more_exams(self, request: Request) -> Response:
        content = await request.content()
        limit = content["limit"]
        result = self.repository.get_subjects_with_more_exams(limit)
        return Response(result)

    @enroute.broker.event("ExamCreated")
    async def exam_created(self, request: Request) -> None:
        diff = await request.content(resolve_references=False)
        
        try:
            uuid, duration, subject = diff.uuid, diff.get_one("duration"), diff.get_one("subject")
        except Exception:
            raise ResponseException("Some fields could not be extracted.")
        
        self.repository.add(uuid, duration, subject)

    @enroute.broker.event("ExamUpdated.duration")
    async def exam_duration_updated(self, request: Request) -> None:
        diff = await request.content()
        
        try:
            uuid, duration = diff.uuid, diff.get_one("duration")
        except Exception as exc:
            raise ResponseException(f"Some fields could not be extracted: {exc}")
        
        self.repository.update_duration(uuid, duration)
        
    @enroute.broker.event("ExamUpdated.subject")
    async def exam_subject_updated(self, request: Request) -> None:
        diff = await request.content(resolve_references=False)
        
        try:
            uuid, subject = diff.uuid, diff.get_one("subject")
        except Exception as exc:
            raise ResponseException(f"Some fields could not be extracted: {exc}")
        
        self.repository.update_subject(uuid, subject)

    @enroute.broker.event("ExamDeleted")
    async def exam_deleted(self, request: Request) -> None:
        diff = await request.content()
        
        try:
            uuid = diff.uuid
        except Exception as exc:
            raise ResponseException(f"Some fields could not be extracted: {exc}")
        
        self.repository.delete(uuid)
```