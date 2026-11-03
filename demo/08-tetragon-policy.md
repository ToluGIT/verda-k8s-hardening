# Demo 08: runtime detection with a Tetragon policy

Date: 2026-11-01T13:33Z. runtime/tetragon/sensitive-file-access.yaml watches reads and writes of /etc/shadow, /etc/sudoers, the kubelet and control-plane PKI directories and /root/.ssh, on the host and in every container. Delivered by Argo CD like everything else. Observe-only.

```
$ kubectl -n kube-system exec ds/tetragon -c tetragon -- tetra tracingpolicy list
ID   NAME                    DOMAIN   STATE     FILTERID   NAMESPACE   SENSORS          KERNELMEMORY   MODE           NPOST   NENFORCE   NMONITOR
2    sensitive-file-access   k8s      enabled   0          (global)    generic_kprobe   698.80 kB      monitor_only   0       0          0

# on worker-1, as root over SSH: try the kubelet key (not present at that path on this kubeadm layout) and read the shadow file. Then, inside team-a/web-5bc746474f-49sxr, try the same.
$ ssh root@65.108.32.164 'cat /var/lib/kubelet/pki/kubelet.key > /dev/null && echo "host: read kubelet.key as root"; cat /etc/shadow > /dev/null && echo "host: read /etc/shadow as root"'
cat: /var/lib/kubelet/pki/kubelet.key: No such file or directory
host: read /etc/shadow as root
$ kubectl -n team-a exec web-5bc746474f-49sxr -- cat /etc/shadow
cat: /etc/shadow: Permission denied

# tetra getevents -o compact on worker-1 during those two commands
📚 read    worker-1 /usr/sbin/sshd /etc/shadow
📚 read    worker-1 /usr/sbin/sshd /etc/shadow
📚 read    worker-1 /proc/self/fd/9 /etc/shadow
📚 read    worker-1 /proc/self/fd/9 /etc/shadow
🚀 process worker-1 /bin/bash "-c \"cat /var/lib/kubelet/pki/kubelet.key > /dev/null && echo \"host: read kubelet.key as root\"; cat /etc/shadow > /dev/null && echo \"host: read /etc/shadow as root\"\"
🚀 process worker-1 /usr/bin/cat /var/lib/kubelet/pki/kubelet.key
💥 exit    worker-1 /usr/bin/cat /var/lib/kubelet/pki/kubelet.key 1
🚀 process worker-1 /usr/bin/cat /etc/shadow
📚 read    worker-1 /usr/bin/cat /etc/shadow
📚 read    worker-1 /usr/bin/cat /etc/shadow
```

Three things in that output. The SSH login itself reads /etc/shadow (sshd, PAM), which is the baseline you have to learn before alerting on this. The root read of /etc/shadow is recorded as a file read event with host, binary and path. And the attempt from the tenant pod produced only the exec of cat, no file event: the kernel refused the open before any read happened, and this policy hooks the read/write permission check, not the open. Closing that gap is a second kprobe on security_file_open, which I left out so the policy stays a copy of the upstream example with narrower paths. It is also why this runs in observe mode for a while before anyone is paged on it.
