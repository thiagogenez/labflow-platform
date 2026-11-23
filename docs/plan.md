# Plan

Day 0 (2026-11-18 and 2026-11-19) covered the plan, the decisions, both repositories and the contribution workflow. Fifteen days of about six hours start on 2026-11-20, with up to four buffer days, since quality comes before the schedule; days are numbered rather than dated. The README status table records what is actually done.

## Goal

Build and operate the LabFlow order system in three scenarios: a self-managed private platform, the same system on AWS managed services, and a hybrid in which AWS adds elastic capacity to the private platform. Then break it on purpose: load balancer loss, node and quorum failures, total cluster loss, traffic bursts, upgrades, disaster recovery. The result is working infrastructure, evidence and runbooks, plus interview-ready explanations for Senior Platform Engineer, SRE and Kubernetes roles.

## Scenarios

**Private** (most days). A Lima VM with nested virtualization acts as the hypervisor host. Inside it, Terraform (libvirt provider) creates eight KVM VMs: two HAProxy/keepalived load balancers, three control planes and three workers. Ansible prepares the nodes, runs kubeadm and installs the CNI: Calico until Day 11, Cilium after the Day 11 rebuild. Argo CD then installs everything else from this repository. Load generators run on the hypervisor VM, outside the cluster, so traffic enters through the VIP like real traffic. The scenario's only cloud dependencies live in the retained `baseline` stack: the backup buckets and the IRSA discovery document.

**AWS** (short sessions). Session stacks create a two-zone VPC with one NAT gateway, an EKS Auto Mode cluster on Graviton nodes, RDS for PostgreSQL, ElastiCache for Valkey, SNS + SQS, an ACM certificate and a WAF web ACL; the application's Ingress makes Auto Mode create the ALB, and ExternalDNS writes its record into the domain's hosted zone. Flux installs the application from `clusters/aws`, and workloads reach AWS APIs through EKS Pod Identity. Session stacks are deleted at the end of every session; a retained, termination-protected `baseline` stack keeps what every day uses: buckets, the IRSA OIDC provider, the hosted zone, the CloudFormation deploy role and budgets.

**Hybrid** (short sessions). The private cluster stays primary and keeps all state. A session stack adds a WireGuard hub (an EC2 instance with a fixed public IP that also provides NAT for the burst subnet, so the hybrid scenario needs no NAT gateway) and an EC2 Auto Scaling Group of burst nodes in a private subnet; the Lima hypervisor VM, configured by Ansible, connects outbound to the hub as the private side's WireGuard gateway. When KEDA creates more consumer pods than the private workers can hold, Cluster Autoscaler raises the Auto Scaling Group, and new instances join the private cluster with `kubeadm join` from user data, using short-lived bootstrap tokens that a CronJob rotates into SSM Parameter Store. A `burst` taint admits only stateless consumers, and the group scales back to zero when idle.

## Components

