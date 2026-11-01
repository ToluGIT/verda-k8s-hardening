# Demo 10: secrets encrypted at rest

Date: 2026-11-01T18:47Z. infra/kubeadm-config.yaml points the API server at an EncryptionConfiguration (aescbc) whose key was generated on cp-1 and never left it. Proof: write a secret, read the raw etcd value.

```
$ kubectl -n team-a create secret generic enc-probe --from-literal=token=plaintext-marker-12345
$ kubectl -n kube-system exec etcd-cp-1 -- etcdctl get /registry/secrets/team-a/enc-probe   # first 120 bytes, non-printables as .


# the marker is not in etcd in the clear
occurrences of plaintext-marker in the raw value: 0

# and the API server still decrypts it for an authorised caller
$ kubectl -n team-a get secret enc-probe -o jsonpath='{.data.token}' | base64 -d
plaintext-marker-12345
```

The value is prefixed k8s:enc:aescbc:v1:key1, which is the provider and key name from the encryption config. Rotation is a second key in the list plus a re-write of all secrets, which is not automated here.
