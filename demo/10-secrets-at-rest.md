# Demo 10: secrets encrypted at rest

Date: 2026-11-01T18:49Z. infra/kubeadm-config.yaml points the API server at an EncryptionConfiguration (aescbc) whose key was generated on cp-1 and never left it. Proof: write a secret, read the raw etcd value through the etcd pod (distroless image, so etcdctl is called directly).

```
$ kubectl -n team-a create secret generic enc-probe --from-literal=token=plaintext-marker-12345
$ kubectl -n kube-system exec etcd-cp-1 -- etcdctl --endpoints=https://127.0.0.1:2379 --cacert=... --cert=... --key=... \
    get /registry/secrets/team-a/enc-probe --print-value-only | head -c 96   # non-printable bytes shown as .
k8s:enc:aescbc:v1:key1:..0.@...9e..d`e..8{..).A....w...4].`j.S@...^4.n..j.e.3............M......
$ ... --print-value-only | grep -c plaintext-marker
0
# the API server still decrypts it for an authorised caller
$ kubectl -n team-a get secret enc-probe -o jsonpath='{.data.token}' | base64 -d
plaintext-marker-12345
```

The stored value starts with k8s:enc:aescbc:v1:key1, the provider and key name from the encryption config, followed by ciphertext; the marker string does not appear anywhere in it. Key rotation means a second key in the providers list and a re-write of every secret, which is documented in infra/encryption.yaml.example and not automated here.
