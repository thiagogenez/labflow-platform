# Repository Guidelines

## Purpose

LabFlow Platform is a public Platform Engineering/SRE learning lab: a self-managed Kubernetes platform for the [LabFlow](https://github.com/thiagogenez/labflow) application in three scenarios (local, AWS, hybrid), operated through failures and bursts to prepare for Senior Platform Engineer, SRE and Kubernetes interviews. Read [the plan](docs/plan.md) and [the decisions](docs/decisions.md) before changing anything. Decisions recorded there are settled unless the learner reopens them.

## Working Agreement

- When unsure about a decision, do not guess: run the `grilling` skill (rounds of numbered questions, each with a recommended answer) and wait for the learner's answers. Record settled decisions in `docs/decisions.md`.
- Work one plan day at a time. For each step: explain the proposal, build it, verify it against the day's exit check, compare expected and observed behaviour, name the limitations, ask the day's interview questions, and wait before advancing.
- Code, documentation and commit messages are in English. Teach in Portuguese. Interview drills (questions and the learner's answers) are in English.
- The agent writes application code and infrastructure drafts. The learner runs the key commands and writes one small piece per day. Give the learner exactly one command per message, say what it should print, and wait for the output. Several read-only checks may go together.
- Prefix agent shell commands with `rtk`; use `rtk proxy` when unfiltered output is needed.
- Keep the tree clean: scratch work goes to the session scratchpad or `/tmp/` and is never committed; do not keep copies of old states, patches or logs in the repository.

## Honesty

- Distinguish implemented, tested locally, tested on AWS, simulated and planned capabilities. Never invent results, traffic, uptime, scale or benchmarks.
- A configured workflow is pending until it runs green on GitHub. Update the README status table when a capability changes state.
- `psp-simulator` and Mailpit stand in for external systems; say so wherever they appear.

## Safety and Cost

- AWS: run `terraform apply` only after the learner approves that specific apply, and `terraform destroy` at the end of every session. Keep the budget alarm on. No NAT gateways.
- No secrets in Git, logs, metrics or CI artifacts. Prefer workload identity (IRSA, GitHub OIDC) to long-lived keys. Keep the EC2 IMDS hop limit at 1.
- Name the kubeconfig context and namespace explicitly in every `kubectl` and `helm` command.
- Destructive drills (stopping nodes, killing primaries, dropping tables) run only against the lab clusters and are announced before they run.

## Ownership

- Terraform owns machines and cloud resources. Ansible owns node configuration and kubeadm. After bootstrap, Argo CD (local and hybrid) or Flux (AWS) owns everything inside the cluster; never run two GitOps controllers on one cluster.
- Helm installs operators and infrastructure charts; Kustomize (base plus overlays) describes applications. One scaler (KEDA or HPA) per workload.
- Render Helm and Kustomize output and validate it with kubeconform before it reaches a cluster.
- Pin every version: charts, images (by digest for LabFlow images), tools, and GitHub Actions by full commit SHA. Check ARM64 support and current official documentation (Context7) before adopting a component.

## Layout

Create directories when their plan day needs them:

```text
terraform/local/   libvirt VMs inside the Lima hypervisor VM
terraform/aws/     VPC, EC2, IAM, S3, SNS, SQS, budget
ansible/           node preparation, kubeadm, CNI, WireGuard
deploy/            Helm values and Kustomize bases/overlays
clusters/local/    Argo CD applications (local and hybrid)
clusters/aws/      Flux kustomizations
docs/              plan, decisions, runbooks/, interview/, assets/
```

## Quality Gates

- Run `make setup` once per clone; run `make check` before asking for a commit. CI runs the same hook configuration on native ARM64.
- Current hooks: file hygiene, gitleaks, actionlint, and commit-message checks. Add with their first files: `terraform fmt`/`validate` and tflint, ansible-lint, `helm lint`, kubeconform, ShellCheck.
- Never bypass hooks (`--no-verify`, `SKIP`, suppressions) to get a commit through; fix the finding.
- Shell scripts use `set -euo pipefail`, pass ShellCheck, and verify checksums of anything they download.

## Commits and Pull Requests

These rules are identical in `labflow` and `labflow-platform`; change them in both.

- Before every commit, documentation included, three fresh agents review the staged diff independently. They get the diff, both repositories' `AGENTS.md`, `labflow-platform/docs/plan.md` and `docs/decisions.md`, and may open any file the change refers to, but never the author's reasoning. Reviewer A checks correctness, consistency, duplication and reinvented wheels (official documentation, Context7); B checks over-engineering and bloat (`ponytail-review`); C checks security and operability (secrets, least privilege, failure handling, timeouts, resource limits, observability, rollback). Each does a second round that re-verifies its own findings and looks for misses. Findings raised by two or more reviewers are consensus. Present every finding to the learner with an apply/reject verdict and reason; a fix that changes logic gets one re-review, then goes to the learner.
- The learner approves each commit message and pull request description. The agent then commits, pushes to a topic branch (never to `main`) and opens the pull request; the learner merges it with a merge commit titled by the pull request (no squash, no rebase). Prefer few, step-sized commits.
- Conventional Commits, subject at most 72 characters. Write each body paragraph as one unwrapped line; GitHub wraps it. Never add `Co-authored-by`, "Generated with" or any other attribution line to commits or pull requests. The commit-msg hooks check the Conventional Commits header, the subject length and `Co-authored-by` trailers, and CI runs them on every pull-request commit; the remaining rules are checked in review.
- Pull requests state what changed, how it was verified, what remains limited, and a Review section summarizing the findings and their resolution.

## Documentation

- The README is the portfolio entry point: short introduction, honest status table, target architecture in Mermaid, scenario table and links. Updating it is part of every day's definition of done.
- Animated illustrations are SVG generated from versioned source in `docs/assets/src/`, labelled "Illustration", shipped with a static alternative, and respect `prefers-reduced-motion`. Generator dependencies are development-only.
- Runbooks live in `docs/runbooks/` (symptom, impact, diagnosis, recovery, prevention). Interview questions and answers live in `docs/interview/`.
