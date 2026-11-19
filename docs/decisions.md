# Decisions

Settled on 2026-11-18 while planning the reboot. Each line is a decision and its main reason. Reopen a decision explicitly before changing it, and record the change here.

## Repositories and Process

| ID | Decision | Why |
| --- | --- | --- |
| D01 | Delete and recreate the repository; discard the previous domain-model work. | Force-pushing would keep old commits reachable through closed pull requests. |
| D02 | Two repositories: `labflow` (application code, images) and `labflow-platform` (IaC, GitOps, docs). Linked by image-digest pull requests, not Git submodules. | Separate access, deploy without rebuild, no CI loops, a clean deploy history. Submodules couple versions at the Git level. |
| D03 | One pre-commit configuration per repository, identical locally and in CI, with commit-message checks. | The editor, the hook and CI must reject the same things. |
| D04 | Conventional Commits; subject at most 72 characters; unwrapped body paragraphs; no co-author trailers or "Generated with" lines in commits and pull requests. Pull requests merge with a merge commit titled by the pull request; no squash, no rebase. Every commit and pull request references a labflow-platform issue, automated ones such as digest promotion included. Amended 2026-11-19: ticket references and pull request checks. | Clean, readable history in public portfolio repositories; every change traces back to its ticket; signed commits stay intact and each pull request remains one revertible unit. |
| D05 | The agent writes code and infrastructure drafts and, once the learner approves the text, commits, pushes topic branches and opens pull requests; the learner runs the key commands, writes one small piece per day, owns design calls and merges. | Fifteen days are not enough to type everything by hand; running and breaking it is where the learning is. |
| D06 | README as portfolio: Mermaid diagrams, generated animated SVG with static fallback, honest status table. Docs live in `labflow-platform`. | The platform is the focus; animations stay sharp, small and theme-aware. |
| D07 | Three interview questions per day in English; mock interview on Day 15. | Target: Senior Platform Engineer, SRE and Kubernetes roles, interviewed in English. |
| D08 | Every commit passes three independent agent reviews (correctness and duplication; over-engineering; security and operability), two rounds each; findings are compared for consensus and the learner decides. Amended 2026-11-19: review stops when a round raises no high or medium finding, with at most two full rounds per change. | Clean, DRY, right-sized code in public repositories; independent reviewers catch different problems. |
| D09 | When a decision is unclear, the agent runs the grilling skill instead of guessing. | Decisions stay with the learner and get recorded. |
| D18 | labflow-platform holds the single contribution guide and backlog for both repositories; small per-repository checks stay copied until a third consumer justifies a shared action. Settled 2026-11-19. | One place to read rules and file work; the rule of three keeps shared abstractions earned. |

## Application

| ID | Decision | Why |
| --- | --- | --- |
| D10 | Thin but realistic application in Go (order-api, payment-api, inventory, psp-simulator) and Python (shipping-api, email-worker, analytics-worker), plus a static storefront. | Polyglot like real companies; Go on the hot path, Python for workers and data. |
| D11 | One broker adapter per language (RabbitMQ, Kafka, SNS + SQS); one broker per deployment: Kafka and RabbitMQ stressed on the private cluster, native SNS + SQS on AWS. Reopened 2026-11-18: AWS moved from self-hosted brokers to native SNS + SQS. | Measure before choosing; publishing an event to two brokers only adds failure modes. |
| D12 | Payments through `psp-simulator`, a black-box stand-in for Stripe: browser tokenization, idempotent charges, signed asynchronous webhooks that can arrive duplicated, late or out of order. | Real integrations work this way; card numbers never reach the backend. |
| D13 | Refunds: full or per-item partial, within 30 days, idempotent, as a choreographed saga. | Real return flows; saga versus two-phase commit is a common interview topic. |
| D14 | Shipping quotes from `shipping-api` with a timeout and a flat-rate fallback. | Exercise timeouts, retries, circuit breaking and graceful degradation. |
| D15 | Database per service on one CloudNativePG cluster; active-passive. | Service ownership without the cost of one cluster per service. Active-active is a separate project. |
| D16 | Valkey for the server-side cart, catalog cache and the gateway's global rate limit. | Concrete use cases; BSD licence; Redis protocol. |
| D17 | No extra NoSQL store; ClickHouse is a stretch goal. | No real use case justifies the memory and time. |