| | Private | AWS |
| --- | --- | --- |
| Provisioning | Terraform (libvirt) and Ansible | CloudFormation YAML: a retained `baseline` stack and session stacks |
| Kubernetes | kubeadm 1.35, upgraded to 1.36; containerd 2.3 LTS; Ubuntu 24.04 ARM64; etcd encryption and API audit logging | EKS Auto Mode with a Graviton NodePool capped by `limits`; public endpoint limited to the learner's IP, private endpoint enabled |
| Network and storage | Calico until Day 11, then Cilium; local-path-provisioner | VPC CNI; two Availability Zones; one NAT gateway |
| Edge | HAProxy + keepalived VIP, MetalLB, Envoy Gateway (Gateway API), cert-manager with a local CA for `*.labflow.test` | ALB created by Auto Mode from an Ingress; ACM certificate for the cheapest available `labflow` domain (self-signed import as the fallback); AWS WAF rate-based rule; ExternalDNS writes the ALB record; restricted to the learner's IP |
| Data | CloudNativePG (one primary, two replicas, database per service; Barman Cloud plugin backups and a periodic `pg_dump` to S3); Valkey with Sentinel | RDS for PostgreSQL (db.t4g.micro, Multi-AZ during the failover drill); ElastiCache for Valkey |
| Messaging, one broker per deployment | Kafka via Strimzi (three KRaft nodes, replication factor 3, `min.insync.replicas` 2), then RabbitMQ (three nodes, quorum queues), stressed with the same scenarios, equivalent durability (Kafka `acks=all`, RabbitMQ publisher confirms), repeated runs and the hardware recorded; both operators stay installed, each watching only its broker's namespace | SNS + SQS, judged on semantics, limits and cost, not against private numbers |
| Secrets and identity | Vault + External Secrets Operator; self-hosted IRSA for AWS access from the private cluster; Istio ambient mTLS between services | Secrets Manager + External Secrets Operator, whose Pod Identity role assumes a per-service role for each SecretStore (so the controller can read every secret); EKS Pod Identity for workloads |
| Delivery | Argo CD (private and hybrid) | Flux |
| Observability | kube-prometheus-stack; the private Grafana also shows the AWS data | Amazon Managed Service for Prometheus with the managed collector and the same PromQL rules, alerts through SNS; CloudWatch metrics, logs and alarms for managed services |

