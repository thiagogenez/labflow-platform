# Contributing

LabFlow lives in two repositories: [labflow](https://github.com/thiagogenez/labflow) (application) and [labflow-platform](https://github.com/thiagogenez/labflow-platform) (platform, documentation and the shared backlog). This guide covers both; repository-specific coding rules live in each `AGENTS.md`.

## Tickets

Every change starts from an issue in [labflow-platform](https://github.com/thiagogenez/labflow-platform/issues/new/choose), opened with one of its forms: feature, chore, bug, incident or spike.

- Every commit you write contains a `Refs: thiagogenez/labflow-platform#<number>` line.
- Every pull request description names at least one issue. A closing keyword (`Closes`, `Fixes`, `Resolves`), in the description or a commit message, needs an open issue and closes it on merge; `Refs` may name an open or closed issue and leaves it unchanged. Always write the full `thiagogenez/labflow-platform#<number>`.

## Before Each Commit

- Run `make setup` once per clone. `make check` passes, and CI repeats it on native ARM64. Never bypass hooks (`--no-verify`, `SKIP`, suppressions); fix the finding.
- Three fresh agents review the staged diff independently. They get the diff, both repositories' `AGENTS.md`, this guide, `docs/plan.md` and `docs/decisions.md`, and may open any file the change refers to, but never the author's reasoning. Reviewer A checks correctness, consistency, duplication and reinvented wheels (official documentation, Context7); B checks over-engineering and bloat (`ponytail-review`); C checks security and operability (secrets, least privilege, failure handling, timeouts, resource limits, observability, rollback). Each does a second round that re-verifies its own findings and looks for misses. Findings raised by two or more reviewers are consensus. Every finding goes to the maintainer with an apply/reject verdict and reason. An applied high or medium code finding comes with a regression test, confirmed by mutation: undo the fix or seed the bug, and watch the test fail. The pull request's Review section keeps the per-reviewer ledger its template describes. Review stops when a round raises no high or medium finding: low findings are applied or filed as issues, and any logic change among them gets one focused re-review. A change gets at most two full rounds; whatever remains becomes issues, unless the maintainer asks for another round.
- Shell scripts use `set -euo pipefail` and verify the checksum of anything they download.

## Commits

Conventional Commits, subject at most 72 bytes (72 characters of plain English). Write each body paragraph as one unwrapped line; GitHub wraps it. Never add `Co-authored-by`, "Generated with" or any other attribution line to commits or pull requests. Hooks catch these trailers and AI-assistant lines; review checks wrapping and the rest. Squash `fixup!` commits before review.

## Pull Requests

- Fill in the template; CI checks its title, sections and issue references.
- Both repositories run one gate, kept in labflow-platform: the reusable workflow `.github/workflows/quality-gate.yml`, which runs the calling repository's `make setup`, `make check` and hooks, and the composite action `.github/actions/check-pull-request`. labflow-platform runs the pull request's own copy. labflow pins both to the merge commit of the labflow-platform pull request that made the change, as `gh pr view <number> --repo thiagogenez/labflow-platform --json mergeCommit --jq .mergeCommit.oid` prints it. A gate change references its issue with `Refs`, and the labflow pull request that moves the pin closes it.
- A change to the shared workflow or action, to the pin, or to a repository's `ci.yml` and `pull-request.yml`, `setup` and `check` targets, hooks, `.gitleaks.toml`, `.gitleaksignore` or `.gitattributes` is a change to the gate and is reviewed as one.
- Required checks in `protect-main`: `shared / quality gate` and `pull request metadata` in both repositories, and `go tests` in labflow. A change that renames or adds a check runs once on its pull request; then the maintainer updates the ruleset, reads it back with `gh api repos/thiagogenez/<repository>/rulesets/<id> --jq '{enforcement, types: [.rules[].type], checks: [.rules[] | select(.type == "required_status_checks").parameters], bypass: .bypass_actors}'` (enforcement `active`; types `deletion`, `non_fast_forward`, `pull_request` and `required_status_checks`; every check from integration 15368, strict `true`; bypass `[]`), and merges.
- The maintainer approves each commit message and pull request description; then the agent commits, pushes to a topic branch (never to `main`) and opens the pull request. Prefer few, step-sized commits.
- A secret that reaches `main` fails every later history scan, since `main` cannot be rewritten: revoke or rotate it at once, remove any copy left in the tree, then add the fingerprints gitleaks reports for it to `.gitleaksignore`, the one suppression this guide allows.
- The pull request template and the shared hooks of `.pre-commit-config.yaml` stay copied, identical in both repositories, and change together; each repository appends only the hooks for its own files.
- `main` accepts changes only through pull requests whose checks pass. The maintainer merges with a merge commit titled by the pull request: no squash, no rebase.
- To revert a merged change, run `git revert -m 1 <merge commit>` on a topic branch, change the subject to `revert: <short summary>` (git's "This reverts commit" line keeps the link), add a `Refs:` line naming the incident issue, and open a pull request from the template. GitHub's Revert button produces neither and fails the checks. If the reverted change added, removed or renamed a required check, update `protect-main` before merging the revert and record both edits in the incident issue. A labflow-platform revert reaches labflow only when labflow moves its pins back.
- Break-glass: if the checks themselves block an urgent fix, the maintainer temporarily adds the repository admin role to `protect-main` as a "pull requests only" bypass actor, merges, removes it immediately and records both actions in the incident issue, including the output of the ruleset read-back above, whose `bypass` must be `[]`. The deletion and force-push protections stay on throughout.
