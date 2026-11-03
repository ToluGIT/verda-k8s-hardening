# Attack scenarios

Three scenarios run against the cluster as built: what I tried, what stopped it, what recorded it. The raw output is
in the demo logs referenced.

## 1. Compromised tenant workload tries to move laterally and call home

Start: a shell in team-a's web pod (as happens after an RCE in the app).

| Step | Result | Prevented by | Detected by |
|---|---|---|---|
| Look for a service account token | none mounted | `automountServiceAccountToken: false` on the Deployment | Tetragon: `ls /var/run/secrets/kubernetes.io` recorded (demo/04) |
| Reach team-b's service | timeout | Cilium clusterwide default deny; team-b's allow is same-namespace only | Hubble: `DROPPED POLICY_DENIED` team-a -> team-b (demo/02) |
| Reach the API server | timeout | `egressDeny` to the kube-apiserver entity | Hubble drop |
| Reach the kubelet on its own node | timeout | `egressDeny` to host and remote-node entities | Hubble: `Policy denied by denylist` to 10250 (demo/02) |
| Exfiltrate to an arbitrary internet host | timeout | egress default deny; only api.github.com:443 allowed by FQDN | Hubble drop; the DNS lookup itself is logged by the DNS proxy |
| Exfiltrate over DNS (encode data in query names) | queries resolve | **not prevented**: DNS to CoreDNS is allowed with `matchPattern: "*"` | Hubble DNS visibility records every query name; a rate or entropy rule would be the next control |

The last row is an open item. Tightening `matchPattern` to the names a tenant actually needs closes it, at the cost
of a per-tenant allow-list that someone has to maintain. For a platform where tenants bring arbitrary workloads
I would start with visibility and alert on query volume per pod before restricting names.

## 2. A tenant tries to escalate through RBAC and admission

Start: team-a's service account credentials (demo/03).

| Step | Result | Prevented by |
|---|---|---|
| Read secrets in team-b | Forbidden | Role is namespaced; no binding outside team-a |
| Bind cluster-admin to itself | Forbidden | no RBAC verbs in the Role; the API server's escalation check would refuse anyway |
| List nodes, read kube-system secrets | Forbidden | nothing cluster-scoped granted |
| Create a privileged pod to escape to the node | admission denied | Kyverno PSS restricted, Enforce (demo/01) |
| Pull a pre-built malicious image from a registry or Docker Hub account not on the publisher allow-list | admission denied | registry allow-list (demo/01). An image published under an allow-listed publisher is not caught by this rule; see the threat model |
| Run `image:latest` to slip a changed image under an existing tag | admission denied | no-latest rule (demo/01) |

## 3. Noisy neighbour, deliberate or not

Start: a tenant creates far more objects than anyone planned for. This came up in the interview: a quota on GPUs but
none on deployment count, and Prometheus fell over under the load.

| Step | Result | Prevented by |
|---|---|---|
| Scale a Deployment to 25 replicas | 20 pods then `exceeded quota: pods=20` | ResourceQuota (demo/03) |
| Create an 11th Deployment | would be refused | `count/deployments.apps: 10` |
| Create pods with no resource requests to dodge the CPU quota | pods get defaults | LimitRange `defaultRequest` |
| Open a NodePort or LoadBalancer | refused | `services.nodeports: "0"`, `services.loadbalancers: "0"` |

What is not covered: a tenant generating very high cardinality metrics or log volume. That is a per-namespace
scrape limit on Prometheus and a log rate limit, neither of which is configured here.

## Not attempted

A container breakout through a kernel bug. I didn't run an exploit. The controls that bound that class are PSS
restricted (no privileged, no host namespaces, seccomp RuntimeDefault, all capabilities dropped),
`protectKernelDefaults` on the kubelet, and Tetragon watching for the exec that usually follows. A kernel CVE of that
kind is what the triage note is for.
