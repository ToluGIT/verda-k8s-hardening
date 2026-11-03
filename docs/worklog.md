# Worklog

Notes as I went. Dates are UK time. Where something broke I've kept the error text I was looking at.

## Fri 31 Oct
- Re-read the brief and my notes from the call with Vishal. Going with kubeadm over RKE2 so the control plane flags are
  mine to set and explain. Cilium for the CNI, Argo CD for everything after the CNI. Sketched the repo layout and the
  build order: nodes, control plane, CNI, firewall, Argo CD, then everything else through git.
- Commands go into docs/runbook.md as I run them, so it ends up as a record of what happened, not a plan.

## Sun 1 Nov
- Verda console: the 4-core Turin in FIN-03 showed as available, then refused with "not enough resources". The Rome+
  4-core (CPU.4V.16G) in FIN-01 worked. FIN-01 doesn't have private networking, so public IPs only. $0.062/h a node.
- First `kubeadm init` stalled at wait-control-plane. kubelet log: "Failed to start ContainerManager ... invalid
  kernel flag: vm/overcommit_memory expected 1 actual 0, kernel/panic expected 10, kernel/panic_on_oops expected 1".
  That's `protectKernelDefaults: true` refusing to run on a node that isn't configured the way it demands, which is
  what I want from it. Set the three sysctls in /etc/sysctl.d/k8s.conf, reset, init again. Kept the flag.
- Three nodes joined. Cilium 1.20.4 with kube-proxy replacement, VXLAN, Hubble relay and UI. The default deny comes
  in later as a CiliumClusterwideNetworkPolicy so it's a reviewable object in git, not a Helm flag.
- nftables on every node: drop by default, nodes talk freely to each other, SSH from anywhere (keys only), 6443 from
  my IP. SSH-from-anywhere is because the reviewers need a way in and I don't have their addresses. Noted in the
  threat model.
- Argo CD 10.11.6 installed with Helm, values in git. From here on it owns everything under platform/.
- Then pods had no DNS, no ClusterIP, no egress, while the nodes themselves pulled images fine. I suspected the
  firewall, flushed it, still broken. `cilium-dbg status` said "Masquerading: IPTables". Cilium writes its NAT and
  the proxy's TPROXY rules through iptables-nft, and my nftables.conf started with `flush ruleset`, which deleted
  them when I applied it. Two changes: the firewall now flushes only its own `inet filter` table, and Cilium
  masquerades in BPF (`bpf.masquerade: true`) so pod egress doesn't depend on netfilter tables at all. Restarted the
  agents, pods could reach the API, DNS and the internet again. About an hour lost. Argo CD's pre-install hook had
  failed earlier for the same reason.
- Kyverno policies went in as Audit. Policy reports for team-a and team-b: both web deployments pass every rule.
  Switched all three to Enforce in the next commit. If a policy breaks the platform team's deploy on day one it gets
  turned off and stays off, so Audit first was the whole point.
- Monitoring never got its Deployments. Argo's sync result: "failed calling webhook validate.kyverno.svc-fail ...
  timeout". Kyverno's webhook is fail-closed for every namespace except kube-system, and the first monitoring sync
  had run while pod networking was still broken, so the API server couldn't reach Kyverno and refused the
  Deployments. Argo then waited for a Prometheus that was never going to appear. Forced a resync and added a retry
  policy to every Application. Fail-closed is right for tenant namespaces; whether platform namespaces should be
  fail-open is a real question and it's in the threat model.
- Trivy: both tenant images (nginx-unprivileged 1.27.3) carry 14 critical CVEs. That means the critical-CVE alert
  is going to fire on a real image, and it gave me the subject for the CVE triage note.
- Image pipeline, first run: the Trivy gate failed the build. The fresh `1.29-alpine` base ships brotli-libs 1.2.0-r0
  with a critical (CVE-2026-111497) that has a fix, so the image was never pushed or signed. Added
  `apk upgrade brotli-libs` to the Dockerfile instead of an ignore entry.
- Second run: clean scan, pushed, signed keyless (GitHub OIDC), SBOM attested.
  ghcr.io/tolugit/verda-k8s-hardening/web@sha256:1d7bac0b049f82bf18ec0f6217f2099ed5d210976728c8a8d1fc604ca0e32236
  The package came out private, so the tenants stayed on the Docker Hub image and the alert kept firing until I made
  it public. verifyImages was already in Enforce; it only covers the repo's GHCR path, so nothing else was affected.
