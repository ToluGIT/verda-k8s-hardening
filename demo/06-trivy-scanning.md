# Demo 06: vulnerability and misconfiguration scanning (Trivy Operator)

Date: 2026-11-01T09:46Z. The operator scans every image and workload config in the cluster and writes the result as a CRD. The Grafana dashboard and the TenantImageHasCriticalCVE alert read the same data.

```
# tenant images
NAME                            REPOSITORY                    TAG      SCANNER   AGE
team-a         replicaset-web-5bc746474f-web                                    nginxinc/nginx-unprivileged     1.27.3         Trivy   7m46s
team-b         replicaset-web-5bdd4b4b89-web                                    nginxinc/nginx-unprivileged     1.27.3         Trivy   14s

# whole cluster, by namespace: critical / high
  argocd         crit=8    high=92   reports=5
  kube-system    crit=4    high=72   reports=12
  kyverno        crit=0    high=5    reports=5
  team-a         crit=14   high=68   reports=1
  team-b         crit=14   high=68   reports=1
  trivy-system   crit=0    high=0    reports=1

# misconfiguration findings on the tenant web deployments (config audit)
NAME                         SCANNER   AGE
team-a         limitrange-team-a-limits                                Trivy   14m
team-a         replicaset-web-5bc746474f                               Trivy   14m
team-a         resourcequota-team-a-quota                              Trivy   14m
team-a         service-web                                             Trivy   14m
team-b         limitrange-team-b-limits                                Trivy   14m
team-b         replicaset-web-5bdd4b4b89                               Trivy   14m
team-b         resourcequota-team-b-quota                              Trivy   14m
team-b         service-web                                             Trivy   14m

# node (CIS) checks, one report per node including the control plane (needed a toleration for the node collector, see worklog)
$ kubectl get clusterinfraassessmentreports
node-cp-1       Trivy   11h
node-worker-1   Trivy   11h
node-worker-2   Trivy   12h
```
