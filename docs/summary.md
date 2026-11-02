# Summary

## What I built

Three Verda CPU VMs in FIN-01: a kubeadm 1.36 control plane and two workers on public IPs. Cilium is the CNI and
replaces kube-proxy, Hubble keeps the flow log. Argo CD syncs everything after the CNI from this repo. Kyverno
enforces PSS restricted, an explicit tag or digest, a publisher allow-list, and cosign signatures (with digest
rewriting) for the image this repo builds. Trivy Operator scans images, workload config and nodes. Tetragon records
process and credential-file activity on every node. kube-prometheus-stack carries four alerts and a Trivy
dashboard. Two tenants, team-a and team-b, each with its own service account, quota and a default-deny network they
can only leave for DNS, and team-a for api.github.com.

Every preventive control has a demo with a blocked attempt and the output (demo/01 to 05, 09, 10). The detective
controls show what they caught (06, 07, 08, 11), including an alert that fired on the tenant image's 14 critical
CVEs and cleared after the rebuild.

## What worked

- Audit first, then Enforce one commit later, after the policy reports showed both tenants clean. Nothing broke.
- Default deny as a clusterwide Cilium policy in git. A tenant's cross-namespace request shows up in Hubble as
  `POLICY_DENIED`, and the host and API server attempts as `denied by denylist`.
- The alert fired on the image as deployed, not on anything planted. Triage, rebuild, sign, admit, and it cleared.
- The signing pipeline refused to sign its own first build because the new base image had a fixable critical.

## What did not work, and what I did

- `protectKernelDefaults: true` stopped kubelet on the Verda image until three sysctls were set. Kept the flag.
- My nftables rule set started with `flush ruleset`, which deleted Cilium's iptables-nft rules. Pods lost DNS and
  egress for about an hour. Scoped the firewall to its own table and moved Cilium masquerade to BPF.
- Kyverno's fail-closed webhook blocked the monitoring stack's first sync while networking was broken, and Argo CD
  waited on a Prometheus that never existed. Resynced, added a retry policy to every Application.
- cosign v3 stores signatures as Sigstore bundles. Kyverno's default verifier wanted the legacy `.sig` tag and its
  bundle verifier would not accept the bundle. The workflow now writes both formats.
- Argo CD showed the Kyverno apps OutOfSync with an empty diff: server-side apply plus API server defaults. Diffed
  live against git field by field and ignored exactly those fields.

## Trade-offs

- kubeadm over RKE2: more to set up, but every hardening flag is mine to explain.
- Public IPs only: FIN-01 had capacity; node traffic crosses the public interface, allowed by source IP.
- SSH open to the internet, key only, because reviewers need a way in and I do not have their IPs. The reviewer key
  can only forward the API server port.
- Tetragon observe-only: enforcement before a baseline produces false kills.
- Signature verification only for the repo's own registry path; other allow-listed publishers are not asked for one.

## What is missing

In the order I would add them: ship the audit log off the control plane; narrow the node-to-node firewall rule to
the cluster ports; persistent storage for the monitoring stack; mTLS once Cilium's mutual auth is stable; Tetragon
enforcement for the two or three escape precursors that are never legitimate; per-tenant DNS allow-lists instead of
`matchPattern: "*"`; a second control plane node; Alertmanager receivers; verification of the Helm charts Argo pulls.

## Cost and shutdown

About $0.19 per hour for the three nodes. The cluster stays up until you say you are done, then I delete it.
