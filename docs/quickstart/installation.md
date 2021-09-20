# Installation

## Introduction

Before starting to think in multiple microservices, distributed environments and so on, it's easier and better to start by knowing what components are needed to build a microservice individually. For sake of simplicity, each microservice can be seen as one small monolithic system that provides as limited or specialized functionality. Then, each microservice will probably need to interact with a storage system, contain some kind of business logic and expose that functionality over an external API, to be used by external clients and another microservices. 

## Modularity

To be able to provide all that environment functionalities, `minos` relies on external dedicated libraries. For example, to read or store data in postgres database `aiopg` is used, to publish or subscribe to kafka topics `aiokafka` is used and to expose a REST interface `aiohttp` is used. In any case, the `minos` philosophy is that the surrounding technologies must not be frozen to these and each of project must have its own ones so the list of supported databases and brokers will be increased in future `minos` releases.

An important thing to know is that `minos` relies on latest `Python` functionalities so the minimal required version is `3.9` or greater. One of the reasons are the use of *async* code together with *type hinted* code.

Another important detail about how the `minos` framework is how it is packaged. Instead of providing all the framework as a single and big component, it relies on `Python` [namespaces](https://packaging.python.org/guides/packaging-namespace-packages/) to provide a modular but organized way to install only the exact part of functionality that will be used. The main packages are the following ones:
* [minos-microservice-common](https://pypi.org/project/minos-microservice-common/): Contains the `minos.common` module.
* [minos-microservice-networks](https://pypi.org/project/minos-microservice-networks/):  Contains the `minos.networks` module.
* [minos-microservice-saga](https://pypi.org/project/minos-microservice-saga/): Contains the `minos.saga` module.
* [minos-microservice-cqrs](https://pypi.org/project/minos-microservice-cqrs/): Contains the `minos.cqrs` module.

## Package Managers

Before starting a `Python` project, a package manager is needed to install and manage external dependencies. As it's expected, `minos` is compatible with any of them, but the recommended one is `Poetry` as it provides simple but powerful functionalities.

### Poetry

To install poetry, it's recommended to follow its own guide, that can be found [here](https://python-poetry.org/docs/). After installing `poetry`, everything is ready to build the first microservice!

The first step is to create the project (the project name can be whatever you want, but a `minos` microservice convention is to name it as `src`):

```bash
poetry new product-microservice-example --name src
```

The project structure will be similar to: 

```bash
product-microservice-example
├── README.rst
├── pyproject.toml
├── src
│   └── __init__.py
└── tests
    ├── __init__.py
    └── test_src.py

2 directories, 5 files
```

To install the `minos` dependencies, simply exec the following command on the project's root directory:

```bash
poetry add \
  minos-microservice-common \
  minos-microservice-networks \
  minos-microservice-saga \
  minos-microservice-cqrs
```

After that, every `minos` dependencies are ready to be used. Let's move to the configuration step!