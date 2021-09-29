# Deployment

## Introduction

TODO

## Command Line Interface

TODO

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

TODO

```python
"""src/__main__.py"""

from .cli import (
    main,
)

if __name__ == "__main__":
    main()
```

### Poetry

TODO

```python
poetry add typer
```

TODO

```toml
# pyproject.toml
...
[tool.poetry.scripts]
microservice = "src.cli:main"
...
```
TODO

```toml
poetry run microservice start
```

TODO

### Pipenv
TODO

### Pip
TODO

## Docker

TODO

```dockerfile
# Dockerfile

FROM ghcr.io/clariteia/minos:0.1.4 as development

COPY ./pyproject.toml ./poetry.lock ./
RUN poetry install --no-root
COPY . .
CMD ["poetry", "run", "microservice", "start"]

FROM development as build
RUN poetry export --without-hashes > req.txt && pip wheel -r req.txt --wheel-dir ./dist
RUN poetry build --format wheel

FROM python:slim as production
COPY --from=build /microservice/dist/ ./dist
RUN pip install --no-deps ./dist/*
COPY ./config.yml ./config.yml
ENTRYPOINT ["microservice"]
CMD ["start"]
```

TODO

## Kubernetes
TODO