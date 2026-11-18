# Plan

Planning finished on 2026-11-18. Day 1 starts on 2026-11-19. Fifteen working days at about six hours each. The README status table records what is actually done.

## Goal

Build a self-managed Kubernetes platform for the LabFlow order system in three scenarios, run the real application on it, then break it on purpose: load balancer loss, node and quorum failures, traffic bursts, upgrades, disaster recovery. The result is working infrastructure, evidence and runbooks, plus interview-ready explanations for Senior Platform Engineer, SRE and Kubernetes roles.

## Scenarios

**Local** (most days). A Lima VM with nested virtualization acts as the hypervisor host. Inside it, Terraform (libvirt provider) creates eight KVM VMs: two HAProxy/keepalived load balancers, three control planes and three workers. Ansible prepares the nodes and runs kubeadm. Argo CD then installs everything else from this repository. Load generators run on the hypervisor VM, outside the cluster, so traffic enters through the VIP like real traffic. The only cloud dependency is the S3 bucket for database backups.

**AWS** (two or three sessions of two to four hours, destroyed afterwards). Terraform creates a VPC without NAT, three EC2 t4g instances (one control plane, two workers), IAM, the IRSA discovery bucket and the backup bucket. The same Ansible roles run kubeadm, with Cilium as the CNI. Flux installs everything from `clusters/aws`. Brokers and databases run with fewer replicas. Traffic arrives through an NLB that cloud-provider-aws creates.

**Hybrid** (two sessions, destroyed afterwards). The local cluster keeps the control plane and all state. Terraform creates one or two EC2 burst workers and a WireGuard tunnel; Ansible joins them to the local cluster. A `burst` taint admits only stateless consumers, which KEDA scales onto the AWS nodes during a spike. Data never leaves the local side.

## Components

- Edge: HAProxy + keepalived VIP and MetalLB (local), AWS NLB through cloud-provider-aws, Envoy Gateway (Gateway API) as reverse proxy and API gateway, cert-manager with a local CA for `*.labflow.test`.
- Kubernetes: kubeadm 1.35, upgraded to 1.36 during the lab; containerd 2.3 LTS; Ubuntu 24.04 ARM64; Calico (local, hybrid), Cilium (AWS); local-path-provisioner (local), EBS CSI (AWS).
- Data: CloudNativePG (one primary, two replicas, database per service, Barman Cloud plugin backups to S3); Valkey with Sentinel; RabbitMQ (three nodes, quorum queues); Kafka via Strimzi (three KRaft nodes, replication factor 3, `min.insync.replicas` 2); SNS + SQS in one AWS session.
- Security: Vault + External Secrets Operator (local), AWS Secrets Manager + External Secrets Operator (AWS); etcd encryption at rest; API audit logging; RBAC personas through client certificates; per-service identities in Kubernetes, PostgreSQL, Kafka, RabbitMQ and Vault; Pod Security Admission `restricted`; default-deny NetworkPolicies; self-hosted IRSA; GitHub OIDC to AWS.
- Delivery: Argo CD (local, hybrid), Flux (AWS); Helm for operators and infrastructure; Kustomize for applications; image digests promoted by pull request from the application repository.
- Observability and load: kube-prometheus-stack; three SLOs with multi-window burn-rate alerts delivered to Mailpit; k6 protocol-level journeys plus one or two browser users; KEDA on queue length and consumer lag.

## Daily Rhythm

1. Briefing in Portuguese: the concept and how interviews probe it.
2. The agent builds; the learner reviews the diff and owns the design calls.
3. The learner runs the key commands, one at a time.
4. The learner writes one small piece (10–30 lines of IaC, YAML or PromQL).
5. Break it: the day's failure drill, observed in Grafana or the CLI.
6. Close: three interview questions answered in English.

## Definition of Done

A day is done when its exit check was observed (not assumed), the README status table is updated, the day's failure has a runbook in `docs/runbooks/`, the interview answers are in `docs/interview/`, hooks pass, CI is green, and the learner approved the commits.

## Days

