# Worklog

Written as I go. Dates are UK time.

## Sun 1 Nov
- Read the brief again and the notes from the call with Vishal. Decided on kubeadm over RKE2 so the control plane
  flags are mine to set and explain. Cilium for the CNI, Argo CD for everything after the CNI.
- Keeping scripts to a minimum. Commands I ran go in docs/runbook.md as run.
- Verda console: Turin 4-core in FIN-03 said "available" then refused with "not enough resources". Rome+ 4-core
  (CPU.4V.16G) in FIN-01 worked. FIN-01 has no private networking, so public IPs only. $0.062/h per node.
- First `kubeadm init` failed at wait-control-plane. kubelet log: "Failed to start ContainerManager ... invalid
  kernel flag: vm/overcommit_memory expected 1 actual 0, kernel/panic expected 10, kernel/panic_on_oops expected 1".
  That is `protectKernelDefaults: true` doing its job. Set the three sysctls in /etc/sysctl.d/k8s.conf, reset, init again.
  Kept the flag; it is the kind of control the brief is asking for.
cilium chart version used: 1.20.4
- Three nodes joined. Cilium 1.20.4 with kubeProxyReplacement, vxlan, Hubble relay and UI. Default-deny comes next
  as a CiliumClusterwideNetworkPolicy so it is reviewable in git rather than a Helm flag.
- nftables on every node: drop by default, nodes talk freely to each other, SSH from anywhere (key only), 6443 from
  my IP. Decision on SSH-from-anywhere: reviewers need a way in and I do not have their IPs; key-only SSH is the
  compromise, noted in the threat model as residual.
argo-cd chart version used: 10.11.6
- Pods had no DNS, no ClusterIP, no egress, while the nodes pulled images fine. Spent a while suspecting the
  firewall, flushed it, still broken. `cilium-dbg status` said "Masquerading: IPTables". Cilium programs NAT and
  its proxy TPROXY rules through iptables-nft, and my nftables.conf opened with `flush ruleset`, which deleted
  them on apply. Two changes: the firewall now flushes only its own `inet filter` table, and Cilium masquerades
  in BPF (`bpf.masquerade: true`) so pod egress no longer depends on netfilter tables at all. Restarted the
  agents, pods reach the API, DNS and the internet. Argo CD's pre-install hook had failed for the same reason.
- Kyverno policies went in as Audit. Checked the policy reports for team-a and team-b: both web deployments pass all
  rules, nothing to fix. Flipped all three to Enforce in this commit. Audit-then-enforce is the whole point; a policy
  that breaks the platform team's deploy on day one gets switched off, not fixed.
- Monitoring never got its Deployments. Argo's sync result: "failed calling webhook validate.kyverno.svc-fail ...
  timeout". Kyverno's webhook is fail-closed for every namespace except kube-system, and the first monitoring sync
  ran while pod networking was still broken, so the API server could not reach Kyverno and refused the Deployments.
  Argo then sat there waiting for a Prometheus that would never exist. Forced a resync and added a retry policy to
  every Application. Fail-closed admission is the right default for tenant namespaces; whether platform namespaces
  should be fail-open is a real decision and it is in the threat model as an open question.
- Trivy: both tenant images (nginx-unprivileged 1.27.3) carry 14 critical CVEs. Useful: it means the critical-CVE
  alert will fire for real rather than from a planted image, and it is the subject of the CVE triage note.
- Image pipeline, first run: the Trivy gate failed the build. The fresh `1.29-alpine` base ships brotli-libs 1.2.0-r0
  with a critical (CVE-2026-111497) that has a fix. So the image was never pushed or signed, which is the point of the
  gate. Added `apk upgrade brotli-libs` to the Dockerfile instead of an ignore entry.
- Image pipeline, second run: clean scan, pushed, signed keyless (GitHub OIDC), SBOM attested.
  ghcr.io/tolugit/verda-k8s-hardening/web@sha256:1d7bac0b049f82bf18ec0f6217f2099ed5d210976728c8a8d1fc604ca0e32236
  Then the boring blocker: a package pushed by a workflow in a private repo is private, and GitHub only lets you
  change that in the web UI. Until it is public the tenants keep the Docker Hub image and the critical-CVE alert
  keeps firing. The verifyImages policy is in Enforce already; it only covers the repo's GHCR path, so nothing else
  is affected by the wait.
- Tetragon policy loaded on all three nodes (monitor_only). The in-container read of /etc/shadow produced no file
  event because the kernel denied the open; the policy hooks the read/write permission check. Wrote that down in the
  demo rather than pretending.
- Rolled the signed digest out to the tenants. Kyverno refused it: "no signatures found", while `cosign verify` on my
  laptop passed. `cosign tree` showed why: cosign v3 (from cosign-installer v4) writes the signature as an OCI
  referrer Sigstore bundle and no longer creates the `sha256-<digest>.sig` tag that Kyverno's default `Cosign`
  attestor type reads. Kyverno 1.19 has `type: SigstoreBundle` for exactly this. One line in the policy.
  Worth remembering: a signing tool upgrade on the CI side can silently break admission on the cluster side, and
  the failure mode is "nothing deploys", which is the right failure mode.
- `type: SigstoreBundle` got further ("sigstore bundle verification failed: no matching signatures found") but still
  refused an image `cosign verify` accepts with the same issuer and subject. Did not chase it. The workflow now also
  writes the legacy `.sig` tag (`--new-bundle-format=false`) alongside the bundle, and the policy is back on the
  default Cosign attestor. Two signatures of the same digest, one verifier path that is well trodden. The bundle
  path stays on the list to revisit with a newer Kyverno.
