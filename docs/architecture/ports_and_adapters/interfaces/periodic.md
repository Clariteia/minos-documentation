# Periodic

## Introduction

This section provides a comprehensive explanation about periodic task execution and how `minos` implement this functionality. So, the first step is to define what a periodic task is:

**Periodic Task**: According to [Aroflo](https://help.aroflo.com/display/office/Periodic+Tasks) a periodic task is a task that can be automatically generated at regular intervals (e.g. daily, weekly, monthly, yearly). This is ideal for recurring work and preventative maintenance.

This definition mostly matches the idea of periodic tasks implemented on the `minos` framework. So, to define a periodic task, two important parts are needed, a function that acts as the task logic, and an interval that defines the periodicity of the function execution. 

There are many ways to define a time interval, but one of the most populars on the software industry is the one based on the [cron](https://en.wikipedia.org/wiki/Cron) command-line utility implemented by most of the unix systems. This strategy is based on a string pattern composed of 5 or more values representing the time (minutes, hours, days, etc.) and allows defining really complex rules without many effort. To understand better how the *cron* expressions works it's recommended to visit [this](https://en.wikipedia.org/wiki/Cron#CRON_expression) link and take a first contact on the [crontab guru](https://crontab.guru/) website. 

## Implementation

Once a brief introduction to the periodic tasks was provided, the next step is to understand how the `minos` framework implemented them. An important detail to notice at this point is that the framework strongly relies on the [asyncio](https://docs.python.org/3/library/asyncio.html), so this functionality is highly supported by that module.

### Periodic Task

The minimal unit of logic to start creating periodic tasks with the `minos` framework is the `minos.networks.PeriodicTask` class, which is composed by a *cron pattern* and a *callable function*. The class provides also the necessary methods to start, stop and get information about the execution status. 

An important detail to know about how the periodicity is implemented is that it only uses a single `asyncio.Task`, that runs an infinite loop paused with calls to `asyncio.sleep`. This is an important difference respect to another periodic task implementations based on `asyncio`, that instead of a single task, create a new task for each cycle postponing it with the `asyncio.AbstractEventLoop.call_later` method. But, to reduce the task creation overhead and keep the code simple `minos` follows the first approach.

[TODO: Include a diagram about periodic execution.]

* `PeriodicTask(crontab: str | crontab.CronTab, fn: Callable[[ScheduledRequest], Awaitable[None]])`: 
Construct a new instance containing the given crontab representing the periodicity of the task, and a callable function representing the task itself. The callable must follow the common prototype defined by :doc:`/architecture/ports_and_adapters/messaging/_toc`.

* `run_once()`:
Perform a single execution of the given callable.

* `run_forever()`:
Perform the execution of the periodic task blocking the workflow.

* `start()`:
Start the execution of the periodic task without blocking the workflow.

* `stop()`:
Stop the execution of the periodic task.

### Periodic Task Scheduler

The `minos.networks.PeriodicTaskScheduler` is the class which acts as a container of `PeriodicTask` instances so, its functionality is really simple. From one side, the tasks are set through the constructor and from the other side, the `start()` and `stop()` methods are the ones who transmit the corresponding call to all the tasks. 

[TODO: include a diagram about concurrent executions.]

* `PeriodicTaskScheduler(tasks: Iterable[PeriodictTask])`:
Construct a new instance containing the given iterable of tasks.

* `start()`:
Start the execution of all the tasks concurrently.
  
* `stop()`:
Stop the execution of all the tasks concurrently.

### Periodic Task Scheduler Service

As usual in `minos`, every component that needs to keep alive along the microservice execution needs to have an associated service. In this case, the `minos.networks.PeriodicTaskSchedulerService` takes this role. To know more about how services are implemented in `minos`, it's recommended to read the [TODO: include service section] section.

## Examples

[TODO: Include some examples]