| Day | Focus | Exit check | Failure drill |
| --- | --- | --- | --- |
| 1 | Application repository: hooks, CI, all services in Docker Compose, README diagram | A storefront order produces an email in Mailpit, a stock change and an analytics row | Review the event contract |
| 2 | The reel, live: RabbitMQ vs Kafka vs dead-letter queues; webhook chaos; shipping fallback; signed multi-arch images on GHCR; reel animation | The email worker crashes and recovers without losing or duplicating an email on either broker | Worker crash, duplicate delivery |
| 3 | Lima hypervisor with nested KVM (two-hour spike); Terraform libvirt VMs; Ansible base roles | Eight VMs reachable; a VIP answers from a client | Fallback decision if nesting fails |
| 4 | kubeadm 1.35 HA behind the keepalived VIP; Calico; etcd encryption; audit log; VIP failover animation | Six nodes Ready; API reachable through the VIP | Kill a load balancer; stop one, then two control planes |
| 5 | Argo CD bootstrap; MetalLB; Envoy Gateway; cert-manager TLS; local-path | Storefront served over TLS through the gateway | Manual drift self-healed; bad sync rolled back through Git |
| 6 | CloudNativePG, Valkey Sentinel, Strimzi, RabbitMQ; per-service users; application via Kustomize | End-to-end order through the gateway on both brokers | Kill the PostgreSQL primary; kill a broker node |
| 7 | AWS account guardrails, GitHub OIDC role, IRSA discovery bucket; CloudNativePG backups to S3; Vault + External Secrets | Applications read credentials from Vault; a base backup exists in S3 | Point-in-time restore after dropping `orders`; Vault sealed after restart |
| 8 | kube-prometheus-stack; dashboards; SLOs and burn-rate alerts to Mailpit | An alert fires from an induced failure and reaches Mailpit | Induced error budget burn |
| 9 | RBAC personas via CSR; ServiceAccounts; Pod Security Admission; default-deny NetworkPolicies | Each persona can do exactly its job; only declared traffic flows | Developer tries to read a Secret; blocked connection traced |
| 10 | k6 scenarios; KEDA vs CPU HPA; global rate limit; backpressure | Observed scale-out and measured drain time | Black Friday spike |
| 11 | CKA drills: kubeadm upgrade 1.35 to 1.36 under load; etcd snapshot and restore; certificate renewal | Cluster on 1.36 with no failed orders during the upgrade | Node NotReady; kubelet stopped |
| 12 | AWS: Terraform, Ansible, Cilium, cloud-provider-aws, EBS CSI with IRSA, Flux | Order placed through the NLB | `LoadBalancer` Service pending before the cloud provider |
| 13 | AWS: SNS + SQS session; disaster-recovery drill; Calico vs Cilium (Hubble); destroy | RPO and RTO measured from an S3 restore | Loss of the local site |
| 14 | Hybrid: WireGuard, burst workers, KEDA onto AWS nodes, MTU; destroy | Consumers run on EC2 and drain local queues | Tunnel cut (network partition) |
| 15 | Surprise game day; mock interview (45 min system design, 30 min troubleshooting); final system-design illustration and write-up | Every drill has a runbook; README reflects reality | Unannounced failures |

## Cut Order If Late

Dex/OIDC login, Loki and Tempo, ClickHouse, Kyverno image-signature enforcement, dynamic database credentials, k6 browser users. The two hybrid and AWS sessions and the failure drills are not cut.

## Risks

- Nested KVM inside Lima on macOS 27 is untested here. Day 3 starts with a two-hour spike; the fallback is Multipass without a VIP (DNS-based load balancer failover, NodePort instead of MetalLB).
- Days 12 and 13 are heavy. The infrastructure code is prepared beforehand; the sessions apply, observe and destroy.
- The AWS account must exist by Day 7. Free-plan limits on t4g.medium and NLB are unverified; t4g.small is the fallback size.
- Memory: the local scenario needs about 24 GiB of the 36 GiB host. Docker Desktop stays stopped while the lab runs.

## AWS Cost Guardrails

Free-plan credits, `eu-central-1`, t4g instances, no NAT gateway, budget alarm at US$20. Every apply needs explicit approval; every session ends with `terraform destroy`.

## Out of Scope

Active-active databases (candidate for a second project), service mesh, extra NoSQL stores, managed AWS databases, caches and brokers (covered as interview trade-offs only), login, 3-D Secure and chargebacks.
