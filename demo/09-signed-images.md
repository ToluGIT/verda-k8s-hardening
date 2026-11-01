# Demo 09: signed images end to end

Date: 2026-11-01T18:39Z. The image is built, scanned, pushed and cosign-signed (keyless, GitHub OIDC) by .github/workflows/image.yaml. policies/kyverno/verify-image-signatures.yaml requires that signature on anything from ghcr.io/tolugit/verda-k8s-hardening/*.

```
# the signature, verified from my laptop against the workflow identity
$ cosign verify --certificate-oidc-issuer https://token.actions.githubusercontent.com \
    --certificate-identity https://github.com/ToluGIT/verda-k8s-hardening/.github/workflows/image.yaml@refs/heads/main \
    ghcr.io/tolugit/verda-k8s-hardening/web@sha256:9592375ba07eb80306e76013419bb38fe9ce1694d1409c150997753f85cc6790
Verification for ghcr.io/tolugit/verda-k8s-hardening/web@sha256:9592375ba07eb80306e76013419bb38fe9ce1694d1409c150997753f85cc6790 --
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The code-signing certificate was verified using trusted certificate authority certificates

# 1. the tenants run it, admitted by Kyverno (annotation written at admission)
$ kubectl get pods -A -l app=web -o custom-columns=NS:.metadata.namespace,IMAGE:.spec.containers[0].image
NS       IMAGE
team-a   ghcr.io/tolugit/verda-k8s-hardening/web@sha256:9592375ba07eb80306e76013419bb38fe9ce1694d1409c150997753f85cc6790
team-b   ghcr.io/tolugit/verda-k8s-hardening/web@sha256:9592375ba07eb80306e76013419bb38fe9ce1694d1409c150997753f85cc6790
$ kubectl -n team-a get pod web-f9798d7d4-c5x4d -o jsonpath='{.metadata.annotations}' | tr ',' '
' | grep verify
  kyverno.io/verify-images: {"ghcr.io/tolugit/verda-k8s-hardening/web@sha256:9592375ba07eb80306e76013419bb38fe9ce1694d1409c150997753f85cc6790":"pass
  kyverno.io/verify-images-scoped: {"version":1,"policies":{"/verify-image-signatures":{"cosign-keyless-github":{"ghcr.io/tolugit/verda-k8s-hardening/web@s

# 2. the same path, a digest with no signature the policy accepts: refused
$ kubectl -n team-a run sigtest --image=ghcr.io/tolugit/verda-k8s-hardening/web@sha256:27451a1daa36b17e0658580ebce199b7b1fff3a4672d962701bacfd00bb44ff0 --restart=Never --dry-run=server -o jsonpath='{.metadata.annotations.kyverno\.io/verify-images}' --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":101,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"c","image":"ghcr.io/tolugit/verda-k8s-hardening/web@sha256:27451a1daa36b17e0658580ebce199b7b1fff3a4672d962701bacfd00bb44ff0","securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
Error from server: admission webhook "mutate.kyverno.svc-fail" denied the request: 
resource Pod/team-a/sigtest was blocked due to the following policies 
verify-image-signatures:
  cosign-keyless-github: 'failed to verify image ghcr.io/tolugit/verda-k8s-hardening/web@sha256:27451a1daa36b17e0658580ebce199b7b1fff3a4672d962701bacfd00bb44ff0: .attestors[0].entries[0].keyless: no signatures found'

# 3. a tampered digest: refused
$ kubectl -n team-a run sigtest --image=ghcr.io/tolugit/verda-k8s-hardening/web@sha256:0000000000000000000000000000000000000000000000000000000000000000 --restart=Never --dry-run=server -o jsonpath='{.metadata.annotations.kyverno\.io/verify-images}' --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":101,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"c","image":"ghcr.io/tolugit/verda-k8s-hardening/web@sha256:0000000000000000000000000000000000000000000000000000000000000000","securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
Error from server: admission webhook "mutate.kyverno.svc-fail" denied the request: 
resource Pod/team-a/sigtest was blocked due to the following policies 
verify-image-signatures:
  cosign-keyless-github: 'failed to verify image ghcr.io/tolugit/verda-k8s-hardening/web@sha256:0000000000000000000000000000000000000000000000000000000000000000: .attestors[0].entries[0].keyless: no signatures found'

# 4. a mutable tag on the protected path: Kyverno resolves it, verifies the digest, and rewrites the pod to the digest
$ kubectl -n team-a run sigtest --image=ghcr.io/tolugit/verda-k8s-hardening/web:main --restart=Never --dry-run=server -o jsonpath='{.metadata.annotations.kyverno\.io/verify-images}' --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":101,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"c","image":"ghcr.io/tolugit/verda-k8s-hardening/web:main","securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
{"ghcr.io/tolugit/verda-k8s-hardening/web@sha256:9592375ba07eb80306e76013419bb38fe9ce1694d1409c150997753f85cc6790":"pass"}

```

On case 2: that digest is not unsigned in the strict sense. It is the image from the run where the legacy-format signing step failed, so it carries cosign v3's Sigstore bundle but not the sha256-<digest>.sig tag the Cosign attestor reads. For this policy the result is the same as unsigned, and the worklog has the full story: Kyverno 1.19's SigstoreBundle verifier would not accept the bundle, so the workflow now writes both formats. The pipeline also refused to sign at all on the first attempt because the base image had a fixable critical CVE (demo 06, docs/cve-triage.md).
