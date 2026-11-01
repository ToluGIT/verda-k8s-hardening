# Demo 05: public surface

Date: 2026-11-01T19:10Z. Two vantage points: my laptop (149.241.162.239, the one address in the firewall's ADMIN set) and a GitHub Actions runner (.github/workflows/public-surface-probe.yaml), which is an address the firewall has never heard of. Same twelve ports, all three nodes.

```
# from the admin IP. Command per port: nc -z -G 3 <ip> <port>
  cp-1      65.108.33.68    22     open
  cp-1      65.108.33.68    6443   open
  cp-1      65.108.33.68    10250  closed/filtered
  cp-1      65.108.33.68    10256  closed/filtered
  cp-1      65.108.33.68    2379   closed/filtered
  cp-1      65.108.33.68    2380   closed/filtered
  cp-1      65.108.33.68    8472   closed/filtered
  cp-1      65.108.33.68    4240   closed/filtered
  cp-1      65.108.33.68    4244   closed/filtered
  cp-1      65.108.33.68    9100   closed/filtered
  cp-1      65.108.33.68    9153   closed/filtered
  cp-1      65.108.33.68    30080  closed/filtered
  worker-1  65.108.32.164   22     open
  worker-1  65.108.32.164   6443   closed/filtered
  worker-1  65.108.32.164   10250  closed/filtered
  worker-1  65.108.32.164   10256  closed/filtered
  worker-1  65.108.32.164   2379   closed/filtered
  worker-1  65.108.32.164   2380   closed/filtered
  worker-1  65.108.32.164   8472   closed/filtered
  worker-1  65.108.32.164   4240   closed/filtered
  worker-1  65.108.32.164   4244   closed/filtered
  worker-1  65.108.32.164   9100   closed/filtered
  worker-1  65.108.32.164   9153   closed/filtered
  worker-1  65.108.32.164   30080  closed/filtered
  worker-2  65.108.33.95    22     open
  worker-2  65.108.33.95    6443   closed/filtered
  worker-2  65.108.33.95    10250  closed/filtered
  worker-2  65.108.33.95    10256  closed/filtered
  worker-2  65.108.33.95    2379   closed/filtered
  worker-2  65.108.33.95    2380   closed/filtered
  worker-2  65.108.33.95    8472   closed/filtered
  worker-2  65.108.33.95    4240   closed/filtered
  worker-2  65.108.33.95    4244   closed/filtered
  worker-2  65.108.33.95    9100   closed/filtered
  worker-2  65.108.33.95    9153   closed/filtered
  worker-2  65.108.33.95    30080  closed/filtered

# from a GitHub Actions runner (workflow_dispatch, run 41341375261; one line per probe, /dev/tcp with a 4s timeout)
  runner public ip 74.249.180.179
  cp-1      65.108.33.68    22     open
  cp-1      65.108.33.68    6443   closed/filtered
  cp-1      65.108.33.68    10250  closed/filtered
  cp-1      65.108.33.68    10256  closed/filtered
  cp-1      65.108.33.68    2379   closed/filtered
  cp-1      65.108.33.68    2380   closed/filtered
  cp-1      65.108.33.68    8472   closed/filtered
  cp-1      65.108.33.68    4240   closed/filtered
  cp-1      65.108.33.68    4244   closed/filtered
  cp-1      65.108.33.68    9100   closed/filtered
  cp-1      65.108.33.68    9153   closed/filtered
  cp-1      65.108.33.68    30080  closed/filtered
  worker-1  65.108.32.164   22     open
  worker-1  65.108.32.164   6443   closed/filtered
  worker-1  65.108.32.164   10250  closed/filtered
  worker-1  65.108.32.164   10256  closed/filtered
  worker-1  65.108.32.164   2379   closed/filtered
  worker-1  65.108.32.164   2380   closed/filtered
  worker-1  65.108.32.164   8472   closed/filtered
  worker-1  65.108.32.164   4240   closed/filtered
  worker-1  65.108.32.164   4244   closed/filtered
  worker-1  65.108.32.164   9100   closed/filtered
  worker-1  65.108.32.164   9153   closed/filtered
  worker-1  65.108.32.164   30080  closed/filtered
  worker-2  65.108.33.95    22     open
  worker-2  65.108.33.95    6443   closed/filtered
  worker-2  65.108.33.95    10250  closed/filtered
  worker-2  65.108.33.95    10256  closed/filtered
  worker-2  65.108.33.95    2379   closed/filtered
  worker-2  65.108.33.95    2380   closed/filtered
  worker-2  65.108.33.95    8472   closed/filtered
  worker-2  65.108.33.95    4240   closed/filtered
  worker-2  65.108.33.95    4244   closed/filtered
  worker-2  65.108.33.95    9100   closed/filtered
  worker-2  65.108.33.95    9153   closed/filtered
  worker-2  65.108.33.95    30080  closed/filtered

# the rule set on cp-1 (identical on the workers)
  table inet filter {
  	chain input {
  		type filter hook input priority filter; policy drop;
  		iif "lo" accept
  		ct state established,related accept
  		ct state invalid drop
  		ip protocol icmp accept
  		ip6 nexthdr ipv6-icmp accept
  		ip saddr { 65.108.32.164, 65.108.33.68, 65.108.33.95 } accept
  		ip saddr 10.244.0.0/16 accept
  		tcp dport 22 accept
  		ip saddr 149.241.162.239 tcp dport 6443 accept
  		limit rate 5/second burst 5 packets log prefix "nft-drop " drop
  	}
  
  	chain forward {
  		type filter hook forward priority filter; policy accept;
  	}
  
  	chain output {
  		type filter hook output priority filter; policy accept;
  	}
  }

# dropped packets so far on cp-1 (kernel log, nft-drop prefix), by destination port
  42702
    28963 DPT=443
     4433 DPT=67
     3454 DPT=5678
      685 DPT=80
      186 DPT=23
       95 DPT=445
```

From the admin IP: 22 and 6443 answer, nothing else. From the runner: 22 only. The kubelet, etcd, the VXLAN port, node-exporter and CoreDNS metrics are reachable from the other two nodes and from pods, and from nowhere else. The remaining exposure is SSH itself, key-only, and the reviewer key cannot open a shell (README).
