# Demo 02: Cilium network policy, default deny for tenants

Date: 2026-11-01T19:03Z. Source pod: team-a/web-f9798d7d4-c5x4d. Clusterwide default deny plus a DNS carve-out; team-a may reach its own namespace and api.github.com:443, nothing else. Commands are shown as typed; curl's -w format is quoted.

```
# policies in force
$ kubectl get ccnp -o custom-columns=NAME:.metadata.name,DESCRIPTION:.spec.description
NAME                                 DESCRIPTION
tenant-allow-dns                     tenant pods may resolve names through kube-dns
tenant-default-deny                  default deny for tenant namespaces
tenant-deny-host-and-control-plane   tenants cannot reach nodes, kubelet, or the API server
$ kubectl get cnp -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name
NS       NAME
team-a   allow-same-namespace
team-a   egress-github-api-only
team-b   allow-same-namespace

# 1. same namespace: allowed
$ kubectl -n team-a exec web-f9798d7d4-c5x4d -- curl -s -m 5 -o /dev/null -w "%{http_code}\n" http://web.team-a.svc.cluster.local
200

# 2. cross-tenant via the service name (DNS resolves, the connection is dropped; 000 = timed out)
$ kubectl -n team-a exec web-f9798d7d4-c5x4d -- curl -s -m 5 -o /dev/null -w "%{http_code}\n" http://web.team-b.svc.cluster.local
000
command terminated with exit code 28

# 3. cross-tenant straight to team-b's pod IP, bypassing the service
$ kubectl -n team-a exec web-f9798d7d4-c5x4d -- curl -s -m 5 -o /dev/null -w "%{http_code}\n" http://10.244.2.80:8080
000
command terminated with exit code 28

# 4. the one allowed external name
$ kubectl -n team-a exec web-f9798d7d4-c5x4d -- curl -s -m 8 -o /dev/null -w "%{http_code}\n" https://api.github.com
200

# 5. any other external name
$ kubectl -n team-a exec web-f9798d7d4-c5x4d -- curl -s -m 5 -o /dev/null -w "%{http_code}\n" https://example.com
000
command terminated with exit code 28

# 6. the API server and a node's kubelet (egressDeny beats any allow)
$ kubectl -n team-a exec web-f9798d7d4-c5x4d -- curl -sk -m 5 -o /dev/null -w "%{http_code}\n" https://kubernetes.default.svc
000
command terminated with exit code 28
$ kubectl -n team-a exec web-f9798d7d4-c5x4d -- curl -sk -m 5 -o /dev/null -w "%{http_code}\n" https://65.108.32.164:10250/healthz
000
command terminated with exit code 28

# dropped flows involving team-a, grouped by destination and verdict (source column removed, counts on the left)
# hubble observe --namespace team-a --verdict DROPPED --last 4000 -o compact
  12 <> team-b/web-85bc747bb4-dzc6r (ID:57646) policy-verdict:none EGRESS DENIED
  12 <> team-b/web-85bc747bb4-dzc6r (ID:57646) Policy denied DROPPED
   6 <> 65.108.33.68 (kube-apiserver) policy-verdict:L3-Only EGRESS DENIED
   6 <> 65.108.33.68 (kube-apiserver) Policy denied by denylist DROPPED
   6 <> 65.108.32.164 (remote-node) policy-verdict:L3-Only EGRESS DENIED
   6 <> 65.108.32.164 (remote-node) Policy denied by denylist DROPPED
   6 <> 172.66.147.243 (world) policy-verdict:none EGRESS DENIED
   6 <> 172.66.147.243 (world) Policy denied DROPPED
   6 <> 104.20.23.154 (world) policy-verdict:none EGRESS DENIED
   6 <> 104.20.23.154 (world) Policy denied DROPPED

# forwarded flows for contrast: the DNS lookups by name, and the two allowed destinations
# hubble observe --namespace team-a --verdict FORWARDED --last 4000 -o compact
  12 <- kube-system/coredns-589f44dc88-cmz2f dns-response proxy FORWARDED (DNS Answer RCode: Non-Existent Domain
   6 <- kube-system/coredns-589f44dc88-79qql dns-response proxy FORWARDED (DNS Answer RCode: Non-Existent Domain
   3 -> kube-system/coredns-589f44dc88-cmz2f dns-request proxy FORWARDED (DNS Query web.team-b.svc.cluster.local.team-a.svc.cluster.local. AAAA)
   3 -> kube-system/coredns-589f44dc88-cmz2f dns-request proxy FORWARDED (DNS Query web.team-b.svc.cluster.local.team-a.svc.cluster.local. A)
   3 -> kube-system/coredns-589f44dc88-cmz2f dns-request proxy FORWARDED (DNS Query web.team-b.svc.cluster.local.svc.cluster.local. AAAA)
   3 -> kube-system/coredns-589f44dc88-cmz2f dns-request proxy FORWARDED (DNS Query web.team-b.svc.cluster.local.svc.cluster.local. A)
   3 -> kube-system/coredns-589f44dc88-cmz2f dns-request proxy FORWARDED (DNS Query web.team-b.svc.cluster.local.cluster.local. AAAA)
   3 -> kube-system/coredns-589f44dc88-cmz2f dns-request proxy FORWARDED (DNS Query web.team-b.svc.cluster.local.cluster.local. A)
```
