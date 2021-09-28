# Exposing Information: Query Service

## Introduction

TODO


## Setting up the environment!

TODO
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

TODO

```python
class ExamQueryService(QueryService):
    
    @inject
    def __init__(self, repository: ExamQueryRepository = Provide["query_repository"], *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.repository = repository

    ...
```

TODO

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
TODO

## Handling `Exam` events...

```python
class ExamQueryService(QueryService):
    ...

    @enroute.broker.event("ExamCreated")
    async def exam_created(self, request: Request) -> None:
        diff = await request.content(resolve_references=False)
        self.repository.add(diff.uuid, diff.get_one("duration"), diff.get_one("subject"))

    @enroute.broker.event("ExamUpdated.duration")
    async def exam_duration_updated(self, request: Request) -> None:
        diff = await request.content()
        self.repository.update_duration(diff.uuid, diff.get_one("duration"))
        
    @enroute.broker.event("ExamUpdated.subject")
    async def exam_subject_updated(self, request: Request) -> None:
        diff = await request.content(resolve_references=False)
        self.repository.update_subject(diff.uuid, diff.get_one("subject"))

    @enroute.broker.event("ExamDeleted")
    async def exam_deleted(self, request: Request) -> None:
        diff = await request.content()
        self.repository.delete(diff.uuid)
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
    async def exam_added(self, request: Request) -> None:
        diff = await request.content(resolve_references=False)
        self.repository.add(diff.uuid, diff.get_one("duration"), diff.get_one("subject"))

    @enroute.broker.event("ExamUpdated.duration")
    async def exam_duration_updated(self, request: Request) -> None:
        diff = await request.content()
        self.repository.update_duration(diff.uuid, diff.get_one("duration"))
        
    @enroute.broker.event("ExamUpdated.subject")
    async def exam_subject_updated(self, request: Request) -> None:
        diff = await request.content(resolve_references=False)
        self.repository.update_subject(diff.uuid, diff.get_one("subject"))

    @enroute.broker.event("ExamDeleted")
    async def exam_deleted(self, request: Request) -> None:
        diff = await request.content()
        self.repository.delete(diff.uuid)
```