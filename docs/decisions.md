# Decisions

The choices that weren't obvious, and why I made them. Short on purpose.

**kubeadm instead of RKE2.** RKE2 would have got me a CIS-shaped cluster in less time. I went with kubeadm because
the brief asks what I picked and why, and with kubeadm every hardening setting is one I had to choose: the audit
policy, encryption at rest, `protectKernelDefaults`, `seccompDefault`, dropping kube-proxy. With RKE2 a lot of that
comes pre-set and I'd be explaining someone else's defaults.

**Cilium replaces kube-proxy.** I skipped kube-proxy at `kubeadm init` so there's one datapath to reason about, not
BPF policy sitting next to an iptables service path. The cost is that the API server address is hard-coded in the
Cilium values, which would need handling in a multi-control-plane setup.

**Default deny is a Cilium policy object, not a Helm flag.** Both work. I wanted it in git as a reviewable object
that Argo CD owns, next to the per-tenant allow rules, so the whole network posture for a tenant is visible in one
directory. Tenants can't read the clusterwide object with their namespaced role, so the README and the Hubble drop
reason are how they'd find out why something was blocked.

**Kyverno over Gatekeeper.** The platform team should be able to read a policy without learning Rego. Kyverno
policies look like the resources they govern, the CLI tests are easy, and image verification is built in. Kyverno is
moving its policy types to CEL; the policies here are small and would port when that becomes necessary.

**Audit first, then Enforce, as two commits.** If a policy breaks the platform team's deploy on the first day it
gets switched off and nobody trusts the next one. So the policies went in as Audit, I checked the policy reports
for both tenants, saw them clean, and only then committed the switch to Enforce.

**Argo CD owns everything after the CNI.** Cilium has to be installed manually because Argo CD needs a working
network before it can run. Argo CD itself I also installed with Helm, with its values in git. After that nothing was
applied to the cluster by me directly: policies, alert rules, tenants, all of it goes through the repo. In the demos
I use kubectl to attack the cluster, not to configure it.

**The host firewall only touches its own table.** My first rule set started with `flush ruleset` and wiped Cilium's
iptables-nft rules along with it. Now it flushes only the `inet filter` table, and Cilium masquerades in BPF, so the
firewall and the CNI don't depend on each other.

**Public IPs only.** FIN-01 had capacity when I needed it and private networking is only in FIN-03, which had a
maintenance window that evening. So node-to-node traffic crosses the public interface, allowed by source address. In
production that's a private network and the node rule becomes a subnet.

**Tetragon in observe mode, and no Falco.** One eBPF sensor is enough to show the signal. Turning on enforcement
before I know what normal looks like would kill legitimate processes, and that's how sensors end up disabled. Falco
would be a second sensor doing a similar job.

**Image signatures checked only for this repo's registry path.** The tenant image is built, scanned and signed
here, and Kyverno refuses anything on that path without the signature. Images from the other allow-listed
publishers aren't asked for one, because deciding whose signature to trust for a tenant's own images is a product
decision, not a policy line. The allow-list is named publishers, not whole registries, because `docker.io/*` would
let any account through.

**No mTLS, no Harbor.** Each is a real control and each is half a day or more. The brief asks for a defensible set,
not everything. Both are in "what is missing" with the order I'd add them.

**Reviewer access through SSH tunnels, not an ingress.** Putting Grafana and Argo CD behind a public TLS endpoint
adds an exposed service to a cluster whose whole point is a small surface. The reviewer key on cp-1 can only forward
127.0.0.1:6443 (`restrict,port-forwarding,permitopen=...` in authorized_keys, no shell), and the kubeconfig is a
read-only ServiceAccount token with create on pods/portforward so the Grafana and Argo CD tunnels work.

**One Trivy dashboard, four alerts.** The community dashboard gives counts by severity and type. Several of its
per-namespace panels are empty because the operator only emits those metrics with extra flags, so the
VulnerabilityReports are the drill-down. The four alerts are the ones I'd want to be paged for: a critical CVE in a
tenant, a burst of admission denials, a burst of policy drops, a security agent missing from a node.
