# Demo 07: a security alert that fires

Date: 2026-11-01T10:10Z. The rule is platform/monitoring/security-alerts.yaml, delivered by Argo CD. The image is the real tenant image, not one planted for the demo; docs/cve-triage.md is what I would do about it.

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
