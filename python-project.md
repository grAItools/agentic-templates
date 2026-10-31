# Python Project Repository Blueprint

This blueprint defines a development stack for installable Python libraries
and command-line applications: package layout, dependencies, code quality,
testing, documentation, CI, and releases.

This blueprint builds on the [Harness-Agnostic Project Blueprint](harness-agnostic-project.md),
which defines shared instructions, skills, and harness configuration. It adds
the Python stack and workflow.

This document specifies a repository to create. Script names below define
interfaces to implement; this document does not ship a scaffold or generator.

## 1. Design principles

1. **Separate deliverables from development infrastructure.** Consumers install
   the package under `src/`; repository tooling, tests, and documentation have
   their own locations and distribution policies.
2. **Give each decision one owner.** Metadata and tool scope belong in
   `pyproject.toml`; executable checks belong in shared scripts. Hooks and CI
   reuse those definitions instead of maintaining separate command lists.
3. **Validate promises at their boundaries.** Test public behavior, supported
   platforms, package contents, and installation independently of editable
   development imports. Measure coverage as supporting evidence.
4. **Keep portability explicit.** Development commands work on supported
   platforms. Declare interpreter, shell, system-package, and network
   requirements rather than relying on one developer's environment.
5. **Make checks reproducible and inspectable.** Commit dependency resolution,
   reuse commands locally and in CI, reject stale dependency locks, and publish
   the artifacts that were actually verified.

## 2. Default profile and project decisions

The following is an opinionated default profile. Record deliberate variations
in `CONTRIBUTING.md` and keep commands, CI, and documentation consistent with
them.

| Concern | Default |
| --- | --- |
| Packaging | Hatchling; one typed import package under `src/` |
| Environment | uv; committed `uv.lock`; explicit `dev` and `docs` groups |
| Python | Start with a 3.12 minimum if dependencies permit; declare and test the actual supported range |
| Style | Ruff for lint, import ordering, and formatting; 100 columns; Google-style docstrings |
| Typing | mypy, Pyright, Zuban, and Pyrefly as blocking checks |
| Tests | pytest and coverage.py; branch measurement; 95% combined coverage gate |
| Hooks | prek using the pre-commit configuration format |
| Docs | Markdown and Zensical; strict builds |
| CI | GitHub Actions; Linux and Windows correctness tests |
| History | Conventional Commits and a maintained changelog |
| License | BSD-3-Clause when appropriate for the project's ownership and distribution policy |

Four type checkers and a 95% threshold are policy choices, not prerequisites
for Python packaging. Adopt this profile deliberately; document a different
checker set or coverage target when justified. Never quietly weaken an adopted
gate to accommodate a failing change. Select macOS and other platform jobs
when they are part of the support promise. Compiled extensions require an
additional platform-wheel and native-toolchain design beyond this pure-Python
baseline.

## 3. Repository layout

Use a distribution name such as `project-name` and a valid import name such as
`project_name`. Keep the distinction in metadata, imports, commands, and tests.

```text
project-name/
├── pyproject.toml
├── uv.lock
├── .python-version                 # Development interpreter pin
├── README.md
├── CONTRIBUTING.md
├── CHANGELOG.md
├── LICENSE
├── .license-header.txt              # If canonical source notices are adopted
├── .gitignore
├── .gitattributes
├── .pre-commit-config.yaml
├── Makefile                         # Optional thin command aliases
├── src/project_name/
│   ├── __init__.py                  # Deliberate public exports
│   ├── py.typed
│   ├── api.py                       # Optional public facade
│   ├── cli.py                       # Optional CLI implementation
│   ├── __main__.py                  # Optional python -m entry point
│   └── ...                          # Domain modules and subpackages
├── tests/
│   ├── conftest.py
│   ├── test_*.py
│   └── fixtures/                    # Optional input data
├── docs/
│   ├── index.md
│   ├── quickstart.md
│   ├── api.md
│   ├── architecture.md
│   └── ...                          # Configuration, limitations, assets as needed
├── zensical.toml
├── examples/                        # Optional runnable examples
├── scripts/
│   ├── check.py                     # Shared validation and formatting commands
│   ├── release.py                   # Metadata, Git-state, artifact validation
│   ├── package_smoke.py             # Isolated installed-artifact checks
│   └── license_headers.py           # If source notices are adopted
└── .github/
    ├── workflows/
    │   ├── ci.yml
    │   ├── docs.yml
    │   └── release.yml
    └── release-notes/               # Optional curated per-version summaries
```