Across both: signed images pulled from GHCR by digest; three SLOs with multi-window burn-rate alerts; RBAC personas, per-service identities, Pod Security Admission `restricted` and default-deny NetworkPolicies; GitHub OIDC to AWS if a workflow ever needs it (D42); k6 protocol-level journeys plus one or two browser users; KEDA on queue length and consumer lag.

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
| 1 | Application repository: language hooks and CI jobs, all services in Docker Compose, README diagram | A storefront order produces an email in Mailpit, a stock change and an analytics row | Review the event contract |
| 2 | The reel, live: RabbitMQ vs Kafka vs dead-letter queues; webhook chaos; shipping fallback; signed multi-arch images on GHCR; reel animation | On either broker, the email worker recovers from a crash and delivers the email; retries reuse the `Message-ID`, the metric counts resend attempts, and Mailpit shows how many copies arrived | Worker crash, duplicate delivery |
| 3 | Lima hypervisor with nested KVM (two-hour spike); Terraform libvirt VMs; Ansible base roles | Eight VMs reachable; a VIP answers from a client | Fallback decision if nesting fails |
| 4 | kubeadm 1.35 HA behind the keepalived VIP; Calico; etcd encryption; audit log; VIP failover animation | Six nodes Ready; API reachable through the VIP | Kill a load balancer; stop one, then two control planes |
| 5 | Argo CD bootstrap; a `LoadBalancer` Service left pending, then MetalLB; Envoy Gateway; cert-manager TLS; local-path | Storefront served over TLS through the gateway | Pending `LoadBalancer` explained; manual drift self-healed; bad sync rolled back through Git |
| 6 | CloudNativePG, Valkey Sentinel, Strimzi, RabbitMQ; per-service users; application via Kustomize; broker cutover runbook (drain, switch `BROKER` in one ConfigMap, verify, remove old volumes) | End-to-end order through the gateway on Kafka, then on RabbitMQ | Kill the PostgreSQL primary; kill a broker node |
| 7 | AWS account and cost guardrails (below), with a one-account AWS Organization and IAM Identity Center; EC2 managed-resource visibility on; `baseline` stack (CloudFormation deploy role, IRSA discovery, hosted zone, encrypted non-public backup buckets with retention and separate write and restore roles); CloudNativePG backups and a periodic `pg_dump` to S3; Vault + External Secrets with scheduled snapshots; `sa.key`/`sa.pub`, the cluster CA and Vault snapshots copied outside the cluster, Vault unseal keys in the learner's password manager; register the cheapest available `labflow` domain on a TLD with WHOIS privacy | Applications read credentials from Vault; a base backup and a dump exist in S3 | Point-in-time restore after dropping `orders`; Vault sealed after restart |
| 8 | kube-prometheus-stack; dashboards; SLOs and burn-rate alerts to Mailpit; Tempo, kept if it fits once reservations follow measured usage; an alert on every non-empty dead-letter queue | An alert fires from an induced failure and reaches Mailpit | Induced error budget burn |
| 9 | RBAC personas via CSR; ServiceAccounts; Pod Security Admission; default-deny NetworkPolicies that allow HBONE (15008) and the ambient probe address; Istio ambient mTLS with AuthorizationPolicy allow-lists, Istio reservations set from measured usage | Each persona can do exactly its job; only declared traffic flows, between mutually authenticated services, while the gateway, Prometheus, KEDA and the operators keep working | Developer tries to read a Secret; blocked connection traced; a call from an undeclared workload is denied |
| 10 | k6 scenarios on Kafka, then RabbitMQ; KEDA vs CPU HPA; global rate limit; backpressure | Observed scale-out and measured throughput, latency and drain time for each broker | Black Friday spike; SMTP outage |
| 11 | kubeadm upgrade 1.35 to 1.36 under load. Then the cluster is declared lost: rebuild it from the repository with Cilium, putting `sa.key`/`sa.pub` and the CA back before `kubeadm init` so IRSA keeps working, restore PostgreSQL from the Barman backup and Vault from its snapshot, and repeat the NetworkPolicy and mTLS tests (Hubble, kube-proxy replacement) | No failed orders during the upgrade; the rebuilt cluster serves the restored data; the same policies hold on Cilium; RTO and RPO measured | Node NotReady; kubelet stopped; total cluster loss |
| 12 | AWS session stacks: network; EKS Auto Mode with a Graviton NodePool; RDS (`DeletionPolicy: Delete`); ElastiCache; SNS + SQS with dead-letter queues and raw message delivery; the EKS network policy controller enabled; ACM certificate and WAF web ACL with its limit above the Day 13 k6 rate. Pod Identity; Secrets Manager + External Secrets; ExternalDNS; Flux; NodePool `limits` and KEDA `maxReplicaCount` as compute ceilings; the Ingress creates the ALB | Order placed through the ALB over TLS; a default-deny NetworkPolicy blocks an undeclared call | WAF throttles a burst |
| 13 | AWS: k6 scenarios on SNS + SQS with capped rate and duration; Managed Prometheus and CloudWatch alerting through SNS, with the private Grafana showing both environments; RDS Multi-AZ failover; disaster recovery by restoring the latest `pg_dump` into RDS and moving traffic to AWS; delete the session stacks | SNS + SQS judged on semantics, limits and cost; RPO and RTO measured | Loss of the private site; RDS failover |
| 14 | Hybrid: WireGuard hub accepting only UDP 51820, with no SSH, also NATing the burst subnet; burst Auto Scaling Group with IMDSv2 required, kubelet `--provider-id`, node-template taint tags and MaxSize as the cost ceiling; the Lima hypervisor VM as the private WireGuard gateway, forwarding only cluster ports (Ansible); join tokens rotated by a CronJob with self-hosted IRSA, only during hybrid sessions, into a tagged SSM SecureString that only the burst instance role can read, with a pinned CA hash; Cluster Autoscaler scoped to that group; KEDA onto AWS nodes over Cilium; MTU; the broker that did better on Day 10 (Kafka topics need at least as many partitions as consumers); drain the burst nodes, then delete the session stacks | Pending consumers trigger scale-out; EC2 nodes join, drain the private queues and scale back to zero | Tunnel cut (network partition) |
| 15 | Surprise game day, including an etcd restore, expired certificates and a break-glass drill (normal cluster access unavailable, recovery with the kubeadm `super-admin.conf`); mock interview (45 min system design, 30 min troubleshooting); lab messaging decision record from the broker measurements; final system-design illustration and write-up | Every drill has a runbook; README reflects reality | Unannounced failures |

