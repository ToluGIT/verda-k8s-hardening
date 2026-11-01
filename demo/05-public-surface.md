# Demo 05: public surface

Date: 2026-11-01T09:45Z. From my laptop (149.241.162.239, the one allow-listed admin IP) against cp-1. The only ports that answer are 22 and 6443; from any other address 6443 is closed too (the ADMIN set in infra/nftables.conf).

```
# from the admin IP
  22     open
  6443   open
  10250  closed/filtered
  2379   closed/filtered
  2380   closed/filtered
  8472   closed/filtered
  4240   closed/filtered
  4244   closed/filtered
  9100   closed/filtered
  30080  closed/filtered

# the rule set on cp-1
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

# packets dropped so far on cp-1 (kernel log, nft-drop prefix), by destination port
  2519
     1736 DPT=443
      231 DPT=67
      193 DPT=5678
       27 DPT=80
       12 DPT=5353
       12 DPT=23
```

Open from the internet: 22 (key-only SSH). Open from the admin IP: 6443. Everything else, including the kubelet, etcd and the VXLAN port, is reachable only from the other two nodes.
