# Threat model

Short on purpose. One page of who can hurt what, and which control answers each one. Where a control is not
there yet, it says so.

## What is being protected

- The boundary between tenants. team-a and team-b are two customers who do not trust each other and share nodes.
- The control plane: API server, etcd (secrets at rest), the kubelets.
- The platform's own components: Argo CD (can deploy anything), Kyverno (decides what is admitted), the
  monitoring stack (sees everything).
- The nodes themselves, three VMs on public IPs.

## Who the attackers are

| Actor | Starting position |
|---|---|
| A1 Internet | Any address, no credentials. Sees the three public IPs. |
| A2 Compromised tenant workload | Code execution inside a team-a pod. Typical start of a real incident: a vulnerable app or a malicious image. |
| A3 Malicious or careless tenant admin | Holds team-a's service account. Can create workloads in team-a. |
| A4 Supply chain | Controls an image or a chart we pull, or a commit to this repo. |
| A5 Noisy neighbour | Not malicious. A tenant whose workload exhausts something shared. |

## Attack paths and what stops them

| # | Path | Actor | Control | Proof |
|---|---|---|---|---|
| 1 | Reach the API server, kubelet, etcd or VXLAN from the internet | A1 | nftables on every node: default drop; 22 open (key only); 6443 from one admin IP; cluster ports only from the other nodes | demo/05 |
| 2 | Run a privileged or host-mounting pod, escape to the node | A3 | Kyverno PSS restricted, Enforce, on every tenant namespace | demo/01 |
| 3 | Pull an unknown or unpinned image | A3, A4 | Kyverno: tag or digest required, no :latest, registry allow-list | demo/01 |
| 4 | From a team-a pod, reach team-b pods or services | A2 | Cilium clusterwide default deny; per-namespace allow only to itself | demo/02 |
| 5 | From a team-a pod, call home or exfiltrate over the internet | A2 | Egress default deny; team-a is allowed api.github.com:443 only, by FQDN | demo/02 |
| 6 | From a team-a pod, reach the kubelet, the node, or the API server | A2 | Cilium egressDeny to host, remote-node and kube-apiserver entities (deny wins over any allow); no SA token mounted | demo/02, demo/04 |
| 7 | Use team-a's identity against team-b or the cluster | A3 | Namespaced Role only, no RBAC verbs, no cluster-scoped grants | demo/03 |
| 8 | Create enough objects to hurt shared components | A5, A3 | ResourceQuota with pod and deployment counts, LimitRange defaults | demo/03 |
| 9 | Run a known-vulnerable image and nobody notices | A4 | Trivy Operator scans every image; alert when a tenant image carries a critical CVE | demo/06, alert TenantImageHasCriticalCVE |
| 10 | Get a shell in a pod and look around quietly | A2 | Tetragon records every exec with pod, binary and arguments; Hubble records every denied flow | demo/04, demo/02 |
| 11 | Read secrets from an etcd backup or disk | A1 with disk access | EncryptionConfiguration (aescbc) for secrets at rest; key generated on cp-1, not in git | infra/encryption.yaml.example |
| 12 | Push a bad policy or manifest to the cluster | A4 | Everything after the CNI goes through Argo CD from this repo; policies have CLI tests in CI; audit before enforce | .github/workflows/policy-tests.yaml |
| 13 | Tamper with the audit trail | A2 after node compromise | API server audit log on the control plane node with secrets, RBAC, exec and policy changes at RequestResponse level | infra/audit-policy.yaml |

## What is not covered, and I know it

- SSH is open to the internet (key only). The reviewers need a way in and I do not have their IPs. In a real
  deployment this is a bastion or a VPN, and 22 joins the admin allow-list.
- Image signatures are not verified at admission yet. Registry allow-list and tag pinning are a weaker substitute.
  cosign plus Kyverno verifyImages is the next control; see docs/summary.md.
- No mTLS between services. Cilium can do it, but it is still a beta feature and I would rather not promise it.
- Tetragon is observe only. No enforcement policy, no sigkill on escape precursors. First you learn the baseline.
- The audit log sits on the control plane node. If cp-1 is lost, so is the log. It needs shipping off the node.
- Kyverno's webhook is fail-closed everywhere except kube-system. Right for tenants; for platform namespaces it
  means a Kyverno outage blocks the platform's own rollouts. I hit this once (see worklog). Open question.
- Single control plane node. Not a security control, but worth saying.
- The firewall relies on three fixed node IPs. Adding a node means editing a file and re-applying it on every node.