The tree shows the Python development stack. Keep developer utilities in
`scripts/` and workflow-specific orchestration in `.github/workflows/`.
Git does not retain empty directories: use placeholders only where a directory
must survive cloning.

## 4. Package and dependency contracts

### Metadata and distribution contents

Define the Hatchling backend in `[build-system]`. Put the distribution name,
version, description, README, Python requirement, license, classifiers, URLs,
and runtime dependencies in `[project]` and its subtables. Set
`[tool.hatch.build.targets.wheel].packages = ["src/project_name"]`.
Require `hatchling>=1.27` in `build-system.requires` so the baseline uses modern
license metadata and core metadata 2.4 by default. Keep build constraints
compatible with this floor. See the [Hatchling release history](https://hatch.pypa.io/latest/history/hatchling/#1270-2024-11-26).
Use an SPDX license expression, such as `license = "BSD-3-Clause"`, and
`license-files = ["LICENSE"]`; omit deprecated `License ::` classifiers.

Declare wheel and source-distribution contents deliberately. The wheel must
contain `py.typed`, runtime resources, and required license metadata. The sdist
must contain everything needed to build the wheel without the original Git
checkout. Exclude private settings, scratch trees, environments, and
repository-only configuration from published distributions unless a documented
use requires them. Git ignore rules alone are not a complete distribution
policy. Define an explicit sdist allow-list, for example:

```toml
[tool.hatch.build.targets.sdist]
only-include = ["src", "pyproject.toml", "README.md", "LICENSE"]
```

This default omits repository tests and developer tooling. Include additional
build inputs when needed. If tests are shipped for downstream use, include
their fixtures and helpers and verify they run from the unpacked sdist.
Release validation must assert required and unexpected archive paths, accounting
for generated metadata and files Hatchling always includes. See
[Hatch file selection](https://hatch.pypa.io/latest/config/build/#file-selection)
and the [sdist defaults](https://hatch.pypa.io/latest/plugins/builder/sdist/#default-file-selection).

Keep `__init__.py` small and make public exports deliberate. Use `__all__` when
the package presents a facade. Organize implementation into domain modules,
introducing subpackages when there are meaningful boundaries. For a CLI,
declare `[project.scripts]` and have `__main__.py` call the same function in
`cli.py`, propagating its exit code.

Use `project.version` as the version authority and read the installed version
with `importlib.metadata` when runtime access is needed. If a different version
source is chosen, define it explicitly and validate every derived copy. Do not
silently return a plausible release version from an uninstalled source tree.

### Environments and compatibility

Put runtime requirements in `project.dependencies`, consumer-installable
features in `[project.optional-dependencies]`, and checkout tooling in
`[dependency-groups]`. Keep heavy optional test integrations in separate groups.
Set `[tool.uv].default-groups = ["dev"]` explicitly.

Maintain three separate declarations: `requires-python` for consumers,
`.python-version` for the default development interpreter, and the CI matrix
for verified Python/platform combinations. Override the interpreter explicitly
in matrix jobs. Default `.python-version` to the minimum supported Python so
local coverage and the canonical CI coverage job use the same baseline. A
local run on another interpreter supplements that baseline rather than replacing
it. Confirm that development-tool Python requirements do not
accidentally exclude a supported test interpreter; split tool and test groups
or environments if necessary.

Commit `uv.lock`. After editing dependencies, regenerate it and review the
change. CI starts with `uv sync --locked`; `--frozen` uses the lock without
checking freshness, so it cannot replace that validation. The lock controls
the selected development environment; consumers resolve declared package
requirements independently. See [uv locking and syncing](https://docs.astral.sh/uv/concepts/projects/sync/).

Give every direct runtime dependency, including dependencies of published
extras, a verified lower bound where the requirement format permits one.
For conditional requirements, verify each bound in an applicable environment.
Use a disposable minimum-Python environment and `lowest-direct` resolution
over the package's runtime requirements, without applying that strategy to
the development groups or modifying the normal lock. Take the direct test-tool
versions from `uv.lock`, constraining the tested direct runtime
dependencies to the selected minima so the tools cannot silently upgrade them.
If these requirements conflict, adjust the test harness or the declared floor.
This checks direct dependency floors, not every possible combination of
transitive dependencies. Test published extras and meaningful optional
integrations separately. See [uv resolution strategies](https://docs.astral.sh/uv/concepts/resolution/#resolution-strategy).

## 5. One executable workflow

Use `scripts/check.py` as the command owner. It invokes tools with argument
lists and the active Python environment, resolves paths relative to the
repository, and propagates failures. Keep its subcommand definitions in one
place; hooks, CI, and optional Make targets call those definitions.
Use portable Python for orchestration rather than requiring Make on Windows.

Implement the following interfaces:

| Subcommand of `scripts/check.py` | Responsibility |
| --- | --- |
| `lint` | Ruff lint and format checks; adopted source-header check |
| `type` | Every adopted type checker over its configured project scope |
| `test` | pytest without coverage |
| `coverage` | pytest under coverage.py; reports and the configured threshold |
| `docs` | Zensical strict build using the docs dependency group |
| `check` | `lint`, `type`, `coverage`, and `docs`; no autofixing |
| `fmt` | Explicitly requested source fixes and formatting |

The completion gate is `scripts/check.py check`: all four validation
subcommands above must pass. Invoke subcommands with
`uv run --locked python scripts/check.py <subcommand>`, adding `--group docs`
before `python` for `docs` and `check`. Dependencies are resolved before the
Python runner starts. A complete local workflow in a scaffold implementing
these interfaces is:

```bash
uv sync --locked --group docs
uv run --locked prek install
uv run --locked --group docs python scripts/check.py check
```

Configure both `pre-commit` and `commit-msg` installation when using a
Conventional Commits hook; keep ordinary file hooks out of the `commit-msg`
stage. Every clone installs its own hooks. Pin uv in CI and document the
required local version. Run Ruff and Python checkers from the locked project
environment; pin any separately downloaded hook environments too. Use local
`language: system` hooks whose entries call these shared commands through
`uv run --locked`. The default commit-time checks are `lint` and `type`;
tests, coverage, and documentation builds belong to the completion gate.

Keep tool scope and exclusions in `pyproject.toml`, including maintained
utilities under hidden directories. Whole-project hooks use
`pass_filenames: false`; changing a configuration file must trigger the checks
it affects. Use `always_run: true` for checks that must notice deletions or
configuration-only changes. Merge this stack's hooks with the foundation
blueprint's hooks in one configuration and preserve each
blueprint's ownership of its checks. Use the locked prek runner wherever that
blueprint says `pre-commit`; install one hook manager per clone. Shell-based hooks
retain the foundation blueprint's platform prerequisites. Windows contributors
using those hooks must satisfy those prerequisites; Python correctness tests
do not depend on running the entire hook suite.

`prek run --all-files` selects Git-known files for filename-based hooks; it may
miss untracked new files. The completion gate must discover maintained source
directly as well. Run the configured hook suite once in a Linux CI job so local
installation is not the only enforcement point. CI can distribute completion
subcommands across jobs: when the hook job already runs the full `lint` and
`type` scopes, separate jobs run `coverage` and `docs` without repeating them.
All those required jobs together satisfy the completion gate.
The all-files run does not exercise `commit-msg` hooks. Enforce Conventional
Commits separately in CI against PR commit messages, or the PR title when
squash merging is the required merge policy.

## 6. Code quality and tests

Use Ruff for formatting and the adopted lint rules. Enable the `D` rule family
with `[tool.ruff.lint.pydocstyle].convention = "google"`, relaxing missing
docstring rules where appropriate. Pick the remaining rule families explicitly
or use `ALL` with explained exclusions, and avoid redundant formatters. Adopt
module imports, absolute imports, and import-alias conventions where they help
the public design; allow intentional facade re-exports. Do not assume Ruff
proves that every imported symbol is a module.

For the default four-checker profile, configure mypy in strict mode and Pyright
in standard mode over package source and maintained utilities. Configure Zuban
and Pyrefly to include maintained tests, relaxing test signature requirements
where appropriate while checking their bodies. The wider pair covers tests
without requiring annotations on every fixture-driven function; mypy and
Pyright focus on the typed package and utilities. Give Zuban an explicit section
when its scope differs from mypy's. Resolve each checker's interpreter from
the active environment and explicitly set each checker's target Python version
to the package minimum, including when its own execution interpreter is newer.
Avoid hard-coded `.venv/bin/python` paths.

Pyright's Python wrapper targets a checker version but may provision Node
and npm assets outside the Python environment. Use `pyright[nodejs]` in the
locked dev group or provision a pinned Node runtime separately, and document
first-run network/cache requirements. Avoid environment overrides that select
an unpinned checker version. See the [wrapper documentation](https://github.com/RobertCraigie/pyright-python).

Keep deliberate invalid input and generated snapshots separate from maintained
test harness code. Exclude data only where checking or formatting it would
destroy the test's purpose. Verify hidden-path discovery and project-mode
exclusions for each checker; a successful exit must not conceal an empty scope.
Use accurate annotations and narrowing, with narrow documented suppressions
only at unavoidable boundaries. Standard `# type: ignore[code]` comments can
affect multiple checkers; do not treat them as private to the checker that
motivated them. Set Pyright's `enableTypeIgnoreComments = false` and use
`# pyright: ignore[rule]` for Pyright-specific exceptions. Enable unused-ignore
reporting where available, including `reportUnnecessaryTypeIgnoreComment` for
Pyright. Inspect the unsuppressed diagnostics before adding a shared ignore,
then rerun all adopted checkers for regressions; a passing rerun cannot prove
that the comment has hidden no other diagnostic. See
[Pyright configuration](https://github.com/microsoft/pyright/blob/main/docs/configuration.md).

Start with focused `tests/test_*.py` files. Add layers such as `unit/`,
`integration/`, `behavior/`, `golden/`, and `perf/` when the suite benefits.
Test supported public behavior, error behavior, and compatibility boundaries.
Use property, differential, corpus, and performance checks when the product's
claims need them; keep expensive external checks scheduled or manually
triggered and document when a change requires them before merging.

Configure coverage.py with the whole package as `source`, `branch = true`, and
`fail_under = 95` for the default profile. This threshold combines statements
and branches; it does not require 95% branch coverage independently. Produce
terminal, XML, and HTML reports even when tests or the threshold fail, while
preserving the failing final result. Subprocess execution needs separate
measurement configuration; do not count it implicitly.

By default, the Linux job on the minimum supported Python enforces this
threshold; other matrix jobs run correctness tests without a coverage gate.
If substantial platform-specific code requires combined measurement, collect
compatible branch data in the relevant jobs, configure relative paths and
path mappings, and apply the threshold after combining. Do not silently
exclude platform code to preserve a single-platform score. See
[coverage data paths](https://coverage.readthedocs.io/en/latest/config.html#run-relative-files).

Test architecture or documentation contracts where drift would matter. Ensure
examples run. An editable checkout test run does not establish that a wheel
contains required files or declares every runtime dependency; release
verification must exercise installed artifacts separately.

## 7. Documentation, CI, and release

Keep the README focused on purpose, installation, one working example, and
links. Put usage, public API, configuration, architecture, and limitations in
`docs/`. `CONTRIBUTING.md` owns the contributor workflow, tool prerequisites,
support matrix, completion gate, and release procedure. Build Zensical with
strict warning handling.

Consider the [Diátaxis framework](https://diataxis.fr/) for organizing user
documentation: tutorials guide learning, how-to guides address specific tasks,
reference describes APIs and configuration, and explanation develops concepts
and design rationale. Use these distinctions to shape pages and navigation as
the documentation grows, without requiring a fixed directory structure.

The API page may be handwritten; update it alongside public behavior and
cross-check its names and examples against the package. Automated extraction
is optional and requires a separately verified documentation integration.
Keep the foundation blueprint's project instructions concise
and link to `CONTRIBUTING.md` and the shared commands for authoritative workflow
details rather than maintaining a second command reference.

CI validates PRs and pushes to the integration branch. Sync with `--locked`,
selecting the `docs` group in the documentation job. Run the Linux hook job
and the completion subcommands as described above; test every declared Python
minor version on Linux. Cover at least the minimum and newest supported Python
on Windows using `scripts/check.py test`, without the merged shell-hook suite.
Linux versions other than the coverage baseline also use `scripts/check.py test`.
Add platforms when promised. Static checks run once on their declared
interpreter. Configure `.gitattributes` for LF, use explicit UTF-8 and newline
handling in test writers, and make path and interpreter discovery portable.

Only trusted integration-branch runs deploy documentation. If deployment
follows another workflow, use the exact verified commit or artifact, not the
branch's newer tip. Give validation read-only permissions and grant deployment
or publishing permissions only to the relevant jobs. Pin external actions and
tools and maintain those pins through reviewed updates.

An automated release follows this contract:

1. Resolve a `v<version>` tag to an immutable commit on the integration branch.
   A manual trigger resolves its input before building and never moves an
   existing tag. Require agreement with package metadata, changelog, and any
   other version copies; do not infer review or passing CI from ancestry alone.
2. Verify the completion gate on that commit, or require every CI job that
   collectively satisfies it to have passed on exactly that commit. Reject
   unexplained checkout changes, a stale
   lockfile, or inconsistent release metadata.
3. Build into an empty output directory with `uv build --no-sources`. This
   disables uv source overrides during build dependency resolution; it neither
   tests runtime dependency availability nor substitutes for installation
   checks. A development lockfile alone does not pin every isolated build
   requirement. Pin build tooling or supply build constraints when stronger
   repeatability is required. See [uv distribution builds](https://docs.astral.sh/uv/concepts/projects/build/)
   and its [CLI reference](https://docs.astral.sh/uv/reference/cli/#uv-build).
4. Validate the wheel and sdist against the archive policy. Install
   each independently into a fresh environment outside the checkout, with only
   declared runtime dependencies, at least on the minimum supported Python.
   Verify every promised extra from the built artifacts in separate environments
   or documented compatible combinations. Import the public API, check installed
   version and `py.typed`, run representative behavior, and exercise CLI entry
   points when present. Do not let `PYTHONPATH`, an editable install, or a
   cached project wheel substitute for building the sdist.
   Keep the smoke assertions standard-library-only apart from imports of the
   installed package and its declared dependencies; test tools must not mask
   missing runtime requirements.
5. Retain those exact verified artifacts. Create the GitHub release with them
   and generated or curated notes linking to the changelog. Publish to PyPI
   only if enabled, preferably with trusted publishing, using the same files.
   Serialize publication per version and define retry behavior without
   overwriting tags or republishing an occupied package version.

Installation from a repository, publication of release artifacts, and
publication to a package index are distinct capabilities. State which are
supported and document any repository/environment setup required for release.

## 8. Acceptance checklist

- [ ] Declare distribution/import names, Python/platform support, dependency floors, license, and profile deviations.
- [ ] Create the `src/` package, public API, typed marker, and any CLI entry points.
- [ ] Define metadata, explicit wheel/sdist contents, dependency groups, interpreter pin, and committed lock.
- [ ] Implement shared check commands; configure Ruff, adopted type checkers, pytest, coverage, and strict docs.
- [ ] Write README, contributor instructions, architecture, changelog, and relevant user docs.
- [ ] Configure ignore rules, LF checkout, and any adopted source-header policy.
- [ ] Install one locked prek hook setup per clone, including commit-message validation when adopted; validate hooks on Linux CI.
- [ ] Verify the completion gate and compatibility tests on the declared interpreters and platforms.
- [ ] Test runtime dependency floors separately from development-tool resolution; define the coverage-gating job.
- [ ] Configure documentation deployment and release verification/publication.
- [ ] Assert archive contents and install both wheel and sdist, including promised extras, independently outside the checkout before release.
