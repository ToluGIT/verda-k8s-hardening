# Decisions

Five lines each. What I chose, what I did not, and why.

**kubeadm, not RKE2.** RKE2 would have given me a CIS-shaped cluster faster. kubeadm meant every flag is mine:
the audit policy, encryption at rest, `protectKernelDefaults`, `seccompDefault`, no kube-proxy. The brief asks what
I picked and why; with RKE2 the honest answer to half of it would be "the distro did it".

**Cilium replaces kube-proxy.** One datapath instead of two. kube-proxy was skipped at init, so there is no iptables
service path to reason about alongside BPF policy. Cost: the API server address is hard-coded in the Cilium values.

**Default deny as a CiliumClusterwideNetworkPolicy, not a Helm flag.** Both work. The policy object is in git,
it is reviewable, Argo CD owns it, and a tenant can read it to understand why their traffic is dropped.

**Kyverno, not Gatekeeper.** The platform team should be able to read a policy without learning Rego. Kyverno
policies look like the resources they govern, the CLI tests are trivial, and verifyImages is built in for later.
Kyverno moving to CEL policies is noted; the three policies here are small enough to port in an afternoon.

**Audit first, then enforce, as two commits.** A policy that blocks the platform team's own deploy on day one gets
switched off and stays off. Audit showed both tenant workloads clean, then Enforce went in. The history shows it.

**Argo CD owns everything after the CNI.** Cilium is installed by hand because Argo CD needs a network to exist.
Argo CD itself is installed by hand with values in git. From then on nothing is `kubectl apply`'d by a human,
including the policies and the alert rules. The demos use kubectl to attack, not to configure.

**Host firewall in nftables, scoped to its own table.** First version used `flush ruleset` and silently deleted
Cilium's iptables-nft rules. Now it only flushes `inet filter`. Cilium masquerades in BPF so pod egress does not
depend on netfilter at all. The two now coexist without knowing about each other.

**Public IPs only.** FIN-01 had capacity; private networking is FIN-03 only, which had a maintenance window the
same evening. Node-to-node traffic therefore crosses the public interface, allowed by source IP. In production
this is a private network and the firewall's node rule becomes a subnet.

**Tetragon observe-only, Falco not installed.** One eBPF sensor is enough to show the signal. Enforcement before a
baseline exists produces false kills. Falco would duplicate Tetragon here.

**No mTLS, no Harbor.** Each is a real control and each is half a day. The brief says a defensible set, not
everything. What is missing is listed, with the order I would add them.

**Signature verification scoped to our own registry path.** The tenants' image is built, scanned and signed by
this repo and Kyverno refuses anything on that path without the signature. Images from other allow-listed
publishers are not asked for a signature, because whose signature to trust for a tenant's own image is a product
decision. The allow-list is publishers, not registries: `docker.io/*` would make it meaningless.

**Reviewer access through SSH tunnels, not an ingress.** A public TLS endpoint for Grafana and Argo CD adds an
exposed service to a cluster whose point is a small surface. The reviewer key can only forward 127.0.0.1:6443 on
cp-1 (`restrict,port-forwarding,permitopen=...` in authorized_keys, no shell), and the kubeconfig is a read-only
ServiceAccount token with create on pods/portforward so the Grafana and Argo CD tunnels work.

**One Trivy dashboard, four alerts.** The community Trivy dashboard gives the counts by severity and type; the per-namespace and per-image panels in it depend on metrics the operator only emits with extra flags, and several are empty here, so the VulnerabilityReports themselves are the drill-down. The alerts
are the four things an on-call would want paged for: a critical CVE in a tenant, a burst of admission denials, a
burst of policy drops, a security agent missing from a node.
