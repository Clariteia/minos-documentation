# Exposing Information: Query Service

## Introduction

TODO


## Setting up the service!

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
from datetime import (
    timedelta,
)
from uuid import (
    UUID,
)
from minos.common import (
    ModelType,
)


ExamDurationDTO = ModelType.build("ExamDurationDTO", {"exam": UUID, "duration": timedelta})
SubjectExamsDTO = ModelType.build("SubjectExamsDTO", {"subject": UUID, "exams": int})
```

TODO

## Get the longest exams...

TODO

## Get the subjects with more exams...

TODO

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
SubjectExamsDTO = ModelType.build("SubjectExamsDTO", {"subject": UUID, "exams": list[UUID]})


class ExamQueryRepository(MinosSetup):
    
    def __init__(self, database: str = "exam.sqlite", *args, **kwargs):
        super().init(*args, **kwargs)
        self.connection = sqlite3.connect(database)
    
    async def _setup(self) -> None:
        with self.connection.cursor() as cursor:
            cursor.execute("CREATE TABLE IF NOT EXISTS exams (uuid TEXT PRIMARY KEY, duration INT, subject TEXT)")
            
    def get_longest_exams(self, limit: int = 100) -> list[ExamDurationDTO]:
        params = (limit, )
        with self.connection.cursor() as cursor:
            cursor.execute("SELECT uuid, duration FROM exams ORDER BY duration DESC LIMIT ?", params)
            rows = cursor.fetchall()
            
        return [ExamDurationDTO(*row) for row in rows]
            
    def get_subjects_with_more_exams(self, limit: int = 100) -> list[SubjectExamsDTO]:
        params = (limit, )
        with self.connection.cursor() as cursor:
            cursor.execute(
                "SELECT subject, COUNT(*) AS exams FROM exams GROUP_BY subject ORDER BY exams DESC LIMIT ?", params
            )
            rows = cursor.fetchall()      
            
            return [SubjectExamsDTO(*row) for row in rows]
            
    def add(self, uuid: UUID, duration: timedelta, subject: UUID)-> None:
        params = (str(uuid), duration // timedelta(microseconds=1), str(subject))
        with self.connection.cursor() as cursor:
            cursor.execute("INSERT INTO exams (uuid, duration, subject) VALUES (?, ?, ?)", params)
    
    def update_duration(self, uuid: UUID, duration: timedelta)-> None:
        params = (duration // timedelta(microseconds=1), str(uuid))
        with self.connection.cursor() as cursor:
            cursor.execute("UPDATE exams SET duration = ? WHERE uuid = ?", params)

    def update_subject(self, uuid: UUID, subject: UUID)-> None:
        params = (str(subject), str(uuid))
        with self.connection.cursor() as cursor:
            cursor.execute("UPDATE exams SET subject = ? WHERE uuid = ?", params)

    def delete(self, uuid: UUID) -> None:
        params = (str(uuid), )
        with self.connection.cursor() as cursor:
            cursor.execute("DELETE FROM table WHERE uuid = ?", params)
        

class ExamQueryService(QueryService):
    
    @inject
    def __init__(self, repository: ExamQueryRepository = Provide["query_repository"], *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.repository = repository

    @enroute.broker.query("GetExamsWithMoreDuration")
    async def exams_with_more_duration(self, request: Request) -> Response:
        content = await request.content()
        limit = content["limit"]
        result = self.get_longest_exams(limit)
        return Response(result)

    @enroute.broker.query("GetSubjectsWithMoreExams")
    async def subjects_with_more_exams(self, request: Request) -> Response:
        content = await request.content()
        limit = content["limit"]
        result = self.get_subjects_with_more_exams(limit)
        return Response(result)

    @enroute.broker.event("ExamAdded")
    async def exam_added(self, request: Request) -> None:
        diff = await request.content()
        self.repository.add(diff.uuid, diff.get_one("duration"), diff.get_one("subject"))

    @enroute.broker.event("ExamUpdated.duration")
    async def exam_duration_updated(self, request: Request) -> None:
        diff = await request.content()
        self.repository.update_duration(diff.uuid, diff.get_one("duration"))
        
    @enroute.broker.event("ExamUpdated.subject")
    async def exam_subject_updated(self, request: Request) -> None:
        diff = await request.content()
        self.repository.update_subject(diff.uuid, diff.get_one("subject"))

    @enroute.broker.event("ExamDeleted")
    async def exam_deleted(self, request: Request) -> None:
        diff = await request.content()
        self.repository.delete(diff.uuid)
```