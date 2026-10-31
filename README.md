# project-blueprints

Recommended project blueprints for modern software development processes leveraging coding agents.

## Contents

- [`harness-agnostic-project.md`](harness-agnostic-project.md): Standalone foundation for sharing instructions, skills, and tool configuration across supported AI coding harnesses, for both new and existing project repositories.
- [`python-project.md`](python-project.md): Shared Python baseline for solo maintainers and small teams, with strong automated checks and little coordination overhead.
- [`python-project-community.md`](python-project-community.md): Extends the Python baseline with contributor governance, compatibility commitments, downstream testing, and shared release/access processes; includes applicable Scientific Python SPEC guidance.
- [`python-project-scientific-python-review.md`](python-project-scientific-python-review.md): Dated comparison with the Scientific Python Development Guide and the disposition of its recommendations.

Use the foundation, then the Python baseline. Add the community extension when
external contributions, shared maintainership, downstream consumers, or ecosystem
commitments need formal coordination. Tool configuration and commands are owned
by the baseline and inherited by the extension, including all four blocking
type checkers.

## Credits

`harness-agnostic-project.md` is a fork of [the original gist by davidgibsonp](https://gist.github.com/davidgibsonp/337be9b80b3f03eccd188235c287bb05).
