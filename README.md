# Hardened Kubernetes on Verda Cloud

Take-home for the Kubernetes and Container Platform Security Specialist role. Three CPU VMs in FIN-01, kubeadm
1.36, Cilium with kube-proxy replacement, Argo CD from this repo, Kyverno in Enforce, Trivy Operator, Tetragon,
Prometheus and Grafana. Two tenant namespaces that cannot see each other, running an image this repo builds, scans and signs.

Start with `docs/summary.md` (one page). Then `demo/` for the proof, `docs/threat-model.md` for the why.

## Layout

| Path | What |
|---|---|
| `docs/summary.md` | the one-page deliverable: built, worked, did not, trade-offs, missing |
| `docs/threat-model.md` | actors, attack paths, the control for each, the gaps |
| `docs/decisions.md` | every non-obvious choice, five lines each |
| `docs/attack-scenarios.md` | three scenarios run against the cluster, mapped to demo output |
| `docs/cve-triage.md` | triage of the 14 criticals Trivy found in the tenant image |
| `docs/scale.md` | what changes at 10 clusters / 500 nodes |
| `docs/runbook.md` | the commands I ran, in order |
| `docs/worklog.md` | what broke and what I did, written as it happened |
| `demo/` | one file per control with the attempt and the output (01 to 11); two Grafana screenshots |
| `infra/` | kubeadm config, audit policy, encryption config shape, nftables rule set |
| `bootstrap/` | Cilium values and the Argo CD install values and root app (the two things installed by hand) |
| `platform/` | Argo CD Applications for Kyverno, Trivy, monitoring, Tetragon, policies and tenants; alert rules; dashboard |
| `policies/kyverno/` | PSS restricted, no :latest, registry allow-list, cosign signature verification; `tests/` for the CLI |
| `network/cilium/` | clusterwide default deny, DNS carve-out, host and API server deny |
| `tenants/` | team-a and team-b: namespace, RBAC, quota, network policy, app |
| `app/` | the tenant image: Dockerfile and the workflow that builds, scans, signs it |

## Access for reviewers

Nothing is exposed publicly except SSH (key only) and the API server to one admin IP. Everything else is reached
through an SSH tunnel to the control plane. You need the reviewer SSH key and the reviewer kubeconfig, both sent
separately, not in this repo. The key cannot open a shell: on cp-1 it is restricted to forwarding 127.0.0.1:6443.
The kubeconfig is a ServiceAccount token bound to `platform/reviewer/reviewer-readonly.yaml`: get/list/watch on
most things, no secrets, no exec, plus create on pods/portforward so the tunnels below work.

```sh
# 1. API server through the tunnel (the key allows nothing else)
ssh -i verda-reviewer -N -L 6443:127.0.0.1:6443 root@65.108.33.68 &
export KUBECONFIG=./verda-reviewer.kubeconfig     # server: https://127.0.0.1:6443, read-only ClusterRole
kubectl get nodes

# 2. Grafana (user: reviewer, password sent separately)
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
open http://localhost:3000/d/trivy-operator            # Trivy dashboard
open http://localhost:3000/alerting/list               # the four security alerts

# 3. Argo CD (user: admin, password sent separately)
kubectl -n argocd port-forward svc/argocd-server 8080:80
open http://localhost:8080

# 4. Hubble UI
kubectl -n kube-system port-forward svc/hubble-ui 12000:80

# 5. Tetragon events: the reviewer role has no exec, so read the demo logs or ask me to run it on a call
```

| Node | IP | Role |
|---|---|---|
| cp-1 | 65.108.33.68 | control plane, API server 6443 |
| worker-1 | 65.108.32.164 | worker |
| worker-2 | 65.108.33.95 | worker |

The cluster stays up until you tell me you are done with it. Cost is about $0.19/h for the three nodes.

## Reproducing it

`docs/runbook.md` has the commands in order: node prep, kubeadm init with `infra/kubeadm-config.yaml`, join the
workers, Cilium from `bootstrap/cilium/values.yaml`, nftables, Argo CD from `bootstrap/argocd/`, the root app, and
the reviewer access. Argo CD does the rest from this repo.

## Tooling note

Claude Code was used for Helm values scaffolding, for drafting the longer YAML, and for debugging (it found the
`flush ruleset` versus Cilium iptables problem faster than I would have). The decisions, the policies' intent, the
demo attempts and the write-ups are mine, and every output in `demo/` was produced on this cluster.