## Platform

| ID | Decision | Why |
| --- | --- | --- |
| D20 | Private datacenter: Lima VM with nested KVM and libvirt, VMs created by Terraform; Multipass is the fallback. | Multipass networking passes only each VM's assigned IP, which breaks keepalived VIPs and MetalLB. |
| D21 | Private: 3 control planes, 3 workers, 2 load balancers. AWS: see D31. Amended 2026-11-19: AWS moved from kubeadm on EC2 to EKS. | A load balancer in front of a single API server balances nothing; three workers allow one replica per node for quorum tests. |
| D22 | Hybrid: the private cluster stays primary; burst nodes from an EC2 Auto Scaling Group join it over a WireGuard hub, with the Lima hypervisor VM as the private gateway, scaled by Cluster Autoscaler with short-lived join tokens; state stays private. Amended 2026-11-19: real elasticity replaces statically joined workers. | Capacity only while it is needed; teaches node networking, MTU and partitions. Stretching a cluster across a WAN is documented as a production anti-pattern. |
| D23 | Kubernetes 1.35, upgraded to 1.36 with kubeadm during the lab. | KEDA, Cilium and CloudNativePG support up to 1.36; the upgrade is a CKA-style drill. |
| D24 | Calico on the private cluster until Day 11, when a total-loss rebuild switches it to Cilium for the rest of the lab; AWS VPC CNI on EKS. Amended 2026-11-19: Cilium moved from AWS to the private rebuild. | Compare overlay/BGP, eBPF and VPC-native pod networking; Flannel cannot enforce NetworkPolicy. |
| D25 | Private edge: HAProxy + keepalived, MetalLB, Envoy Gateway, cert-manager with a local CA. AWS edge: ALB created by EKS Auto Mode from an Ingress, an ACM certificate for the cheapest available `labflow` domain (self-signed import as the fallback) and an AWS WAF rate-based rule, restricted to the learner's IP; ExternalDNS writes the record into the `baseline` hosted zone. Amended 2026-11-19: AWS edge added. | ingress-nginx is retired and archived; Envoy Gateway covers routing, TLS, rate limiting and JWT through the Gateway API; the AWS side contrasts a self-managed with a managed API edge. |
| D26 | Superseded 2026-11-19 by D31; the pending-`LoadBalancer` lesson moves to Day 5. | |
| D27 | Argo CD on the private cluster (private and hybrid scenarios); Flux on EKS; same deploy paths; one controller per cluster. Amended 2026-11-19: Flux runs on EKS. | Learn both tools without two controllers fighting over resources. |
| D28 | RabbitMQ with three nodes and quorum queues; Kafka with three KRaft nodes, replication factor 3, `min.insync.replicas` 2. | Kill one node and keep working; kill two and observe writes stop. |
| D29 | KEDA scales consumers on queue length and consumer lag, compared with a CPU-based HPA. | CPU is the wrong signal for queue workers. |
| D30 | CloudNativePG backups to S3 through the Barman Cloud plugin; point-in-time restore, and a full restore after the Day 11 total cluster loss; a periodic `pg_dump` to S3 is restored into RDS in the site-loss drill, measuring RPO and RTO. Amended 2026-11-19: physical Barman backups cannot be restored into RDS. | Replication is not a backup; a backup is proven only by a restore; cross-platform recovery needs logical dumps. |
| D31 | AWS scenario on managed services: EKS Auto Mode with a Graviton NodePool, RDS for PostgreSQL, ElastiCache for Valkey, SNS + SQS; edge, secrets, identity and observability in D25, D40, D41 and D44. Settled 2026-11-19. | Practise both stacks: operate the mechanism on the private cluster, buy it on AWS. AWS teams do not run kubeadm on EC2. |
| D32 | CloudFormation YAML for AWS: a retained, termination-protected `baseline` stack (buckets, the IRSA OIDC provider, the hosted zone, the deploy role, budgets; domain registration stays outside CloudFormation) and session stacks (network, cluster, data, messaging, hybrid) deployed through reviewed change sets and deleted after every session, ending with a check that only baseline resources remain; CI only lints them with cfn-lint and cfn-guard. Settled 2026-11-19. | AWS-managed state, change sets, rollback and drift detection; long-lived resources survive sessions; an ephemeral environment must not deploy on merge. |
| D33 | AWS network: two Availability Zones, private subnets and one NAT gateway, documented as a single point of failure. Settled 2026-11-19. | EKS needs two Availability Zones; one NAT gateway per zone is the production answer, one in total is the lab's cost choice. |

