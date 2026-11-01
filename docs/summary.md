# Summary

## What I built

Three Verda CPU VMs in FIN-01 running a kubeadm 1.36 cluster: one control plane, two workers, public IPs only.
Cilium is the CNI and replaces kube-proxy; Hubble records every flow. Argo CD syncs everything after the CNI from
this repo. Kyverno enforces the restricted Pod Security Standard, an explicit tag or digest (no :latest), a publisher
allow-list, and cosign signatures with digest rewriting for the image this repo builds. Trivy Operator scans images, workload config and the nodes. Tetragon records
process activity on every node. kube-prometheus-stack carries four alerts and a Trivy dashboard. Two tenant
namespaces, team-a and team-b, each with its own service account, quota, and a default-deny network they can only
leave for DNS and (team-a) api.github.com.

Each preventive control has a demo file with a blocked attempt and the output: admission denials, Hubble drop
records, RBAC refusals, quota failures, port probes against the firewall from an admin and a non-admin network,
secrets unreadable in raw etcd. The detective controls show what they caught: Tetragon exec and file events, Trivy
reports, the audit log, and an alert that fired on the tenant image's 14 critical CVEs and cleared after the rebuild.

## What worked

- The audit-then-enforce path. Policies went in as Audit, policy reports showed the tenant workloads clean, Enforce
  was one commit later. Nothing broke.
- Cilium default deny expressed as a clusterwide policy object, with the tenant allow rules alongside it in git.
  The Hubble output for a cross-tenant request says `POLICY_DENIED`, and the deny to the host and API server shows
  `denied by denylist`, which is exactly what you want a tenant to be able to read.
- The alert fired on the tenant image as deployed, before anything was planted for a demo.
- GitOps for the policies: the Kyverno CLI tests run in GitHub Actions on every change. During this build I pushed
  straight to main, so CI ran after Argo CD could already see the commit; with branch protection requiring the
  checks (now enabled on main) a failing policy cannot be merged, which is the shape this would have in production.

## What did not work, and what I did

- `protectKernelDefaults: true` stopped kubelet from starting on the Verda image until three sysctls were set.
  Kept the flag, set the sysctls, wrote it down.
- My nftables rule set began with `flush ruleset`, which deleted Cilium's iptables-nft NAT and proxy rules. Pods
  lost DNS and egress. The fix was to scope the firewall to its own table and move Cilium masquerade into BPF.
  Cost me about an hour; the two now ignore each other by design.
- Kyverno's fail-closed webhook blocked the monitoring stack's first sync while pod networking was broken, and
  Argo CD waited on a Prometheus that never existed. Forced a resync and added a retry policy to every Application.
- Argo CD kept showing the Kyverno apps OutOfSync with an empty diff. Server-side apply plus API server defaults.
  Diffed live against git field by field and ignored exactly those fields, nothing wider.

## Trade-offs

- kubeadm over RKE2: more to configure, but every hardening choice is mine to explain.
- Public IPs only: FIN-01 had capacity and no maintenance window that evening; node traffic crosses the public
  interface, allowed by source IP. In production this is a private network.
- SSH open to the internet, key only: reviewers need a way in and I do not have their IPs.
- Tetragon observe-only: enforcement before a baseline produces false kills.
- Four alerts that an on-call would act on, instead of the chart's full default set.
- Two signature formats on every image until Kyverno's bundle verifier is sorted, instead of pinning an old cosign.

## What is missing

In the order I would add them: ship the audit log off the control plane; mTLS between services once Cilium's
mutual auth is stable; Tetragon enforcement for the two or three escape precursors that are never legitimate;
per-tenant DNS allow-lists instead of `matchPattern: "*"`; a second control plane node; receivers on Alertmanager;
Falco or a second sensor only if Tetragon's coverage turns out to have a gap.

Signed images are done for this repo's own image (demo/09): the workflow builds, scans, pushes and cosign-signs it
with its GitHub OIDC identity; Kyverno refuses anything on that registry path without the signature and rewrites
tags to digests. Images from the other allow-listed publishers are not signature-checked; see the threat model.
Two things happened on the way: the scan gate refused the first build because the new base image had a fixable
critical CVE, and cosign v3's switch to Sigstore bundles broke Kyverno's default verifier until the workflow also
wrote the legacy signature format. Both are in the worklog.

## Cost and shutdown

About $0.19 per hour for the three nodes. The cluster stays up until you say you are done, then it is deleted.
