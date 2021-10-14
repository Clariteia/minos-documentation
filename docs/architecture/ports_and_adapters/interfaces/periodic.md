# Periodic

## Introduction

This section provides a comprehensive explanation about periodic task execution and how `minos` implement this functionality. So, the first step is to define what a periodic task is:

**Periodic Task**: According to [Aroflo](https://help.aroflo.com/display/office/Periodic+Tasks) a periodic task is a task that can be automatically generated at regular intervals (e.g. daily, weekly, monthly, yearly). This is ideal for recurring work and preventative maintenance.

This definition mostly matches the idea of periodic tasks implemented on the `minos` framework. So, to define a periodic task, two important parts are needed, a function that acts as the task logic, and an interval that defines the periodicity of the function execution. 

There are many ways to define a time interval, but one of the most populars on the software industry is the one based on the [cron](https://en.wikipedia.org/wiki/Cron) command-line utility implemented by most of the unix systems. This strategy is based on a string pattern composed of 5 or more values representing the time (minutes, hours, days, etc.) and allows defining really complex rules without many effort. To understand better how the *cron* expressions works it's recomended to visit [this](https://en.wikipedia.org/wiki/Cron#CRON_expression) link and take a first contact on the [crontab guru](https://crontab.guru/) website. 

## Implementation

[TODO: Include an explanation about how periodic tasks are implemented in minos]

## Examples

[TODO: Include some examples]