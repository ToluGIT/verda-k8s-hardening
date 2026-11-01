# Demo 01: Kyverno admission, Enforce mode

Date: 2026-11-01T09:43Z. Run from my laptop with the admin kubeconfig; commands shown in full, the --overrides JSON single-quoted as typed. All three attempts target team-a (tenant=true).

```
# policies in force
$ kubectl get cpol -o custom-columns=NAME:.metadata.name,ACTION:.spec.validationFailureAction,READY:.status.ready
Warning: kyverno.io/v1 ClusterPolicy is deprecated and will be removed in a future release; migrate to ValidatingPolicy, MutatingPolicy, GeneratingPolicy or ImageValidatingPolicy (policies.kyverno.io), see https://kyverno.io/docs/guides/migration-to-cel/
NAME                        ACTION    READY
disallow-latest-tag         Enforce   <none>
pod-security-restricted     Enforce   <none>
restrict-image-registries   Enforce   <none>

# 1. privileged container
$ kubectl -n team-a run priv --image=docker.io/library/busybox:1.37 --restart=Never --overrides='{"spec":{"containers":[{"name":"priv","image":"docker.io/library/busybox:1.37","command":["sleep","3600"],"securityContext":{"privileged":true}}]}}'
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Pod/team-a/priv was blocked due to the following policies 

pod-security-restricted:
  restricted: 'Validation rule ''restricted'' failed. It violates PodSecurity "restricted:latest": (Forbidden reason: allowPrivilegeEscalation != false, field error list: [spec.containers[0].securityContext.allowPrivilegeEscalation is forbidden, forbidden values found: nil])(Forbidden reason: unrestricted capabilities, field error list: [spec.containers[0].securityContext.capabilities.drop: Required value])(Forbidden reason: privileged, field error list: [spec.containers[0].securityContext.privileged is forbidden, forbidden values found: true])(Forbidden reason: runAsNonRoot != true, field error list: [spec.containers[0].securityContext.runAsNonRoot: Required value])(Forbidden reason: seccompProfile, field error list: [spec.containers[0].securityContext.seccompProfile.type: Required value])'

# 2. :latest tag (otherwise compliant pod)
$ kubectl -n team-a run latest --image=docker.io/library/busybox:latest --restart=Never --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"c","image":"docker.io/library/busybox:latest","command":["sleep","3600"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Pod/team-a/latest was blocked due to the following policies 

disallow-latest-tag:
  no-latest: 'validation error: The :latest tag is not allowed. rule no-latest failed at path /spec/containers/0/image/'

# 3. image from a registry not on the allow-list (server-side dry run, policy as of 2026-11-01)
$ kubectl -n team-a run foreign --image=registry.example.internal/team/app:1.0.0 --restart=Never --dry-run=server -o name --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"c","image":"registry.example.internal/team/app:1.0.0","command":["sleep","3600"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request: 
resource Pod/team-a/foreign was blocked due to the following policies 
restrict-image-registries:
  allowed-registries: 'validation error: Image publisher not on the allow-list (see policies/kyverno/restrict-image-registries.yaml). rule allowed-registries failed at path /spec/containers/0/image/'

# 3b. an allow-listed registry but a publisher that is not on the list
$ kubectl -n team-a run hubacct --image=docker.io/someoneelse/app:1.0.0 --restart=Never --dry-run=server -o name --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"c","image":"docker.io/someoneelse/app:1.0.0","command":["sleep","3600"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request: 
resource Pod/team-a/hubacct was blocked due to the following policies 
restrict-image-registries:
  allowed-registries: 'validation error: Image publisher not on the allow-list (see policies/kyverno/restrict-image-registries.yaml). rule allowed-registries failed at path /spec/containers/0/image/'

# 3c. and an allow-listed publisher, for contrast (passes this rule; PSS still applies)
$ kubectl -n team-a run okpub --image=docker.io/library/busybox:1.37 --restart=Never --dry-run=server -o name --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"c","image":"docker.io/library/busybox:1.37","command":["sleep","3600"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
pod/okpub

# 4. the compliant pod from the test suite, for contrast
$ kubectl -n team-a apply -f policies/kyverno/tests/resources/good-pod.yaml
pod/good created
$ kubectl -n team-a delete pod good --wait=false
pod "good" deleted

# nothing leaked through (the compliant test pod is still terminating, it was deleted with --wait=false)
$ kubectl -n team-a get pods -o name
pod/good
pod/web-5bc746474f-49sxr
```
