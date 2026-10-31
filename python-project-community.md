# Python Project Community Extension

This document extends the [Python Project Repository Blueprint](python-project.md),
which in turn uses the [Harness-Agnostic Project Blueprint](harness-agnostic-project.md).
Use the baseline for technical configuration and commands; this extension only
adds or tightens requirements for shared maintenance and downstream users.
The baseline's four blocking type checkers, their scopes, and suppression rules
apply unchanged. This is a specification, not a scaffold or generator.

Adopt this extension when regular external contributions, multiple maintainers
with merge/release rights, downstream packages or redistributors, or ecosystem
commitments require explicit coordination. A small scientific library can
need these processes; a non-scientific community library can need them too.
Scale responsibilities to the actual team and publish any unmet requirements
with an owner and adoption plan rather than claiming full conformance.

## 1. Additions to the baseline

Each requirement has one owner. Link to the baseline instead of copying tool
settings, scripts, command definitions, or release-validation steps. Where this
document tightens a baseline default, the following table identifies the change.

| Baseline area | Additional community requirement |
| --- | --- |
| [Profile decisions](python-project.md#2-default-profile-and-project-decisions) | Maintainer roles, review responsibilities, and published policy decisions |
| [Package contents](python-project.md#metadata-and-distribution-contents) | Library tests, fixtures, and test helpers in the sdist for downstream use |
| [Compatibility](python-project.md#environments-and-compatibility) | Support windows, deprecation commitments, and upstream/downstream testing |
| [Tests](python-project.md#tests-and-coverage) | Full tests against installed distributions in addition to smoke checks |
| [Contributor policy](python-project.md#documentation-and-contributor-policy) | Governance, conduct and review rules; maintained API reference and versioned docs |
| [CI](python-project.md#continuous-integration-and-maintenance) | Protected integration branches, required reviews, scheduled compatibility jobs |
| [Releases](python-project.md#release-and-artifact-validation) | Independent authorization, provenance, shared access, and documented recovery |

Add `GOVERNANCE.md`, `CODE_OF_CONDUCT.md`, `.github/CODEOWNERS`, issue/PR
templates, and `docs/development/` pages for compatibility, releases, and
project operations. Existing baseline documents can link to those pages;
keep one authoritative home for each policy. Never store recovery codes,
credentials, or private incident details in these public documents.

## 2. Governance and contributions

Document how maintainers are appointed, how decisions are made, how disputes
are escalated, and how responsibilities are transferred. Identify release,
security, review, and conduct contacts. Several roles can belong to the same
person, while independent release authorization requires another maintainer.
Keep conduct reports separate from public technical decision records.

Use contribution templates to request reproduction steps, relevant testing,
compatibility impact, and release-note information. Link the baseline AI policy
and explain who is responsible for assisted submissions and reviewer responses.
Record significant design and compatibility decisions in issues or short design
notes; ordinary fixes do not need a formal proposal.

Protect integration and maintained release branches with required CI checks
and at least one independent review. Use CODEOWNERS for maintained areas and
release/CI configuration, with a backup owner for critical components. Configure
review requirements explicitly: a CODEOWNERS file alone does not enforce them.
Document merge strategy and emergency fixes, including who can authorize an
exception and the required retrospective review. Conventional Commits remain
a project choice; do not duplicate a manually maintained changelog unnecessarily.

## 3. Support and compatibility

Publish Python, platform, and dependency support windows, the release lines
receiving fixes, and the process for changing them. For scientific ecosystem
projects, use the SPEC 0 policy described below; otherwise state a policy
suited to the consumers. Review support decisions at least quarterly and when
preparing releases. Update metadata, CI, contributor instructions, and user
documentation together. The baseline's development interpreter remains tied
to the actual minimum rather than a permanently fixed Python version.

Specify the warning, documentation, and migration period for deprecated public
APIs, plus the release in which removal may occur. Announce supported-version
drops and breaking changes early enough for downstream users to respond.
Document exceptions for urgent security or correctness fixes.

Keep the baseline's locked CI and minimum-dependency tests. Add scheduled tests
with freshly resolved runtime dependencies, and prerelease/development builds
of important upstream dependencies where available. Prepare disposable
environments with controlled test tooling and the intended runtime versions;
invoke the shared test runner using that environment's interpreter without
resynchronizing it to the ordinary project lock. Record what was installed.

Give scheduled failures an owner and a notification path, such as a deduplicated
issue. Keep experimental upstream jobs separate from required PR checks, but
require investigation and resolution of known failures affecting a release.
Maintain representative downstream integration tests or coordinate testing with
consumers when public API changes can break them. Add platforms and compiled
wheel matrices according to the published support promise; compiled projects
still require the baseline's additional native-build design.

## 4. Distributions for downstream packagers

For libraries, extend the baseline sdist allow-list to include maintained tests,
fixtures, and required test helpers. Include other necessary test inputs
deliberately, keeping harness settings and private tooling excluded. Declare
test dependencies in a dedicated dependency group that downstream packagers
can install without the full contributor toolchain.

Keep the suite independent of editable installs and checkout-relative runtime
resources. Run the full applicable suite against the installed wheel and a
wheel rebuilt from the sdist in fresh environments outside the checkout,
using tests from the unpacked sdist. Assert that package imports resolve to the
installed distribution. These runs supplement the baseline smoke checks with
only runtime dependencies; test tools must not replace those smoke checks.
Document separately provisioned services or slow integrations and how packagers
run the meaningful offline subset. Add this validation to required PR and
release jobs by reusing the baseline's package validation entry point.

## 5. Releases, access, and continuity

Document release roles, steps, required permissions, support-line selection,
rollback limitations, and incident response in the contributor documentation.
Keep release notes and migration guidance useful to downstream maintainers.
An intentional release trigger identifies the exact verified commit and
artifacts; all baseline validation and publishing protections still apply.

Require authorization by at least one other release maintainer before publishing
to an index or public release channel. Use a protected publishing environment
with self-approval disabled and allowed release refs restricted. Confirm that
the hosting plan and settings enforce this requirement; if unavailable, use a
documented independently recorded approval before an authorized publisher
runs the release. Reviewers approve the identified commit and artifacts; rebuilds
or changed artifacts require new validation and authorization.

Restrict permitted release actions and reusable workflows. Generate provenance
attestations for release artifacts and document how consumers verify them,
including the expected repository/workflow identity. Verify attestations before
publishing when artifacts cross workflow boundaries. Distinguish publisher
attestations from claims of fully reproducible builds. Recommend signed release
commits and document the signing and verification process if adopted.
These measures follow [SPEC 8](https://scientific-python.org/specs/spec-0008/).

Inventory restricted resources, their owners, and access/recovery procedures.
Ensure at least two maintainers can recover each critical service, including
source hosting, package publication, documentation, and relevant domains.
Grant individual least-privilege access; shared secrets, where unavoidable,
belong in an encrypted access-controlled vault. Review access at least annually
and on role changes; remove departing maintainers and rotate secrets they could
have retained. Test a release/access handover and document succession when
maintainers become unavailable. See [SPEC 6](https://scientific-python.org/specs/spec-0006/).

## 6. Documentation and repository review

Maintain a complete public API reference, support policy, migration guidance,
and contribution/release instructions. Use generated API extraction when it
reduces drift and a verified integration supports the chosen engine. Keep the
baseline's Zensical setup unless another engine meets a demonstrated need.

Publish versioned documentation for maintained release lines and retain access
to older release docs. Add PR previews where they help review; preview jobs
must not receive production deployment credentials while executing PR content.
Read the Docs and external coverage reporting are optional service choices.

Run pinned `sp-repo-review` and relevant pyproject-schema checks against the
implemented repository in CI. Assign narrow documented ignores for equivalent
behavior, such as the baseline's custom runner and locked local hooks, and
for inapplicable services. Do not duplicate commands or tools for recognition
by the checker. Review ignores as the blueprint and checker evolve; a green
report is supporting evidence, not proof that release/access policies work.
See [repo-review](https://repo-review.readthedocs.io/en/latest/).

## 7. Scientific Python ecosystem commitments

The [Development Guide](https://learn.scientific-python.org/development/)
informs repository engineering; [SPECs](https://scientific-python.org/specs/)
coordinate particular ecosystem practices. Record applicability and adoption
for each relevant SPEC, the source revision/date reviewed, exceptions, and
the responsible maintainer. Do not claim universal SPEC compliance.

| SPEC | Policy for participating scientific projects |
| --- | --- |
| [0: minimum dependencies](https://scientific-python.org/specs/spec-0000/) | Adopt its rolling support policy: three years for Python feature releases and two for core dependency feature releases. Use its current drop schedule when updating floors, recording justified exceptions. |
| [4: nightly wheels](https://scientific-python.org/specs/spec-0004/) | Test available upstream nightly wheels at least weekly and notify maintainers on failure. Publish development wheels at least weekly for downstream testing when relevant, with distinct development versions and retention rules. Document omissions before claiming full adoption. |
| [6: resource access](https://scientific-python.org/specs/spec-0006/) | Apply the access and continuity requirements above. |
| [8: secure releases](https://scientific-python.org/specs/spec-0008/) | Apply the release authorization and provenance requirements above; audit actual service configuration before claiming adoption. |
| [5: CI](https://scientific-python.org/specs/spec-0005/) and [9: governance](https://scientific-python.org/specs/spec-0009/) | Drafts as checked on 2026-10-31; informative sources. Our CI and governance requirements are explicit project policies, not claims that these drafts are final standards. |

Evaluate other SPECs individually when their subject applies. Check current
status before adopting a SPEC or adding a badge. Generic community projects
keep explicit support and governance policies without having to adopt the
Scientific Python ecosystem's dependency windows or nightly infrastructure.

## 8. Additional acceptance checklist

Complete the baseline checklist, then verify these additions:

- [ ] Publish roles, conduct/decision rules, contribution templates, and policy owners.
- [ ] Configure branch protections, required CI/reviews, CODEOWNERS, and emergency procedures.
- [ ] Publish support/deprecation windows and schedule owned compatibility testing.
- [ ] Ship library tests and inputs in sdists; run the applicable full suite against installed distributions.
- [ ] Verify independent release authorization, provenance, and artifact handover.
- [ ] Inventory access, establish backup maintainers, and exercise recovery/succession.
- [ ] Publish maintained-version documentation and verify any preview isolation.
- [ ] Run repository review with justified ignores; record SPEC applicability, status, and exceptions.
