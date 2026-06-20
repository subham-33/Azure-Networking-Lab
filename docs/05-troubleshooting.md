# Troubleshooting log

## Issue 1 — SSH to JumpBox died after associating route table

**What happened:** Associated the route table (0.0.0.0/0 → JumpBox)
with the subnet that contained the JumpBox itself.
The JumpBox's own reply packets were being routed back through
itself — a loop. SSH connection died instantly.

**Diagnosis:** Realised the UDR applies to ALL VMs in the subnet,
including the JumpBox.

**Fix:** Moved the JumpBox to a separate subnet (snet-jumpbox,
10.20.2.0/24). Applied the route table only to snet-private
(10.20.1.0/24). JumpBox's subnet has no route table.

**Lesson:** The gateway/router VM must never be in the same subnet
as the VMs routing through it.

---

## Issue 2 — Private VMs still couldn't reach internet after fix

**What happened:** iptables MASQUERADE rule was in place,
ip_forward=1, ufw inactive — but `ping 8.8.8.8` from private
VM timed out. FORWARD chain showed 0 packets.

**Diagnosis:** 0 packets in FORWARD meant traffic was never
reaching the JumpBox at all. The NSG on the JumpBox NIC only
allowed port 22 (SSH from home) and port 3142 (apt-cache).
All other inbound traffic from private VMs was blocked by
the NSG before the Linux kernel ever saw it.

**Fix:** Added NSG inbound rule on nsg-jumpbox:
Source 10.20.1.0/24, Port Any, Action Allow.

**Lesson:** Azure NSGs evaluate ALL inbound traffic on a NIC —
including forwarded packets. When a VM acts as a virtual
appliance, its NSG must allow the forwarded traffic through,
not just management ports.
