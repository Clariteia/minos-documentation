# Ports and Adapters

The ports and adapters architecture is a generalization of the traditional layers architecture meant to emphasize that, in modern systems, data sources might be requests initiators as well as receivers. Thus, it creates de `Port` abstraction, meaning any external system that meets a certain protocol. An `Adapter` is then a piece of software which takes that request and converts it into an actually useful structure, usable by the core of the application.

Minos, aiming at building reactive microservices, follows a Ports and Adapters architecture in order to provide a way to easily extend the microservices' functionality with new technologies. 

Adapters are by default abstracted through `Minos` [decorators](decorators.md). This way, the developer can concentrate on writing valuable code instead of dealing with technology specific things.