## Cut Order If Late

Dex/OIDC login, Istio ambient mTLS (NetworkPolicies stay), Loki and Tempo, ClickHouse, Kyverno image-signature enforcement, dynamic database credentials, k6 browser users, the switch to Cilium (the Day 11 rebuild keeps Calico). The AWS and hybrid sessions and the failure drills are not cut.

## Risks

- Nested KVM inside Lima on macOS 27 is untested here. Day 3 starts with a two-hour spike; the fallback is Multipass without a VIP (DNS-based load balancer failover, NodePort instead of MetalLB).
- Days 11 to 14 are heavy. The agent drafts the CloudFormation templates and rebuild automation before they are needed, and the learner reviews them at the start of each session; the sessions deploy, observe and delete.
- The hybrid automated join is the most complex piece. Its fallback is burst nodes joined by Ansible without the Auto Scaling Group.
- The AWS account must exist by Day 7. EKS Auto Mode's built-in general-purpose NodePool is amd64-only, so ARM workloads need a custom NodePool.
- Memory: the private scenario needs about 24 GiB of the 36 GiB host before Tempo and Istio ambient, whose reservations follow measured usage; if memory is still short, Tempo goes first, once the learner has been told and agrees (D44). Docker Desktop stays stopped while the lab runs.
- Istio ambient must keep working after the Day 11 switch to Cilium, which needs `cni.exclusive=false`, `socketLB.hostNamespaceOnly=true` with kube-proxy replacement and no BPF masquerading, and on hybrid burst nodes, where ztunnel and istio-cni must be ready before consumer pods start.

## AWS Cost Guardrails

- Account: Paid plan from an "advanced" sign-up in `eu-central-1`, sign-up credits carried over; root MFA; a one-account AWS Organization with IAM Identity Center, so the CLI uses short-lived credentials and no IAM access keys exist; every change set runs through the `baseline` CloudFormation deploy role (`--role-arn`). The learner's IP range is passed at deploy time and never committed.
- Alerts: budget alerts on actual spend at 25, 50 and 100 % of US$20 and on forecast spend, Cost Anomaly Detection, and a budget action that, at 100 %, attaches a deny on create and run actions (never on deletes) to the deploy role. Budget data lags by hours, so these back up, never replace, the session-exit check.
- Session exit: remove the Ingresses and volumes through Git so Flux does not recreate them, then delete the session stacks; a Resource Explorer search for resources without the `baseline` tag and direct checks of each billable type (EC2 including Auto Mode instances, load balancers, NAT gateways, public IPv4 addresses, volumes, RDS, ElastiCache, Managed Prometheus scrapers, the join-token parameter) must come back empty; Cost Explorer is checked after each session.
- Estimate, not measured: about US$0.35 per hour for the AWS scenario (EKS, two t4g.medium nodes with the Auto Mode fee, NAT gateway, ALB, WAF, RDS and ElastiCache micro instances, the managed Prometheus collector), plus US$0.005 per hour for each public IPv4 address and double the RDS cost during the Multi-AZ drill; the hybrid hub adds one small instance and its public IP. NodePool `limits`, KEDA `maxReplicaCount` and the burst group's MaxSize are the compute ceilings.
- Every change set needs explicit approval; load tests against pay-per-request services (SNS, SQS) run with capped rate and duration.

## Out of Scope

Role-based messaging (Kafka for events plus a task queue for emails and webhooks); active-active databases (candidate for a second project); service mesh traffic management (retries, canaries, fault injection) and mesh on EKS; extra NoSQL stores; Amazon MQ, MSK, Amazon Managed Grafana and an ECR pull-through cache (covered as interview trade-offs only); AWS CDK; login, 3-D Secure and chargebacks.
