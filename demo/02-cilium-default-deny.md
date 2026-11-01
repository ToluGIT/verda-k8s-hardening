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

# what Hubble recorded for team-a since 19:03:33 UTC, one line per destination (hubble observe --namespace team-a --verdict DROPPED --since 3m)
   1 Nov  1 19:04:04.052: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1 Nov  1 19:04:04.052: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) Policy denied by denylist DROPPED (TCP Flags: SYN)
   1 Nov  1 19:04:03.028: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1 Nov  1 19:04:03.028: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) Policy denied by denylist DROPPED (TCP Flags: SYN)
   1 Nov  1 19:04:02.004: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1 Nov  1 19:04:02.004: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) Policy denied by denylist DROPPED (TCP Flags: SYN)
   1 Nov  1 19:04:00.980: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1 Nov  1 19:04:00.980: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) Policy denied by denylist DROPPED (TCP Flags: SYN)
   1 Nov  1 19:03:59.929: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1 Nov  1 19:03:59.929: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.32.164:10250 (remote-node) Policy denied by denylist DROPPED (TCP Flags: SYN)
   1 Nov  1 19:03:58.292: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.33.68:6443 (kube-apiserver) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1 Nov  1 19:03:58.292: team-a/web-f9798d7d4-c5x4d (ID:31476) <> 65.108.33.68:6443 (kube-apiserver) Policy denied by denylist DROPPED (TCP Flags: SYN)

# and the allowed flows, for contrast (verdict FORWARDED, destination api.github.com and web.team-a)
   1 Nov  1 19:03:47.255: team-a/web-f9798d7d4-c5x4d:49680 (ID:31476) <- kube-system/coredns-589f44dc88-cmz2f:53 (ID:31880) dns-response proxy FORWARDED (DNS Answer "140.82.121.5" TTL: 30 (P
   1 Nov  1 19:03:47.238: team-a/web-f9798d7d4-c5x4d:49680 (ID:31476) <- kube-system/coredns-589f44dc88-cmz2f:53 (ID:31880) dns-response proxy FORWARDED (DNS Answer  TTL: 4294967295 (Proxy a
   1 Nov  1 19:03:47.236: team-a/web-f9798d7d4-c5x4d:49680 (ID:31476) -> kube-system/coredns-589f44dc88-cmz2f:53 (ID:31880) dns-request proxy FORWARDED (DNS Query api.github.com. AAAA)
   1 Nov  1 19:03:47.236: team-a/web-f9798d7d4-c5x4d:49680 (ID:31476) -> kube-system/coredns-589f44dc88-cmz2f:53 (ID:31880) dns-request proxy FORWARDED (DNS Query api.github.com. A)
   1 Nov  1 19:03:47.235: team-a/web-f9798d7d4-c5x4d:49680 (ID:31476) <- kube-system/coredns-589f44dc88-cmz2f:53 (ID:31880) dns-response proxy FORWARDED (DNS Answer RCode: Non-Existent Domai
```
