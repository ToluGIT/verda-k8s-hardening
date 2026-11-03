# At 10 clusters and 500 nodes (design only)

## Rolling out policy

Today one Argo CD syncs one repo into one cluster. With ten clusters the repo stays the source of truth, but each
cluster gets its own overlay under `clusters/<name>` and an ApplicationSet stamps the platform apps out per cluster.
Policy changes go through rings: a canary cluster in Audit, then Enforce there, then Audit everywhere, then Enforce
everywhere. Each ring is a commit, and the policy reports from the previous ring are what lets the next one go. The
CI job doesn't change; it's testing the same files.

## Break-glass

Two situations need a documented way around the controls. A tenant who is blocked by a policy gets a Kyverno
PolicyException: scoped to their namespace, expiring automatically through a cleanup policy, and committed to the
same repo so it's visible. An operator who has to act while Argo CD or Kyverno is down uses a break-glass kubeconfig
held outside the cluster, with its own audit rule at RequestResponse and an alert the first time it's used. Argo
CD's self-heal gets paused for that cluster while they work, and the pause is itself a commit, so it's visible and
gets reverted.

## Upgrading the CNI without downtime

Cilium upgrades node by node and only the agent restarts; the datapath keeps forwarding while the agent is down,
and the per-node identity cache keeps policy enforced. The order is: upgrade the operator, roll the DaemonSet with
`maxUnavailable: 1`, keep a PodDisruptionBudget on anything that can't lose a replica, and watch Hubble drop counts
between nodes as you go. The canary cluster goes first, a week ahead. In my experience what breaks CNI upgrades at
scale is a renamed config key, so the safeguard that matters is running `cilium upgrade --dry-run` against the new chart in
CI before anyone touches a node.

## Patching CVEs across the fleet

Three different cadences. Node OS: unattended security updates on, kernel reboots done as a rolling drain with the
same `maxUnavailable: 1`, with the Trivy node collector showing which nodes still need it. Platform images (Cilium,
Kyverno, Argo CD, Trivy): chart bumps arrive as pull requests from Renovate, with the Trivy report diff in the PR.
Tenant images: not ours to patch. The alert in this repo tells the tenant, and after a grace period a Kyverno rule
can start refusing images that have a critical with a fix available, Audit first.

## The firewall

Three fixed node IPs in a file doesn't survive 500 nodes. By then it's a private network with the node subnet in the
rule set, and the rule set is baked into the node image or applied during node bootstrap.

## What doesn't change

Default deny per tenant namespace, PSS restricted at admission, quotas with object counts, one eBPF sensor per node,
one repo for the fleet. Those are the same at 3 nodes and 500. What grows is how many places the evidence has to be
collected from, so dashboards and alerts move to a central Prometheus with `cluster` as a label, and policy reports
get aggregated the same way.
