# Demo 03: tenant RBAC and resource quota

Date: 2026-11-01T09:43Z. Acting as team-a's only identity, the team-a-dev service account, via kubectl --as.

```
# own namespace
$ kubectl auth can-i get secrets -n team-a --as=system:serviceaccount:team-a:team-a-dev
yes
$ kubectl auth can-i create deployments -n team-a --as=system:serviceaccount:team-a:team-a-dev
yes

# team-b's namespace
$ kubectl auth can-i get secrets -n team-b --as=system:serviceaccount:team-a:team-a-dev
no
$ kubectl auth can-i list pods -n team-b --as=system:serviceaccount:team-a:team-a-dev
no
$ kubectl auth can-i get pods/exec -n team-b --as=system:serviceaccount:team-a:team-a-dev
no

# escalation paths
$ kubectl auth can-i create rolebindings -n team-a --as=system:serviceaccount:team-a:team-a-dev
no
$ kubectl auth can-i create clusterrolebindings --as=system:serviceaccount:team-a:team-a-dev
Warning: resource 'clusterrolebindings' is not namespace scoped in group 'rbac.authorization.k8s.io'

no
$ kubectl auth can-i list nodes --as=system:serviceaccount:team-a:team-a-dev
Warning: resource 'nodes' is not namespace scoped

no
$ kubectl auth can-i get secrets -n kube-system --as=system:serviceaccount:team-a:team-a-dev
no
$ kubectl auth can-i create pods -n team-a --as=system:serviceaccount:team-a:team-a-dev --subresource=exec
no

# a real attempt, not just can-i
$ kubectl -n team-b get secrets --as=system:serviceaccount:team-a:team-a-dev
Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:team-a:team-a-dev" cannot list resource "secrets" in API group "" in the namespace "team-b"
$ kubectl create clusterrolebinding team-a-admin --clusterrole=cluster-admin --serviceaccount=team-a:team-a-dev --as=system:serviceaccount:team-a:team-a-dev
error: failed to create clusterrolebinding: clusterrolebindings.rbac.authorization.k8s.io is forbidden: User "system:serviceaccount:team-a:team-a-dev" cannot create resource "clusterrolebindings" in API group "rbac.authorization.k8s.io" at the cluster scope

# quota: object counts, not only CPU. Scale the web deployment to 25 replicas against a 20 pod quota.
$ kubectl -n team-a get resourcequota team-a-quota -o custom-columns=NAME:.metadata.name,PODS:.status.used.pods,POD_LIMIT:.status.hard.pods,DEPLOYMENTS:.status.used.count/deployments\.apps,DEPLOY_LIMIT:.status.hard.count/deployments\.apps
NAME           PODS   POD_LIMIT   DEPLOYMENTS   DEPLOY_LIMIT
team-a-quota   1      20          1             10
$ kubectl -n team-a scale deploy/web --replicas=25
deployment.apps/web scaled
$ kubectl -n team-a get deploy web -o custom-columns=NAME:.metadata.name,DESIRED:.spec.replicas,READY:.status.readyReplicas
NAME   DESIRED   READY
web    1         1
FailedCreate   Error creating: pods "web-5bc746474f-qdf5k" is forbidden: exceeded quota: team-a-quota, requested: limits.cpu=200m,pods=1, used: limits.cpu=4,pods=20, limited: limits.cpu=4,pods=20
$ kubectl -n team-a scale deploy/web --replicas=1
deployment.apps/web scaled
```
