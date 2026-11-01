# Worklog

Written as I go. Dates are UK time.

## Sun 1 Nov
- Read the brief again and the notes from the call with Vishal. Decided on kubeadm over RKE2 so the control plane
  flags are mine to set and explain. Cilium for the CNI, Argo CD for everything after the CNI.
- Keeping scripts to a minimum. Commands I ran go in docs/runbook.md as run.
- Verda console: Turin 4-core in FIN-03 said "available" then refused with "not enough resources". Rome+ 4-core
  (CPU.4V.16G) in FIN-01 worked. FIN-01 has no private networking, so public IPs only. $0.062/h per node.
- First `kubeadm init` failed at wait-control-plane. kubelet log: "Failed to start ContainerManager ... invalid
  kernel flag: vm/overcommit_memory expected 1 actual 0, kernel/panic expected 10, kernel/panic_on_oops expected 1".
  That is `protectKernelDefaults: true` doing its job. Set the three sysctls in /etc/sysctl.d/k8s.conf, reset, init again.
  Kept the flag; it is the kind of control the brief is asking for.
