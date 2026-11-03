# Summary

## What I built

Three Verda CPU VMs in FIN-01: a kubeadm 1.36 control plane and two workers, public IPs only. Cilium is the CNI and
replaces kube-proxy, and Hubble keeps the flow log, so every allowed and dropped connection is on record. Argo CD
owns everything after the CNI, from this repo. Kyverno sits at admission with PSS restricted, an explicit tag or
digest, a publisher allow-list, and signature verification for the image this repo builds. Trivy Operator scans
images, workload config and the nodes. Tetragon watches process and credential-file activity on every node.
Prometheus and Grafana carry four alerts and a Trivy dashboard. Two tenants, team-a and team-b, each with its own
service account, quota and a default-deny network they can only leave for DNS, and in team-a's case api.github.com.

The point of the design is that a tenant is a namespace label. Label a namespace `tenant: "true"` and it inherits
the admission policies, the default deny, the host and API server deny, and the quota, with nothing else to remember.

## What worked

- Audit first, Enforce one commit later. The policy reports showed both tenants clean under Audit, so switching to
  Enforce changed nothing for the workloads and everything for the next bad pod. No one got paged.
- Default deny as a clusterwide Cilium object in git, with the per-tenant allows next to it. When team-a reaches for
  team-b the Hubble line says `POLICY_DENIED`; when it reaches for the kubelet or the API server it says
  `denied by denylist`.
- The critical-CVE alert fired on the image as it was deployed, before I had done anything to provoke it. Triage,
  rebuild on a smaller base, sign, admit, and it resolved on its own.
- The signing pipeline refused its own first build. The new alpine base had a fixable critical in brotli, so no
  signature and no admission until the package was patched.

## What did not work, and what I did

- `protectKernelDefaults: true` kept kubelet from starting on the Verda image. It wanted three sysctls the image
  ships at zero. I set them and kept the flag; it was doing exactly what I asked it to do.
- My nftables rule set opened with `flush ruleset`, which also removed Cilium's iptables-nft rules. Pods lost DNS and
  egress for about an hour before `cilium-dbg status` showed masquerading had gone. The firewall now flushes only
  its own table, and Cilium masquerades in BPF, so the two no longer share a dependency.
- Kyverno's webhook is fail-closed, which is right for tenants. While networking was broken it also refused the
  monitoring stack's Deployments, and Argo CD sat waiting for a Prometheus that would not exist. I resynced and gave
  every Application a retry policy. Whether platform namespaces should fail open is an open question in the threat
  model.
- cosign v3 writes signatures as Sigstore bundles. Kyverno's default verifier looks for the legacy `.sig` tag, and
  its bundle verifier would not accept the bundle either. The workflow now writes both formats; a signing-tool
  upgrade should not be able to stop deploys silently.
- Argo CD showed the Kyverno apps OutOfSync with an empty diff: server-side apply meeting API server defaults. I
  diffed live against git field by field and ignored exactly those fields, nothing wider.

## Trade-offs

- kubeadm over RKE2. Slower to stand up, but every hardening flag is one I chose and can explain.
- Public IPs only. FIN-01 had capacity; node traffic crosses the public interface, allowed by source address. A
  private network is the production answer.
- SSH open to the internet, keys only. Reviewers need a way in and I do not have their addresses. The reviewer key
  can only forward the API server port.
- Tetragon in observe mode. Enforcing before you know the baseline produces false kills, and false kills get sensors
  switched off.
- Signatures verified only for the repo's own registry path. Whose signature to trust for a tenant's own image is a
  product decision.

## What is missing

In the order I would add them:

- Ship the audit log off the control plane node; today losing cp-1 loses the log.
- Narrow the node-to-node firewall rule to the cluster ports instead of every port.
- Persistent storage for Prometheus, Alertmanager and Grafana; a reschedule currently drops metrics and local users.
- mTLS between services. Cilium's own mutual authentication is still marked beta; the alternative is a service mesh.
- Tetragon enforcement, starting with actions that have no legitimate use in a container and usually precede an
  escape: kernel module loads, host filesystem mounts, writes to `/proc/sys`.
- Per-tenant DNS allow-lists instead of `matchPattern: "*"`, which leaves DNS as an exfiltration channel.
- A second control plane node.
- Alertmanager receivers, so the alerts page someone.
- Verification of the Helm charts Argo pulls; they are version-pinned, not provenance-checked.

The cluster will stay running until you confirm the review is finished, then I will delete the three instances.
Running cost is about $0.19 an hour.
