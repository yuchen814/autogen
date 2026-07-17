# AGENTS.md — microsoft/autogen

> **Maintenance mode.** AutoGen no longer receives new features; it is community
> managed. Successor: [Microsoft Agent Framework](https://github.com/microsoft/agent-framework).
> Prefer bug fixes, doc fixes, and maintenance changes — do not add new features.

Monorepo with two independent toolchains: `python/` and `dotnet/`. Cross-language
design docs live in `docs/design/` (01–05); shared protobuf definitions in `protos/`.
Work in one toolchain at a time — they share concepts (event-driven agents,
CloudEvents, gRPC worker protocol) but not tooling.

## Python (`python/`)

Single **uv workspace**; packages live in `python/packages/`: `autogen-core`,
`autogen-agentchat`, `autogen-ext`, `autogen-studio`, `autogen-magentic-one`,
`magentic-one-cli`, `agbench`, `autogen-test-utils`, `component-schema-gen`,
`pyautogen` (legacy 0.2).

```sh
cd python
uv sync --all-extras          # setup — uv is the only supported package manager
source .venv/bin/activate     # all poe tasks require the venv
poe check                     # run ALL CI checks; required before a PR
```

- Individual tasks (poethepoet): `poe format`, `poe lint` (ruff), `poe test` (pytest,
  async tests use `@pytest.mark.asyncio`), `poe mypy`, `poe pyright` — code must pass
  **both** type checkers. `run_task_in_pkgs_if_exist.py` fans tasks out across
  workspace packages; scope one with `poe --directory ./packages/<pkg>/ <task>`.
- Docs (Sphinx + myst, source `python/docs/src/`): `poe docs-build`, `poe docs-serve`,
  `poe docs-check`, `poe docs-clean` (then rebuild to refresh API refs),
  `poe docs-check-examples`, `poe samples-code-check`. Markdown code blocks are
  syntax-checked (`python/check_md_code_blocks.py`).
- New package: use the cookiecutter template in `python/templates/new-package/`
  (validates name `^[a-zA-Z][\-a-zA-Z0-9]+$`, auto-registers in the uv workspace).
- AutoGen Studio frontend (`packages/autogen-studio/frontend/`): Gatsby + TailwindCSS,
  **yarn** (`yarn install`, `yarn start`, serves on :8000).

## .NET (`dotnet/`)

Two package families — keep them separate:
- `AutoGen.*` — legacy 0.2-derived, gradually deprecated.
- `Microsoft.AutoGen.*` — event-driven model; APIs not stable.

Requires **.NET 9.0 SDK** (`dotnet/global.json`) plus the **.NET 8.0 runtime** for
tests (install steps in `.github/copilot-instructions.md`).

```sh
cd dotnet                     # solution: AutoGen.sln
dotnet restore
dotnet build --configuration Release --no-restore
dotnet test --configuration Release --no-build --filter "Category=UnitV2"
dotnet format --verify-no-changes                # CI enforces formatting
dotnet pack --configuration Release --no-build   # output: ./artifacts/package/release
```

- `dotnet/samples/Hello` is the canonical starting point for the new runtime.
- New projects are NOT packable by default; add
  `<Import Project="$(RepoRoot)/nuget/nuget-package.props" />` to the `.csproj`
  (see `dotnet/PACKAGING.md`).
- Website: `dotnet tool restore && dotnet tool run docfx website/docfx.json --serve` (:8080).
- CI: `.github/workflows/dotnet-build.yml`; nightly feed on Azure DevOps.

## Architecture (read before touching runtime code)

`docs/design/01`–`05`: pub/sub programming model, events as **CloudEvents**; agents
subscribe to **topics** (`TopicId` = type + source); agents identified by
`(type, key)` — type must be alphanumeric/underscore, not starting with a digit
(`04 - Agent and Topic ID Specs.md`). Runtime is in-memory in-process (Python and
.NET) or distributed: workers host agents and talk to a service over gRPC
(`03 - Agent Worker Protocol.md`); cross-language messages are protobuf (`protos/`).

## Conventions

- **Versioning** (`CONTRIBUTING.md`): all `autogen-*` packages are versioned together.
  Minor bump (0.X.0) = breaking change; patch (0.0.X) = features/bug fixes.
- **Release**: version-bump PR → git tag `vX.Y.Z` → per-package
  `single-python-package.yml` workflow with approval. NuGet: `dotnet/PACKAGING.md`.
- **PRs** (`.github/PULL_REQUEST_TEMPLATE.md`): explain *why*, link the issue
  (`Closes #1234`), tick the checklist (docs included, tests added, auto checks green).
- **Docstrings**: describe contract, all params/returns/exceptions, include a runnable
  example (checked by `docs-check-examples`). New/changed APIs need
  `.. versionadded::` / `.. versionchanged::` directives.
- **Security**: never report vulnerabilities in public issues — MSRC only (`SECURITY.md`).
- **Support routing**: bugs/features → GitHub Issues; questions → Discussions (`SUPPORT.md`).
- **CLA**: Microsoft CLA required (`CONTRIBUTING.md`).
- **Safety**: task-executing agent code (Magentic-One, agbench) runs in Docker
  containers/virtualenvs by default — preserve those isolation defaults.

## Doc map

| Topic | File(s) |
|---|---|
| Detailed agent/dev guide (timings, install steps, gotchas) | `.github/copilot-instructions.md` |
| Python setup, tasks, docs build | `python/README.md` |
| Versioning, release, triage, docstrings | `CONTRIBUTING.md` |
| .NET overview / packaging | `dotnet/README.md`, `dotnet/PACKAGING.md` |
| Architecture | `docs/design/01`–`05` |
| Security reporting | `SECURITY.md` |
| Support policy | `SUPPORT.md` |
| Responsible AI | `TRANSPARENCY_FAQS.md` |
| Migration 0.2 → 0.4 | `python/migration_guide.md` |
