# What changes at 10 clusters and 500 nodes

Nothing here is built. This is how the design would have to change, and where it already would not.

**Policy rollout.** Today one Argo CD instance syncs one repo into one cluster. At ten clusters the repo stays the
source but gets a `clusters/<name>` overlay per cluster and an ApplicationSet that stamps the platform apps out per
cluster. Policies move through rings: a canary cluster in Audit, then Enforce there, then Audit everywhere, then
Enforce everywhere. Each step is a commit, and the policy reports from the previous ring are the gate for the
next one. The CI job stays the same; what it tests is the same set of files.

**Break-glass.** Two things need a documented way around the controls. A tenant that is genuinely blocked gets a
Kyverno PolicyException: namespaced, time-boxed by a cleanup policy, and visible in the same repo. An operator who
needs to act while Argo CD or Kyverno is unhealthy uses a break-glass kubeconfig held outside the cluster, with its
own audit rule at RequestResponse level and an alert that fires on first use. Argo CD's self-heal is paused for that
cluster for the duration, by committing the pause, not by clicking.

**Zero-downtime CNI upgrade.** Cilium upgrades node by node, agent restart only; the datapath keeps forwarding while
the agent is down and the per-node identity cache means policy stays enforced. The sequence is: upgrade the
operator, then roll the DaemonSet with `maxUnavailable: 1` and a PodDisruptionBudget on anything that cannot lose a
replica, and watch Hubble drop counts between nodes. The canary cluster goes first, a week ahead. The thing that
actually breaks upgrades at scale is a config key that was renamed; the pre-flight is `cilium upgrade --dry-run`
against the new chart in CI, not on a Tuesday night.

**Fleet CVE patching.** Three different clocks. Node OS: unattended security updates on, kernel reboots in a rolling
drain with the same `maxUnavailable: 1`, surfaced per node by the Trivy node collector. Platform images (Cilium,
Kyverno, Argo CD, Trivy): chart version bumps as pull requests opened by Renovate, with the Trivy report diff in the
PR. Tenant images: not ours to patch; the alert in this repo tells the tenant, and a Kyverno rule can start refusing
images with a critical and a fix available after a grace period, Audit first.

**The firewall.** Three fixed node IPs in a file does not scale. At 500 nodes it is a private network with the node
subnet in the rule set, and the rule set is baked into the node image or applied by the node bootstrap, not scp.

**What does not change.** Default deny per tenant namespace, PSS restricted at admission, quotas with object counts,
one eBPF sensor per node, one repo per fleet. Those are the same at 3 nodes and 500. What grows is the number of
places the evidence has to be collected from, so the dashboards and alerts move to a central Prometheus with
`cluster` as a label, and the policy reports get aggregated the same way.
