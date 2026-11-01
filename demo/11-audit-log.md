# Demo 11: the audit trail

Date: 2026-11-01T19:10Z. infra/audit-policy.yaml is loaded by the API server (infra/kubeadm-config.yaml) and writes /var/log/kubernetes/audit.log on cp-1. Four things happened, then the matching events were pulled from the log with a few lines of Python over SSH (fields: level, verb, user, impersonatedUser, objectRef, responseStatus).

What the policy records: secrets at Metadata (who touched which secret, never the content); RBAC, Kyverno and Cilium objects at RequestResponse (the full body); pods/exec, attach, portforward and token requests at RequestResponse; every other write at Request; the rest at Metadata, with kube-proxy, node and health-check noise dropped.

```
# 1. team-a's identity lists secrets in team-b (demo/03). Impersonated via kubectl --as, so both identities are in the event.
# 2. team-a's identity tries to bind cluster-admin to itself.
# 3. an exec into a team-a pod.
# 4. an annotation patch on a Kyverno ClusterPolicy.

  [RBAC denial: team-a identity listing secrets in team-b]
    level=Metadata verb=list user=kubernetes-admin impersonating=system:serviceaccount:team-a:team-a-dev
    resource=secrets ns=team-b name= code=403 reason=Forbidden at=2026-11-01T19:06:50Z
    message=secrets is forbidden: User "system:serviceaccount:team-a:team-a-dev" cannot list resource "secrets" in API group "" in the namespace "team-b"

  [RBAC denial: team-a identity binding cluster-admin to itself]
    level=RequestResponse verb=create user=kubernetes-admin impersonating=system:serviceaccount:team-a:team-a-dev
    resource=clusterrolebindings ns= name= code=403 reason=Forbidden at=2026-11-01T19:06:50Z
    message=clusterrolebindings.rbac.authorization.k8s.io is forbidden: User "system:serviceaccount:team-a:team-a-dev" cannot create resource "clusterrolebindings"
    requestObject present: the attempted ClusterRoleBinding body is in the log

  [kyverno policy change] level=RequestResponse verb=patch user=kubernetes-admin resource=clusterpolicies/ ns= name=disallow-latest-tag code=200 reason= stage=ResponseComplete at=2026-11-01T19:04:11Z
    requestObject present (62 bytes)

  [pod exec] level=RequestResponse verb=get user=kubernetes-admin resource=pods/exec ns=team-a name=web-f9798d7d4-c5x4d code=101 reason= stage=ResponseComplete at=2026-11-01T19:04:10Z

  audit.log: 51246 events, 100M, rotated at 100MB x3, 7 days (kube-apiserver flags)
```

The 403s are recorded with the impersonating admin and the impersonated service account, so a tenant probing its limits is visible even when nothing was allowed. The exec is at RequestResponse, so the command is in the log. The ClusterPolicy patch carries the request body, which is how a policy being weakened would be seen. Not done: shipping the log off cp-1. On a single control plane node, losing the node loses the log; that is in the threat model as an open gap.
