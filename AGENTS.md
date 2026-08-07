# Instructions

## Scope

These instructions apply to the entire repository.

If a deeper directory contains its own `AGENTS.md`, follow the more specific instructions for files within that directory.

## Rules

Before making changes, read the rules relevant to the task.

Always follow:

* [`rules/clean-code.md`](rules/clean-code.md)
* [`rules/file-naming.md`](rules/file-naming.md)
* [`rules/git.md`](rules/git.md)

Apply additional rules based on the files and technologies involved:

* Markdown: [`rules/markdown.md`](rules/markdown.md)
* Python: [`rules/python-general.md`](rules/python-general.md)
* Python packaging: [`rules/python-packaging.md`](rules/python-packaging.md)
* Python tooling and dependencies: [`rules/python-uv.md`](rules/python-uv.md)
* TypeScript: [`rules/typescript.md`](rules/typescript.md)
* Bun: [`rules/typescript-bun.md`](rules/typescript-bun.md)
* React: [`rules/react.md`](rules/react.md)
* Next.js: [`rules/next-js.md`](rules/next-js.md)
* Astro: [`rules/astro.md`](rules/astro.md)
* Research-related tasks only: [`rules/research.md`](rules/research.md)

Multiple rule files may apply to the same change. Follow all applicable rules.

When rules conflict, prefer the more specific rule over the more general rule.

## Existing Projects

Respect the existing architecture, tooling, and conventions of the project being modified.

Do not introduce a new framework, package manager, formatter, linter, test runner, or architectural pattern solely to conform to these boilerplate preferences when the existing project has an established alternative.

Prefer incremental changes over unrelated refactoring.

## Verification

After making changes, run the relevant formatter, linter, type checker, tests, and build commands defined by the project.

Do not consider a change complete while known relevant checks are failing.
