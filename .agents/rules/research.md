# Research Episode Guidelines

## Purpose

Each research episode should represent a self-contained, reproducible
experiment.

An episode must contain enough information, configuration, implementation, and
outputs for another researcher or LLM agent to understand and reproduce the
experiment without relying on undocumented context.

## Episode Directory

Store experiments under the `episodes/` directory.

Each episode directory must begin with its creation date in `YYYY-MM-DD` format,
followed by a short descriptive name.

```text
episodes/2026-08-03-memory-retrieval/
```

Use one episode for one coherent research question or experimental
investigation.

If an episode contains multiple related experimental variants, represent them as
**subepisodes** within the same episode rather than creating unrelated directory
structures.

## Required Structure

An episode should follow this structure:

```text
episodes/<YYYY-MM-DD>-<episode-name>/
├── README.md
├── configuration-<subepisode>.yaml
├── __main__.py
├── artifact-<subepisode>/
├── figure-<figure-name>.py
├── figure-<figure-name>.pdf
└── REPORT.md
```

Multiple configuration files and artifact directories may exist when the episode
contains multiple subepisodes.

## `README.md`

`README.md` describes the design of the research episode.

It should explain:

- the research objective
- the question or hypothesis being investigated
- the experimental design
- the meaning of each subepisode
- important assumptions
- inputs and datasets
- how the experiment can be reproduced
- any external dependencies or unavoidable nondeterminism

The file must begin with YAML frontmatter containing:

```yaml
---
title: ...
date: YYYY-MM-DD
description: ...
---
```

Keep the frontmatter concise and informative.

The frontmatter should be sufficient for quickly determining what an episode
contains without reading the entire document.

## Subepisodes

Use subepisodes when multiple experiment configurations investigate the same
overall research question.

Each subepisode must have a descriptive name.

For example:

```text
configuration-baseline.yaml
configuration-long-context.yaml
configuration-short-context.yaml

artifact-baseline/
artifact-long-context/
artifact-short-context/
```

The same subepisode name should be used consistently across configuration files,
generated artifacts, reports, and other references.

Avoid names based only on sequence numbers such as:

```text
configuration-1.yaml
configuration-2.yaml
```

Prefer names that communicate the experimental distinction.

## `configuration-<subepisode>.yaml`

Each subepisode must have an explicit configuration file.

The configuration file is the **authoritative source of runtime configuration**
for that experiment.

Any parameter or choice that can materially affect experimental behavior must be
declared explicitly in the configuration.

Examples include:

- model and model version
- dataset and dataset revision
- input paths
- random seeds
- sampling parameters
- experimental conditions
- evaluation settings
- iteration counts
- feature flags
- external service options

Keep experiment settings in the YAML configuration. Do not introduce additional
configuration sources through command-line options, direct environment access,
implicit defaults, or machine-specific state. The required `experiment_tag`
argument only selects the configuration file.

The intended execution model is:

```text
configuration-<subepisode>.yaml
        ↓
     __main__.py
        ↓
artifact-<subepisode>/
```

Exceptions are acceptable only when an external system requires a specific
interface and changing it would break compatibility.

### Secrets and Credentials

Declare references to secrets, including API keys and authentication credentials,
in `configuration-<subepisode>.yaml` using OmegaConf resolvers. Keep the actual
secret values outside source code and version-controlled configuration files.

For secrets supplied through the process environment, use the built-in
[`oc.env` resolver][omegaconf-environment-resolver]:

```yaml
api_key: ${oc.env:EXPERIMENT_API_KEY}
```

Supply the referenced environment variables before launching the episode. Episode
code must not load `.env` files, call `load_dotenv()`, or read credentials directly
through `os.getenv()` or `os.environ`.

Resolve these references with `OmegaConf.to_container(..., resolve=True)` at the
entry point, validate the result with `Configuration`, and pass credentials
explicitly from that configuration to clients. Do not rely on clients implicitly
discovering credentials from the environment.

Do not provide fallback values for required secrets; a missing secret must fail
configuration loading. Document the required environment variable names in the
episode's `README.md` without recording their values. Never log or save resolved
secrets, including in configuration snapshots, artifacts, or reports.

[omegaconf-environment-resolver]: https://omegaconf.readthedocs.io/en/latest/custom_resolvers.html#oc-env

## `__main__.py`

`__main__.py` is the executable entry point for the episode.

It should:

1. explicitly load the selected subepisode configuration,
2. execute the experiment,
3. write experimental outputs into the corresponding `artifact-<subepisode>/`
   directory.

Running the experiment from the same source code and configuration should
reproduce the same experimental behavior as closely as the underlying systems
permit.

