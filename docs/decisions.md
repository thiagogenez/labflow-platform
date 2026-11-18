# Decisions

Settled on 2026-11-18 while planning the reboot. Each line is a decision and its main reason. Reopen a decision explicitly before changing it, and record the change here.

## Repositories and Process

| ID | Decision | Why |
| --- | --- | --- |
| D01 | Delete and recreate the repository; discard the previous domain-model work. | Force-pushing would keep old commits reachable through closed pull requests. |
| D02 | Two repositories: `labflow` (application code, images) and `labflow-platform` (IaC, GitOps, docs). Linked by image-digest pull requests, not Git submodules. | Separate access, deploy without rebuild, no CI loops, a clean deploy history. Submodules couple versions at the Git level. |
| D03 | One pre-commit configuration per repository, identical locally and in CI, with commit-message checks. | The editor, the hook and CI must reject the same things. |
| D04 | Conventional Commits; subject at most 72 characters; unwrapped body paragraphs; no co-author trailers. | Clean, readable history in public portfolio repositories. |
| D05 | The agent writes code and infrastructure drafts; the learner runs commands, writes one small piece per day and owns design calls. | Fifteen days are not enough to type everything by hand; running and breaking it is where the learning is. |
| D06 | README as portfolio: Mermaid diagrams, generated animated SVG with static fallback, honest status table. Docs live in `labflow-platform`. | The platform is the focus; animations stay sharp, small and theme-aware. |
| D07 | Three interview questions per day in English; mock interview on Day 15. | Target: Senior Platform Engineer, SRE and Kubernetes roles, interviewed in English. |

## Application

| ID | Decision | Why |
| --- | --- | --- |
| D10 | Thin but realistic application in Go (order-api, payment-api, inventory, psp-simulator) and Python (shipping-api, email-worker, analytics-worker), plus a static storefront. | Polyglot like real companies; Go on the hot path, Python for workers and data. |
| D11 | One broker adapter per language with RabbitMQ, Kafka and SNS + SQS implementations. | Compare the brokers with the same code; SNS fanout to SQS mirrors the RabbitMQ exchange. |
| D12 | Payments through `psp-simulator`, a black-box stand-in for Stripe: browser tokenization, idempotent charges, signed asynchronous webhooks that can arrive duplicated, late or out of order. | Real integrations work this way; card numbers never reach the backend. |
| D13 | Refunds: full or per-item partial, within 30 days, idempotent, as a choreographed saga. | Real return flows; saga versus two-phase commit is a common interview topic. |
| D14 | Shipping quotes from `shipping-api` with a timeout and a flat-rate fallback. | Exercise timeouts, retries, circuit breaking and graceful degradation. |
| D15 | Database per service on one CloudNativePG cluster; active-passive. | Service ownership without the cost of one cluster per service. Active-active is a separate project. |
| D16 | Valkey for the server-side cart, catalog cache and the gateway's global rate limit. | Concrete use cases; BSD licence; Redis protocol. |
| D17 | No extra NoSQL store; ClickHouse is a stretch goal. | No real use case justifies the memory and time. |

## Platform

| ID | Decision | Why |
| --- | --- | --- |
| D20 | Local datacenter: Lima VM with nested KVM and libvirt, VMs created by Terraform; Multipass is the fallback. | Multipass networking passes only each VM's assigned IP, which breaks keepalived VIPs and MetalLB. |
| D21 | Local: 3 control planes, 3 workers, 2 load balancers. AWS: 1 control plane, 2 workers. | A load balancer in front of a single API server balances nothing; three workers allow one replica per node for quorum tests. |
| D22 | Hybrid: one cluster, EC2 burst workers joined over WireGuard; state stays local. | Teaches node networking, MTU and partitions. Stretching a cluster across a WAN is documented as a production anti-pattern. |
| D23 | Kubernetes 1.35, upgraded to 1.36 with kubeadm during the lab. | KEDA, Cilium and CloudNativePG support up to 1.36; the upgrade is a CKA-style drill. |
| D24 | Calico locally and in hybrid; Cilium on AWS. | Compare iptables/BGP-era and eBPF networking; Flannel cannot enforce NetworkPolicy. |
| D25 | Edge: HAProxy + keepalived, MetalLB, Envoy Gateway, cert-manager with a local CA. | ingress-nginx is retired and archived; Envoy Gateway covers routing, TLS, rate limiting and JWT through the Gateway API. |
| D26 | AWS `LoadBalancer` Services through cloud-provider-aws, shown pending first; a Terraform NLB is the fallback. | Explains who implements a `LoadBalancer` Service on self-managed clusters. |
| D27 | Argo CD on local and hybrid; Flux on AWS; same deploy paths; one controller per cluster. | Learn both tools without two controllers fighting over resources. |
| D28 | RabbitMQ with three nodes and quorum queues; Kafka with three KRaft nodes, replication factor 3, `min.insync.replicas` 2. | Kill one node and keep working; kill two and observe writes stop. |
| D29 | KEDA scales consumers on queue length and consumer lag, compared with a CPU-based HPA. | CPU is the wrong signal for queue workers. |
| D30 | CloudNativePG backups to S3 through the Barman Cloud plugin; point-in-time restore locally; disaster-recovery drill on AWS measuring RPO and RTO. | Replication is not a backup; native backups are deprecated. |

## Security and Operations

| ID | Decision | Why |
| --- | --- | --- |
| D40 | Vault + External Secrets Operator locally; AWS Secrets Manager + External Secrets Operator on AWS; etcd encryption at rest; operator-generated infrastructure credentials; no secrets in Git. | The application manifests stay identical; only the `SecretStore` changes. |
| D41 | Self-hosted IRSA (OIDC discovery in S3) on both clusters; scoped access keys only as a fallback; IMDS hop limit stays 1. | No long-lived cloud keys; avoids leaking the node role to every pod. |
| D42 | Human personas (admin, developer, oncall, auditor) through client certificates and RBAC groups; Dex/OIDC is a stretch goal. Per-service identities in every layer. GitHub Actions reaches AWS through OIDC and opens cross-repository pull requests with a GitHub App token. API audit logging on. | Least privilege for people, workloads and automation. |
| D43 | Multi-arch images on GHCR built on native ARM runners, scanned with Trivy, signed with cosign keyless, deployed by digest. Kyverno enforcement is a stretch goal. | What runs is provably what CI built. |
| D44 | kube-prometheus-stack; three SLOs (checkout success 99.5 %, checkout p95 under 500 ms, email within 60 s) with multi-window burn-rate alerts to Mailpit. Loki and Tempo are stretch goals. | Alert on user impact, not on CPU. |
| D45 | k6 protocol-level user journeys from outside the cluster, plus one or two browser users; baseline, spike, soak, stress and mixed scenarios. | Browser automation does not scale to load testing; traffic must enter through the real edge. |
| D46 | AWS Free plan credits, `eu-central-1`, t4g instances, budget alarm at US$20, short sessions, destroy afterwards. | Stay within credits; AWS is for proof, not for running all day. |