- Tetragon policy loaded on all three nodes (monitor_only). The in-container read of /etc/shadow produced no file
  event because the kernel refused the open; the policy hooks the read/write permission check, not the open. The
  demo says so.
- Rolled the signed digest out to the tenants. Kyverno refused it: "no signatures found", while `cosign verify` on
  my laptop passed. `cosign tree` showed why: cosign v3 (from cosign-installer v4) writes the signature as an OCI
  referrer Sigstore bundle and no longer creates the `sha256-<digest>.sig` tag that Kyverno's default `Cosign`
  attestor reads. Kyverno 1.19 has `type: SigstoreBundle` for this. One line in the policy.
- `type: SigstoreBundle` got further ("sigstore bundle verification failed: no matching signatures found") but still
  refused an image that `cosign verify` accepts with the same issuer and subject. I didn't chase it. The workflow now
  also writes the legacy `.sig` tag (`--new-bundle-format=false`) alongside the bundle, and the policy is back on the
  default Cosign attestor. Two signatures of the same digest, one verifier path that's been around for years. The
  bundle path is on the list to revisit with a newer Kyverno.
- Tenants on the signed digest. Trivy rescanned: 0 critical, 10 high. The critical-CVE alert resolved on its own.

## Mon 2 Nov
- Went back through everything against the brief, line by line, as if I were the reviewer. Things I'd got wrong: the
  reviewer SSH key was unrestricted root on all three nodes, now forwarding-only to 127.0.0.1:6443 on cp-1; the
  reviewer role couldn't port-forward, so the README's own tunnel commands would have failed, added create on
  pods/portforward; the registry allow-list permitted any Docker Hub account, which isn't an allow-list, now named
  publishers; the CI step that applies the policies to the tenant manifests was loading zero resources (no
  recursion), now every file, and it fails on any fail; actions pinned to commit SHAs, kyverno and syft downloads
  checksummed; the threat model, decisions and CVE triage still described the state before signing landed; demo 05
  was a typed port list with no non-admin vantage point, now real probes from my IP and from a GitHub runner; there
  was no audit-log evidence (demo/11 now); the Hubble excerpt in demo 02 only showed the kubelet drops.
- Second read, on the access story. Reviewers would have had the Argo CD admin password, which can deploy anything
  no matter how read-only the kubeconfig is; they now get a local `reviewer` account mapped to role:readonly (list
  and view, no sync, no delete, checked with `argocd account can-i`). The audit policy was logging
  serviceaccounts/token responses at RequestResponse, which puts live bearer tokens in audit.log; now Metadata. sshd
  was described as key-only but never configured; password and keyboard-interactive auth are off on all nodes and
  demo 05 shows `sshd -T`. The firewall now drops pod-CIDR sources arriving on eth0 before accepting pod-CIDR sources
  from the Cilium interfaces. SecurityAgentDown fires on an absent DaemonSet, not only a degraded one. CI namespace
  values now include team-b. Demo 01's registry case re-run against the publisher allow-list; demo 04 and 08 command
  lines now match their recorded events; "pinned tags" corrected to what it is: an explicit tag is required, and only
  the repo's own path gets rewritten to a digest.
- Third read, on whether the evidence reproduces. Demo 05 was still showing the rule set from before the anti-spoof
  rule, re-captured; commands that had "..." or were missing their quoting are now the full commands as typed; demo
  03's quota section re-run with the events command shown and the self-heal revert explained; the image workflow was
  building twice, once to scan and once to push, so the scanned and pushed images weren't provably the same bytes,
  now it pushes the image it scanned and the base is pinned by digest; demo 11 still said token requests were at
  RequestResponse; demo 06 now lists the cp-1 node report; "a bad policy fails CI before Argo sees it" was only true
  with branch protection, which is now on for main (required checks: kyverno, yaml); the threat model now says the
  monitoring stack has no persistence at all, and that Helm chart provenance isn't verified.
- Still open, and written down as such: the node-to-node rule accepts every port between the three nodes; UDP 8472
  can't be probed meaningfully from outside; the upstream signature on syft's checksum file isn't verified.
