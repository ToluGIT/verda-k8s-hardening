# Demo 07: a security alert that fires

Date: 2026-11-01T10:10Z. The rule is platform/monitoring/security-alerts.yaml, delivered by Argo CD. The image is the one the tenants were running; docs/cve-triage.md is what I would do about it.

```
# the four rules Prometheus loaded
  TenantImageHasCriticalCVE    state=firing    health=ok
  KyvernoDenialsSpike          state=inactive  health=ok
  CiliumPolicyDropsSpike       state=inactive  health=ok
  SecurityAgentDown            state=inactive  health=ok

# alerts in Prometheus
  TenantImageHasCriticalCVE firing namespace=team-b image=nginxinc/nginx-unprivileged:1.27.3 value=1.4e+01 since=2026-11-01T09:51:14Z
  TenantImageHasCriticalCVE firing namespace=team-a image=nginxinc/nginx-unprivileged:1.27.3 value=1.4e+01 since=2026-11-01T09:51:14Z

# received by Alertmanager
  TenantImageHasCriticalCVE active namespace=team-a severity=high
    nginxinc/nginx-unprivileged:1.27.3 has 14 critical findings. Rebuild or pin a fixed tag.
  TenantImageHasCriticalCVE active namespace=team-b severity=high
    nginxinc/nginx-unprivileged:1.27.3 has 14 critical findings. Rebuild or pin a fixed tag.

# the query behind it
  team-a nginxinc/nginx-unprivileged:1.27.3 = 14 critical
  team-b nginxinc/nginx-unprivileged:1.27.3 = 14 critical
```

Alertmanager has only the default receiver; pointing it at a pager or Slack is a values change. The other three rules are inactive, which is correct for a quiet cluster: demos 01 and 02 produced a handful of denials each, below the spike thresholds on purpose.

## Follow-up, 2026-11-01T18:40Z: the alert cleared

The tenants moved to the image this repo builds (demo/09). Trivy rescanned it: 0 critical, 10 high (was 14 / 68).
The metric behind the alert went to zero for both namespaces and the alert resolved on its own.

```
# Prometheus query (via kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090 and /api/v1/query):
#   sum by (namespace, image_digest) (trivy_image_vulnerabilities{severity="Critical", namespace=~"team-.*"})
  team-a tolugit/verda-k8s-hardening/web sha256:9592375ba07e = 0
  team-b tolugit/verda-k8s-hardening/web sha256:9592375ba07e = 0
$ kubectl get vulnerabilityreports -n team-a -o jsonpath='{range .items[*]}{.report.artifact.digest} crit={.report.summary.criticalCount} high={.report.summary.highCount}{"\n"}{end}'
sha256:9592375ba07eb80306e76013419bb38fe9ce1694d1409c150997753f85cc6790 crit=0 high=10
```

That is the whole loop the brief asks for: scan, alert, triage (docs/cve-triage.md), rebuild on a smaller base with
the one remaining fixable critical patched, sign, admit, alert clears.