## Security and Operations

| ID | Decision | Why |
| --- | --- | --- |
| D40 | Vault + External Secrets Operator on the private cluster; AWS Secrets Manager + External Secrets Operator on EKS through Pod Identity, where the controller's role assumes a per-service role for each SecretStore and can therefore read every secret; etcd encryption at rest; operator-generated infrastructure credentials; no secrets in Git. Amended 2026-11-19: AWS secrets reach EKS through Pod Identity. | The application manifests stay identical; only the `SecretStore` changes. |
| D41 | Self-hosted IRSA (OIDC discovery in S3) on the private cluster; EKS Pod Identity on AWS; no IAM access keys; self-managed EC2 instances require IMDSv2 with a hop limit of 1. Amended 2026-11-19: EKS uses Pod Identity. | No long-lived cloud keys; avoids leaking the node role to every pod. |
| D42 | Human personas (admin, developer, oncall, auditor) through client certificates and RBAC groups; Dex/OIDC is a stretch goal. Per-service identities in every layer. GitHub Actions needs no AWS access today; a future workflow that does uses OIDC with a pinned `sub`. Cross-repository pull requests use a GitHub App token. API audit logging on. Amended 2026-11-19: CI no longer needs an AWS role. | Least privilege for people, workloads and automation. |
| D43 | Multi-arch images on GHCR built on native ARM runners, scanned with Trivy, signed with cosign keyless, deployed by digest; EKS pulls them from GHCR directly. Kyverno enforcement is a stretch goal. Amended 2026-11-19: no ECR pull-through cache, which would need a long-lived GitHub token. | What runs is provably what CI built. |
| D44 | kube-prometheus-stack; three SLOs (checkout success 99.5 %, checkout p95 under 500 ms, email within 60 s) with multi-window burn-rate alerts to Mailpit. On AWS, Amazon Managed Service for Prometheus evaluates the same rules and alerts through SNS, and CloudWatch covers managed services; the private Grafana shows both environments. Loki and Tempo are stretch goals. Amended 2026-11-19: AWS observability added. | Alert on user impact, not on CPU. |
| D45 | k6 protocol-level user journeys from outside the cluster, plus one or two browser users; baseline, spike, soak, stress and mixed scenarios. | Browser automation does not scale to load testing; traffic must enter through the real edge. |
| D46 | AWS Paid plan from an "advanced" sign-up in `eu-central-1`, sign-up credits carried over; root MFA, a one-account AWS Organization with IAM Identity Center and no IAM access keys; budget alerts, Cost Anomaly Detection and a deny action back up a session-exit check; short sessions with session stacks deleted afterwards. Amended 2026-11-19: EKS Auto Mode needs instances outside the Free plan, which also removes its spending cap. | AWS is for proof, not for running all day. |
