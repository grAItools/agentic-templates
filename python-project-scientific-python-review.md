# Python blueprint review against the Scientific Python Development Guide

Reviewed 2026-10-31 against the published guide and the local
[Python blueprint](python-project.md), including its inherited
[harness foundation](harness-agnostic-project.md). Recommendations below are
review judgments, not requirements imposed by the guide.

The comparison below records the baseline before the two-document split
(tracked revision `eebac60`). It is retained as historical rationale; the
[recommendation disposition](#recommendation-disposition) records what the
current blueprints specify. These are documentation decisions, not claims
that an executable template or its CI has been implemented.

This is a manual review of a specification. The repository contains blueprint
documents, not a generated Python project, so missing physical `pyproject.toml`,
`src/`, or workflow files here are not template violations. “Gap” below means
the blueprint does not specify a recommended safeguard; an implementation
could still provide it. No automated compliance score is claimed.

The core structure is already aligned: `src/<package>/`, separate `tests/`
and `docs/`, README, license, contributor instructions, modern `pyproject.toml`
metadata, dependency groups, and typed-package metadata. There is no reason to
reorganize these directories. The guide permits multiple build backends and
manual versioning, so Hatchling and `project.version` are reasonable choices.
Its Python-version guidance also explicitly discourages an upper bound in
`requires-python`; make that prohibition explicit in our §4.
See [simple packaging](https://learn.scientific-python.org/development/guides/packaging-simple/).

| Point | Difference and recommendation |
| --- | --- |
| **1. Pytest defaults — adopt; high priority** | Our §6 specifies pytest and testing principles without defining its configuration. Add an explicit TOML baseline: minimum pytest version, `testpaths`, strict markers/configuration, strict expected failures, warnings as errors with narrow exceptions, failure summaries, and useful failure logs. These catch configuration mistakes and dependency deprecations that coverage cannot catch. Use the syntax appropriate to the selected pytest version. See [testing](https://learn.scientific-python.org/development/guides/pytest/). |
| **2. Repository hygiene — adopt; high priority** | Our §5–6 requires Ruff, type checks, and harness synchronization, but leaves file hygiene unspecified. Add conflict-marker, case-conflict, broken-symlink, oversized-file, YAML, whitespace, and newline checks. These cover files outside Python code, including the foundation's configuration and links. See the guide's [minimal hook suite](https://learn.scientific-python.org/development/guides/style/#pre-commit). |
| **3. Configuration and shell validation — adopt; high priority** | The shared runner does not require schema checks or shell analysis. Add `validate-pyproject`, GitHub workflow schema validation, and ShellCheck for the inherited Bash scripts. A Python type checker cannot validate those scripts or workflow schemas. Apply optional schemas only where their files exist. See [schema validation and ShellCheck](https://learn.scientific-python.org/development/guides/style/). |
| **4. Automated dependency maintenance — adopt; high priority** | Our §7 says to maintain pins through reviewed updates but defines no updater. Add a Dependabot or Renovate configuration for Actions, Python dependencies/lock refreshes, and any external hooks. The guide provides a grouped Actions-update configuration. Require the existing CI checks on update PRs; pinning without regular updates becomes stale. See [CI updating](https://learn.scientific-python.org/development/guides/gha-basic/#updating). |
| **5. CI and publishing security — adopt the missing details; high priority** | Our §7 already calls for restricted permissions and pinned tools. Make action pins explicitly full commit SHAs, add Zizmor workflow analysis, and separate artifact building/testing from the privileged publishing job. Make OIDC trusted publishing the default for enabled PyPI publication, with documented exceptions. The guide recommends these safeguards even though its readable workflow examples use tags. See [security](https://learn.scientific-python.org/development/guides/security/). |
| **6. Packaging checks before release — adopt; high priority** | Our §7 requires strong wheel/sdist verification during release, but ordinary PR CI need only run the completion gate and compatibility tests. Run the existing build, archive checks, and installed-artifact smoke checks on PRs too, and explicitly validate distribution metadata/rendered README, for example with `twine check --strict`. The guide's complete distribution recipe includes PR builds and metadata checking. Reuse our release validation commands so packaging failures appear before tagging. See [pure Python CI/CD](https://learn.scientific-python.org/development/guides/gha-pure/). |
| **7. Full tests against installed distributions — adopt for libraries** | Our §4 excludes tests from the default sdist; §7 requires representative smoke behavior against installed artifacts. The guide emphasizes being able to run the test suite against an installed wheel or downstream package. For reusable libraries, keep the suite independent of editable-source imports, run it against an installed wheel outside the checkout, and normally include tests and fixtures in the sdist for downstream packagers. Keep harness configuration and private tooling excluded. A minimal CLI can retain a smaller sdist with a documented reason. See [installed-package testing](https://learn.scientific-python.org/development/guides/pytest/#tests-should-test-the-installed-version-not-the-local-version). |
| **8. Documentation and spelling checks — adopt selectively** | Our §5–6 formats Python but does not specify prose formatting, spelling, or code-block formatting. Add a spelling checker, a Markdown formatter, and formatting of Python documentation examples through a supported tool. Include maintained hidden-directory documentation deliberately. This complements the existing requirement to execute examples. Avoid adding a second Python formatter. See [style and static checks](https://learn.scientific-python.org/development/guides/style/). |
| **9. Custom task runner instead of Nox — retain our difference** | Our §5 centralizes commands in `scripts/check.py` under uv. The guide recommends Nox, with other established runners accepted, principally for discoverability and isolated environments. Our design already centralizes local/CI commands and specifies separate environments where isolation matters. Do not introduce a second command owner solely for compliance. Require helpful command discovery and visible tool invocations; consider Nox if environment orchestration grows substantially. See [task runners](https://learn.scientific-python.org/development/guides/tasks/). |
| **10. Local hooks instead of published tool hooks — retain our difference** | Our §5 runs project tools through locked local hooks; the guide usually demonstrates published hooks in isolated environments. Keep our approach to preserve one tool version and command definition across local work and CI. Keep the GitHub Actions hook job rather than adopting pre-commit.ci by default: these local hooks depend on uv and the project environment, while that service restricts downloads during checks. See [pre-commit usage and CI limitations](https://learn.scientific-python.org/development/guides/style/#pre-commit). |
| **11. Four blocking type checkers — simplify the default** | Our §2 and §6 require mypy, Pyright, Zuban, and Pyrefly. The guide recommends typed APIs and strong checking, while presenting checkers as alternatives; it does not require four. My recommendation is one primary blocking checker, or two when consumer compatibility justifies it, with the others opt-in or advisory. Four configurations and interacting suppressions impose maintenance costs. Preserve checking of maintained utilities and test bodies when reducing the set. See [typing](https://learn.scientific-python.org/development/guides/typing/) and the [checker alternatives](https://learn.scientific-python.org/development/guides/style/#type-checking). |
| **12. Zensical and documentation hosting — retain the stack; add previews when useful** | Zensical is explicitly supported by the current guide, alongside Sphinx and ProperDocs. Our Markdown, `docs/`, strict builds, and Diátaxis guidance align. Do not switch to Sphinx or add `.readthedocs.yaml` solely for conformity. The guide's Read the Docs setup offers previews and versioned hosting; adopt those capabilities if users need them. Our handwritten API reference is acceptable for a small surface; consider generated reference when drift becomes costly. See [documentation](https://learn.scientific-python.org/development/guides/docs/). |
| **13. Coverage policy — retain our difference** | Our §6 requires branch measurement and a 95% combined gate, normally on minimum-Python Linux. The guide explains coverage thresholds without prescribing 95%, and recommends Codecov or manual combination for multi-job reporting. Keep our explicit policy and conditional cross-platform combination. External coverage hosting is optional; adopt it when PR coverage comparisons justify the service. Do not weaken meaningful tests to optimize the percentage. See [coverage](https://learn.scientific-python.org/development/guides/coverage/). |
| **14. Contributor AI policy — adopt a project-specific policy** | The foundation describes agent configuration thoroughly but does not define contributor disclosure or human responsibility. The guide separately recommends `AI_POLICY.md` and explicit expectations for assisted contributions. Add that file, or equivalent contributor guidance linked from `AGENTS.md`, covering disclosure, responsibility, review, and interaction with maintainers. Choose the project's stance explicitly rather than copying a restrictive sample policy. Keep our shared instructions and harness adapters. See [agentic AI](https://learn.scientific-python.org/development/guides/ai/). |

Other deliberate policies can stay: the Python 3.12 starting minimum, Linux
plus Windows coverage with macOS added when supported, 100-column formatting,
Google-style docstrings, Conventional Commits, and a maintained changelog.
The guide allows configurable styles and support matrices rather than
requiring its examples verbatim. Our committed uv lock, dependency-floor
tests, exact-commit release validation, and reuse of verified artifacts are
valuable safeguards to preserve.

A useful additional check, inferred from the distinction between consumer
requirements and our development lock, is a scheduled test of freshly resolved
runtime dependencies in a disposable environment. Keep test tools controlled
and leave the normal lock unchanged. This complements our existing lowest-direct
tests and detects compatibility failures hidden by an older lock; it is a
review recommendation, not a claimed mandatory repo-review check.

If repo-review is introduced, run it against a representative implemented
project. The CLI needs the Scientific Python plugin (`sp-repo-review`);
`repo-review` itself is only a framework. Its structural checks are useful but
do not prove workflow correctness.
See [repo-review's plugin and configuration documentation](https://repo-review.readthedocs.io/en/latest/).

Expect some tool-specific mismatches: the inspected checker recognizes named
task-runner configurations rather than our arbitrary Python runner, and
recognizes published hook repositories/IDs rather than following local hooks
into `scripts/check.py`. Record narrowly justified ignores for equivalent
behavior; do not duplicate runners or tool installations to obtain a green
score. Conversely, do not suppress genuinely missing hygiene or pytest checks.
Check identifiers and recognition behavior should be verified against the
plugin version adopted. See the upstream
[runner checks](https://github.com/scientific-python/cookie/blob/main/src/sp_repo_review/checks/general.py)
and [hook checks](https://github.com/scientific-python/cookie/blob/main/src/sp_repo_review/checks/precommit.py).

## Recommendation disposition

The [baseline](python-project.md) owns shared technical requirements. The
[community extension](python-project-community.md) adds coordination and
downstream commitments without redefining the baseline's commands.

| Original finding | Disposition in the revised specifications |
| --- | --- |
| 1. Pytest defaults | Baseline: explicit pytest TOML settings, strictness, warnings, and failure reporting. |
| 2. Repository hygiene | Baseline: file hygiene included in the shared lint interface. |
| 3. Configuration/shell checks | Baseline: pyproject/workflow/updater schemas, ShellCheck, and workflow analysis. |
| 4. Dependency maintenance | Baseline: reviewed automated dependency and lock updates. |
| 5. CI/publishing security | Baseline: SHA pins, least privilege, isolated publishing and OIDC. Extension: independent authorization, provenance, shared access and recovery. |
| 6. PR package validation | Baseline: shared package checks required on PRs and integration pushes as well as releases. |
| 7. Installed full-suite testing | Extension: library tests and inputs in sdists and full tests against installed wheels, including wheels rebuilt from sdists. Baseline retains isolated runtime-only smoke checks. |
| 8. Documentation/spelling | Baseline: spelling, Markdown, and verified Python-example formatting. |
| 9. Custom runner | Retained in baseline with discoverable commands and short rationale; no second command owner. |
| 10. Locked local hooks | Retained in baseline with GitHub Actions enforcement and service-compatibility rationale. |
| 11. Fewer type checkers | **Rejected by user decision:** retain mypy, Pyright, Zuban, and Pyrefly as blocking checks with their existing scopes and suppression rules in both profiles. |
| 12. Documentation stack | Baseline retains Zensical. Extension adds maintained-version docs and optional isolated PR previews. |
| 13. Coverage | Baseline retains the 95% combined statement/branch gate and existing combination policy; external hosting stays optional. |
| 14. AI contribution policy | Baseline: explicit policy linked from contributor and agent instructions. Extension assigns contributor/reviewer responsibilities. |

Additional advisor recommendations adopted: optional Conventional Commits,
a support-policy-based Python minimum, optional manual-release orchestration,
baseline security contact/recovery guidance, and an additive community
acceptance checklist. Windows correctness testing and verification of all
promised extras remain baseline safeguards; documented compatible extra
combinations were already allowed. Scheduled fresh-resolution/upstream tests
and explicit SPEC adoption decisions belong to the community extension.
