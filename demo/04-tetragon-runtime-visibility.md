# Demo 04: runtime visibility with Tetragon

Date: 2026-11-01T09:45Z. Someone gets a shell in team-a/web-5bc746474f-49sxr on worker-1 and does what an attacker does first: who am I, is there a service account token, can I reach the metadata service, what tools are here.

```
# inside the pod
$ kubectl -n team-a exec web-5bc746474f-49sxr -- sh -c 'id; cat /etc/passwd | head -1; ls /var/run/secrets/kubernetes.io 2>&1; wget -q -O- -T 3 http://169.254.169.254/ 2>&1 | head -1; which nc python3'
uid=101(nginx) gid=101(nginx) groups=101(nginx)
root:x:0:0:root:/root:/bin/bash
ls: cannot access '/var/run/secrets/kubernetes.io': No such file or directory
sh: 1: wget: not found
command terminated with exit code 1

# what Tetragon on worker-1 recorded for namespace team-a (tetra getevents -o compact --namespace team-a)
🚀 process team-a/web-5bc746474f-49sxr /usr/bin/sh "-c \"id; cat /etc/passwd | head -1; ls /var/run/secrets/kubernetes.io 2>&1; wget -q -O- -T 3 http://169.254.169.254/ 2>&1 | head -1; which nc python3
🚀 process team-a/web-5bc746474f-49sxr /usr/bin/id 
💥 exit    team-a/web-5bc746474f-49sxr /usr/bin/id  0
🚀 process team-a/web-5bc746474f-49sxr /usr/bin/cat /etc/passwd
🚀 process team-a/web-5bc746474f-49sxr /usr/bin/head -1
💥 exit    team-a/web-5bc746474f-49sxr /usr/bin/head -1 0
💥 exit    team-a/web-5bc746474f-49sxr /usr/bin/cat /etc/passwd 0
🚀 process team-a/web-5bc746474f-49sxr /usr/bin/ls /var/run/secrets/kubernetes.io
💥 exit    team-a/web-5bc746474f-49sxr /usr/bin/ls /var/run/secrets/kubernetes.io 2
💥 exit    team-a/web-5bc746474f-49sxr /usr/bin/sh "-c \"id; cat /etc/passwd | head -1; ls /var/run/secrets/kubernetes.io 2>&1; wget -q -O- -T 3 http://169.254.169.254/ 2>&1 | head -1; which nc python3
🚀 process team-a/web-5bc746474f-49sxr /usr/bin/head -1
💥 exit    team-a/web-5bc746474f-49sxr /usr/bin/head -1 0
🚀 process team-a/web-5bc746474f-49sxr /usr/bin/which "/usr/bin/which nc python3"
💥 exit    team-a/web-5bc746474f-49sxr /usr/bin/which "/usr/bin/which nc python3" 1
```

Every exec in the tenant namespace is recorded with the binary, arguments, pod and parent process, before any policy is written. No service account token is mounted (automountServiceAccountToken: false), the metadata service does not exist on Verda, and the image has no nc or python.