Do not place analysis or visualization logic in `__main__.py` unless it is an
essential part of producing the raw experimental artifact.

Use the following entry-point pattern. The required `experiment_tag` argument
selects the subepisode. Load its configuration with
OmegaConf, resolve interpolations, and validate it once with Pydantic before
passing it to the experiment.

Acquire a non-blocking exclusive lock for the selected tag before running the
experiment. After acquiring the lock, use `try`/`finally` to remove the lock file
when the experiment returns or raises. Keep lock acquisition outside the `try`
block so a process that fails to acquire the lock does not remove another
process's lock file. The file context manager closes the file and releases the
lock. This pattern requires a platform that supports `fcntl`.

```python
import asyncio
from pathlib import Path
from typing import Final

from pydantic import BaseModel

EPISODE_DIRECTORY: Final = Path(__file__).resolve().parent
EXPERIMENT_NAME: Final = EPISODE_DIRECTORY.name


class Configuration(BaseModel): ...


async def main(
    experiment_name: str,
    experiment_tag: str,
    configuration: Configuration,
) -> None: ...


if __name__ == "__main__":
    import argparse
    import fcntl

    from omegaconf import OmegaConf

    argument_parser = argparse.ArgumentParser()
    argument_parser.add_argument("experiment_tag")
    experiment_tag = str(argument_parser.parse_args().experiment_tag)
    raw_configuration = OmegaConf.to_container(
        OmegaConf.load(EPISODE_DIRECTORY / f"configuration-{experiment_tag}.yaml"),
        resolve=True,
    )
    configuration = Configuration.model_validate(raw_configuration)
    lock_path = EPISODE_DIRECTORY / f"{experiment_tag}.lock"

    with lock_path.open("a", encoding="utf-8") as lock_file:
        fcntl.flock(lock_file, fcntl.LOCK_EX | fcntl.LOCK_NB)

        try:
            asyncio.run(
                main(
                    experiment_name=EXPERIMENT_NAME,
                    experiment_tag=experiment_tag,
                    configuration=configuration,
                )
            )
        finally:
            lock_path.unlink()
```

## Experimental Artifacts

Store the outputs of each subepisode in:

```text
artifact-<subepisode>/
```

Artifact structures should remain consistent across subepisodes whenever
possible so that analysis code can operate on them uniformly.

If changes to `__main__.py` require changing the artifact schema or directory
structure, migrate existing artifacts in place to the new structure rather than
leaving multiple incompatible layouts.

Artifacts should contain the information necessary for later analysis without
depending on transient runtime state.

## Figures and Analysis

Keep analysis and visualization separate from experiment execution.

Create one script for each meaningful figure:

```text
figure-<figure-name>.py
```

The script should generate:

```text
figure-<figure-name>.pdf
```

For example:

```text
figure-success-rate.py
figure-success-rate.pdf
```

Figure scripts should consume recorded experiment artifacts rather than
rerunning or modifying the underlying experiment.

This separation allows:

- experiments to remain immutable after execution,
- analyses to be revised independently,
- figures to be regenerated from recorded artifacts,
- experimental results to be inspected without repeating expensive runs.

## `REPORT.md`

`REPORT.md` records the results and interpretation of the episode.

It should distinguish clearly between:

- experimental setup,
- observations,
- quantitative results,
- analysis,
- interpretation,
- limitations.

The report should refer to the corresponding configurations, artifacts, and
figures so that claims can be traced back to their experimental source.

## Reproducibility

Reproducibility takes precedence over convenience.

Any input that can materially affect an experiment must be visible and
controlled.

Where applicable:

- configure random seeds explicitly,
- record model names and versions,
- record dependency versions,
- identify datasets and dataset revisions,
- avoid dependence on wall-clock time,
- avoid ordering-dependent behavior from unordered collections,
- declare external service dependencies,
- document unavoidable nondeterministic components.

Do not introduce hidden sources of experimental variability.

If perfect determinism is impossible—for example, because an external model API
is nondeterministic—the source of nondeterminism should be documented rather
than silently ignored.

## Episode Completion

Before considering an episode complete, verify that:

- the episode directory follows the date-and-name convention,
- `README.md` explains the research question and experimental design,
- its YAML frontmatter accurately summarizes the episode,
- every subepisode has an explicit configuration file,
- experiment behavior does not depend on undeclared configuration,
- `__main__.py` reproduces the experiment from the configuration,
- each subepisode writes to a consistently structured artifact directory,
- analysis and visualization are separated from experiment execution,
- figures are reproducible from recorded artifacts,
- `REPORT.md` documents the results and their interpretation,
- important sources of nondeterminism are documented,
- another researcher or LLM agent can reproduce the episode without relying on
  undocumented assumptions.
