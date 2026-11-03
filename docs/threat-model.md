# Threat model

Who can hurt what, and which control answers each one. Where there is no control yet, it says so.

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
| 3 | Pull an image from an unknown publisher, or one whose tag can be silently repointed | A3, A4 | Kyverno: publisher allow-list; explicit tag or digest and no :latest (a mutable tag is still a mutable tag; only the repo's own path is rewritten to a digest at admission) | demo/01, demo/09 |
| 4 | From a team-a pod, reach team-b pods or services | A2 | Cilium clusterwide default deny; per-namespace allow only to itself | demo/02 |
| 5 | From a team-a pod, call home or exfiltrate over the internet | A2 | Egress default deny; team-a is allowed api.github.com:443 only, by FQDN | demo/02 |
| 6 | From a team-a pod, reach the kubelet, the node, or the API server | A2 | Cilium egressDeny to host, remote-node and kube-apiserver entities (deny wins over any allow); no SA token mounted | demo/02, demo/04 |
| 7 | Use team-a's identity against team-b or the cluster | A3 | Namespaced Role only, no RBAC verbs, no cluster-scoped grants | demo/03 |
| 8 | Create enough objects to hurt shared components | A5, A3 | ResourceQuota with pod and deployment counts, LimitRange defaults | demo/03 |
| 9 | Run a known-vulnerable image and nobody notices | A4 | Trivy Operator scans every image; alert when a tenant image carries a critical CVE | demo/06, alert TenantImageHasCriticalCVE |
| 10 | Get a shell in a pod and look around quietly | A2 | Tetragon records every exec with pod, binary and arguments; Hubble records every denied flow | demo/04, demo/02 |
| 11 | Read secrets out of an etcd snapshot or backup taken without the key | A4, or anyone holding a backup | EncryptionConfiguration (aescbc) for secrets at rest. The key lives on cp-1 next to etcd, so this does not help against someone with root on cp-1; it helps against copies of the data | demo/10 |
| 12 | Push a bad policy or manifest to the cluster | A4 | Everything after the CNI goes through Argo CD from this repo; policies have CLI tests in CI and main is protected by those checks; audit before enforce. Not covered: the Helm charts Argo pulls (Kyverno, Trivy, monitoring, Tetragon) are version-pinned but their provenance is not verified and their rendered output is not reviewed in CI | .github/workflows/policy-tests.yaml |
| 13 | Probe limits quietly, weaken a policy, exec into a pod | A3, insider | API server audit log: secrets at Metadata (who, which, never the content), RBAC, Kyverno and Cilium objects and exec at RequestResponse, other writes at Request | demo/11 |

## What is not covered

- SSH is open to the internet (key only). The reviewers need a way in and I do not have their IPs. Their key is
  restricted in authorized_keys to forwarding one port (127.0.0.1:6443) with no shell, on cp-1 only. In a real
  deployment this is a bastion or a VPN, and 22 joins the admin allow-list.
- Image signatures are verified at admission only for images from this repo's registry path. A tenant that brings
  an image from an allow-listed publisher (docker.io/library, nginxinc, quay.io/cilium, registry.k8s.io) is not asked
  for a signature. Widening verifyImages to every tenant image means tenants sign their images with an identity we
  trust, which is a product decision, not a policy line.
- No mTLS between services. Cilium can do it, but it is still a beta feature and I would rather not promise it.
- Tetragon is observe only. No enforcement policy, no sigkill on escape precursors. First you learn the baseline.
- The audit log sits on the control plane node. If cp-1 is lost, so is the log. It needs shipping off the node.
- Kyverno's webhook is fail-closed everywhere except kube-system. Right for tenants; for platform namespaces it
  means a Kyverno outage blocks the platform's own rollouts. I hit this once (see worklog). Open question.
- Single control plane node.
- No persistent storage anywhere in the monitoring stack (no storage class on these VMs): Prometheus metrics,
  Alertmanager silences and notification state, and Grafana's local users all vanish when the pod is rescheduled.
  Dashboards and alert rules come back from git. The alert history for the demos is in the demo files.
- The firewall relies on three fixed node IPs. Adding a node means editing a file and re-applying it on every node.
- A compromised node has unrestricted host-level reach to the other two nodes: the node rule accepts every port from
  the peer IPs. Narrowing it to the cluster ports (6443, 2379-2380, 10250, 8472/udp, 4240, 4244) is a small change
  I did not make; the current rule is what let me find the Cilium problems quickly, and it is the wrong default.
- Source-address filtering assumes the provider does anti-spoofing on the public network. Not evidenced.
- Within a tenant namespace every pod can reach every other pod on every port. That is tenant isolation, not
  workload segmentation. Per-app policies are the tenant's job and this design leaves them room to write them.
