# Repository Guidelines

## Purpose

LabFlow Platform is a public Platform Engineering/SRE learning lab: a Kubernetes platform for the [LabFlow](https://github.com/thiagogenez/labflow) application in three scenarios (self-managed private, AWS managed services, hybrid), operated through failures and bursts to prepare for Senior Platform Engineer, SRE and Kubernetes interviews. Read [the plan](docs/plan.md) and [the decisions](docs/decisions.md) before changing anything. Decisions recorded there are settled unless the learner reopens them.

## Working Agreement

- When unsure about a decision, do not guess: run the `grilling` skill (rounds of numbered questions, each with a recommended answer) and wait for the learner's answers. Record settled decisions in `docs/decisions.md`.
- Follow [CONTRIBUTING.md](CONTRIBUTING.md); the learner is its maintainer.
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

- AWS: deploy a CloudFormation change set only after the learner approves it; at the end of every session run the plan's session-exit procedure (Ingresses through Git first, then the session stacks; the retained `baseline` stack stays). Pass the learner's IP range at deploy time and never commit it. Keep the budget alerts on. One NAT gateway in total.
- No secrets in Git, logs, metrics or CI artifacts. Prefer workload identity (IRSA, EKS Pod Identity, GitHub OIDC) and use no IAM access keys. Self-managed EC2 instances require IMDSv2 with a hop limit of 1.
- Name the kubeconfig context and namespace explicitly in every `kubectl` and `helm` command.
- Destructive drills (stopping nodes, killing primaries, dropping tables) run only against the lab clusters and are announced before they run.

## Ownership

- Terraform owns private machines and CloudFormation owns AWS resources. Ansible owns private node configuration and kubeadm; CloudFormation user data configures burst nodes. After bootstrap, Argo CD (private cluster) or Flux (EKS) owns everything inside the cluster; never run two GitOps controllers on one cluster.
- Helm installs operators and infrastructure charts; Kustomize (base plus overlays) describes applications. One scaler (KEDA or HPA) per workload.
- Render Helm and Kustomize output and validate it with kubeconform before it reaches a cluster.
- Pin every version: charts, images (by digest for LabFlow images), tools, and GitHub Actions by full commit SHA. Check ARM64 support and current official documentation (Context7) before adopting a component.

## Layout

Create directories when their plan day needs them:

```text
terraform/         libvirt VMs inside the Lima hypervisor VM
cloudformation/    AWS stacks: baseline (retained), network, cluster, data, messaging, hybrid
ansible/           node preparation, kubeadm, CNI, WireGuard
deploy/            Helm values and Kustomize bases/overlays
clusters/private/  Argo CD applications (private and hybrid)
clusters/aws/      Flux kustomizations
docs/              plan, decisions, runbooks/, interview/, assets/
```

## Quality Gates

Add hooks with the first files of each type: `terraform fmt`/`validate` and tflint, cfn-lint and cfn-guard, ansible-lint, `helm lint` and kubeconform.

## Documentation

- The README is the portfolio entry point: short introduction, honest status table, target architecture in Mermaid, scenario table and links. Updating it is part of every day's definition of done.
- Animated illustrations are SVG generated from versioned source in `docs/assets/src/`, labelled "Illustration", shipped with a static alternative, and respect `prefers-reduced-motion`. Generator dependencies are development-only.
- Runbooks live in `docs/runbooks/` (symptom, impact, diagnosis, recovery, prevention). Interview questions and answers live in `docs/interview/`.
