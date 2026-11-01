# Demo 02: Cilium network policy, default deny for tenants

Date: 2026-11-01T09:43Z. Source pod: team-a/web-5bc746474f-49sxr (nginx-unprivileged, has curl). Clusterwide default deny plus DNS carve-out; team-a may reach api.github.com:443 and its own namespace, nothing else.

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
$ kubectl -n team-a exec web-5bc746474f-49sxr -- curl -s -m 5 -o /dev/null -w team-a -> web.team-a: HTTP %{http_code}
 http://web.team-a.svc.cluster.local
team-a -> web.team-a: HTTP 200

# 2. cross-tenant: team-a -> team-b web service (DNS resolves, connection is dropped)
$ kubectl -n team-a exec web-5bc746474f-49sxr -- curl -s -m 5 -o /dev/null -w team-a -> web.team-b: HTTP %{http_code}
 http://web.team-b.svc.cluster.local
team-a -> web.team-b: HTTP 000
command terminated with exit code 28

# 3. allowed FQDN
$ kubectl -n team-a exec web-5bc746474f-49sxr -- curl -s -m 8 -o /dev/null -w team-a -> api.github.com: HTTP %{http_code}
 https://api.github.com
team-a -> api.github.com: HTTP 200

# 4. any other destination on the internet
$ kubectl -n team-a exec web-5bc746474f-49sxr -- curl -s -m 5 -o /dev/null -w team-a -> example.com: HTTP %{http_code} (000 = timed out, dropped)
 https://example.com
team-a -> example.com: HTTP 000 (000 = timed out, dropped)
command terminated with exit code 28

# 5. the API server and the node, denied explicitly (egressDeny beats any allow)
$ kubectl -n team-a exec web-5bc746474f-49sxr -- curl -sk -m 5 -o /dev/null -w team-a -> kube-apiserver: HTTP %{http_code}
 https://kubernetes.default.svc
team-a -> kube-apiserver: HTTP 000
command terminated with exit code 28
$ kubectl -n team-a exec web-5bc746474f-49sxr -- curl -s -m 5 -o /dev/null -w team-a -> node kubelet 10250: HTTP %{http_code}
 https://65.108.32.164:10250/healthz
team-a -> node kubelet 10250: HTTP 000
command terminated with exit code 28

# what Hubble saw (drops from team-a in the last minute)
   1  1 09:43:35.423: team-a/web-5bc746474f-49sxr:41918 (ID:31476) <> 65.108.32.164:10250 (host) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1  1 09:43:35.423: team-a/web-5bc746474f-49sxr:41918 (ID:31476) <> 65.108.32.164:10250 (host) Policy denied by denylist DROPPED (TCP Flags: SYN)
   1  1 09:43:34.399: team-a/web-5bc746474f-49sxr:41918 (ID:31476) <> 65.108.32.164:10250 (host) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1  1 09:43:34.399: team-a/web-5bc746474f-49sxr:41918 (ID:31476) <> 65.108.32.164:10250 (host) Policy denied by denylist DROPPED (TCP Flags: SYN)
   1  1 09:43:33.375: team-a/web-5bc746474f-49sxr:41918 (ID:31476) <> 65.108.32.164:10250 (host) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1  1 09:43:33.375: team-a/web-5bc746474f-49sxr:41918 (ID:31476) <> 65.108.32.164:10250 (host) Policy denied by denylist DROPPED (TCP Flags: SYN)
   1  1 09:43:32.351: team-a/web-5bc746474f-49sxr:41918 (ID:31476) <> 65.108.32.164:10250 (host) policy-verdict:L3-Only EGRESS DENIED (TCP Flags: SYN)
   1  1 09:43:32.351: team-a/web-5bc746474f-49sxr:41918 (ID:31476) <> 65.108.32.164:10250 (host) Policy denied by denylist DROPPED (TCP Flags: SYN)
```
