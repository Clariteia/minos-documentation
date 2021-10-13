# Deployment

## Introduction

The last but not the least important step of building microservices with the `minos` framework is the deployment one, in which everything related with the microservice running will be explained. At this point, the ``minos.common.EntrypointLauncher`` class plays an important role, being the one which is able to orchestrate all the internal services within the microservice.

## Command Line Interface

 A common way to start a ``minos`` microservice is executing a command over the shell. To do that, the microservice must provide a command line interface. A good alternative to do that is using the [typer](https://typer.tiangolo.com/) library, that can be installed as follows:

### Poetry

The `typer` library can be included with the following command: 

```python
poetry add typer
```

Then, to expose the `microservice` command and link it with the `main` function that will be defined on the `src/cli.py` file, the following lines are needed:

```toml
# pyproject.toml
...
[tool.poetry.scripts]
microservice = "src.cli:main"
...
```

Then, the way to activate the Poetry's virtual environment can be done with:

```toml
poetry shell
```

And now, the `microservice` command should be available on the current shell. The next step is to add the CLI code.

### PipEnv
[TODO: Include pipenv guide.] 

### Pip
[TODO: Include pip guide.]


After being set up the `microservice` command to be linked to the `main` function defined on the `src/cli.py` file, the next step is to properly add the `src/cli.py` file. Here is a standard template that can be used among most of the microservices created with the `minos` framework: 

```python
"""src/cli.py"""

import sys
from pathlib import (
    Path,
)
from typing import (
    Optional,
)

import typer
from minos.common import (
    EntrypointLauncher,
)

app = typer.Typer()


@app.command("start")
def start(
    file_path: Optional[Path] = typer.Argument(
        "config.yml", help="Microservice configuration file.", envvar="MINOS_CONFIGURATION_FILE_PATH",
    )
):
    """Start the microservice."""
    launcher = EntrypointLauncher.from_config(file_path, external_modules=[sys.modules["src.queries"]])
    launcher.launch()


@app.callback()
def callback():
    """Minos microservice CLI."""


def main():  # pragma: no cover
    """CLI's main function."""
    app()
```

The most interesting part here is the `start` function, that handles the `microservice start` command. The purpose of this function is to create an `EntrypointLauncher` instance from the configuration file (typically the `config.yml`) and then execute its `launch()` method, that orchestrates everything needed to start the microservice. 

An important detail to notice is the purpose of the `external_modules` argument, which in this case receives a list containing the `src.queries` module. That is the way to "register" modules that must be linked to the `minos` dependency injection system. In this case, is needed to define a single `query_repository` instance among all the microservice and use it by the `ExamQueryService`.

Another way to call the `src.cli:main` function is directly by the module's entrypoint i.e. the `__main__.py` file. In this case, it can be done as follows:

```python
"""src/__main__.py"""

from .cli import (
    main,
)

if __name__ == "__main__":
    main()
```

Once everything related to the command line interface is explained, the next part is to talk about the containerization of the microservice. 

## Containerization

## Docker

TODO

```dockerfile
# Dockerfile

FROM ghcr.io/clariteia/minos-microservice:0.1.5 as development

COPY ./pyproject.toml ./poetry.lock ./
RUN poetry install --no-root
COPY . .
CMD ["poetry", "run", "microservice", "start"]

FROM development as build
RUN poetry export --without-hashes > req.txt && pip wheel -r req.txt --wheel-dir ./dist
RUN poetry build --format wheel

FROM python:3.9-slim as production
COPY --from=build /microservice/dist/ ./dist
RUN pip install --no-deps ./dist/*
COPY ./config.yml ./config.yml
ENTRYPOINT ["microservice"]
CMD ["start"]
```

TODO

## Kubernetes
TODO