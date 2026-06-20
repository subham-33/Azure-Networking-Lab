# 02 — Network Security Group Configuration

## What is an NSG?

A Network Security Group is Azure's layer-4 stateful firewall. It contains
inbound and outbound rules that allow or deny traffic based on:

- Source IP / range
- Destination IP / range
- Port or port range
- Protocol (TCP, UDP, Any)
- Direction (Inbound / Outbound)

NSGs can be attached at two levels:
- **NIC level** — applies to one specific VM
- **Subnet level** — applies to every VM in the subnet

In this lab we attach NSGs at the **NIC level** because the JumpBox and
private VMs need different rules, and they share the same subnet structure
concept. Per-NIC attachment gives us exact per-VM control.

---

## How Rules are Evaluated

Each rule has a **priority number** between 100 and 4096.
Lower number = evaluated first. The first matching rule wins.
Azure stops evaluating after the first match — subsequent rules are ignored
for that packet.

Azure adds three default rules to every NSG that cannot be deleted:

| Priority | Name | Effect |
|---|---|---|
| 65000 | AllowVnetInBound | Allows all traffic between resources in the same VNet |
| 65001 | AllowAzureLoadBalancerInBound | Allows Azure health probes |
| 65500 | DenyAllInBound | Blocks everything else not already allowed |

This means: unless you explicitly allow traffic, it is denied.

---

## Stateful Firewall — What It Means

NSGs are stateful. If an inbound connection is allowed, the return
(outbound) traffic is automatically allowed without needing an explicit
outbound rule. You only need to write rules for the direction that
initiates the connection.

Example: You allow inbound SSH (port 22) from your home IP. When the
JumpBox sends SSH reply packets back to you, Azure automatically allows
those replies — no outbound rule needed.

---

## NSG 1 — nsg-jumpbox

Attached to: JumpBox VM's NIC

### Inbound Rules

| Priority | Name | Port | Protocol | Source | Action | Reason |
|---|---|---|---|---|---|---|
| 100 | Allow-SSH-Home | 22 | TCP | Your home public IP | Allow | Only you can SSH in |
| 110 | Allow-AptCache-VNet | 3142 | TCP | 10.20.0.0/16 | Allow | Private VMs reach apt-cacher-ng |
| 120 | Allow-PrivateSubnet-Forwarding | Any | Any | 10.20.1.0/24 | Allow | Forwarded internet traffic from private VMs |

### Why restrict SSH to your home IP?

Port 22 on a public IP gets targeted by automated bots within minutes of VM
creation. Leaving source as `Any` means your VM faces thousands of
brute-force login attempts per day. Restricting to your home public IP
eliminates this entirely. Find your public IP at `curl ifconfig.me` from
any terminal.

### Why is the forwarding rule needed?

This was discovered as a real bug during this lab (see troubleshooting doc).

When a private VM sends internet-bound traffic, it is routed to the JumpBox
via the UDR. That traffic arrives at the JumpBox's NIC as inbound packets.
The NSG evaluates all inbound traffic — including packets being forwarded
through the VM, not just packets destined for the VM itself.

Without rule 120, the NSG blocks all forwarded traffic from the private subnet
before the Linux kernel ever sees it. The iptables FORWARD chain shows 0
packets and no internet works on the private VMs, even though iptables and
ip_forward are configured correctly.

Adding rule 120 (source: `10.20.1.0/24`, port: Any) allows the forwarded
traffic to pass through the NIC to the kernel, where iptables then NATs it.

### Outbound Rules

No custom outbound rules needed. Azure's default (priority 65001,
AllowInternetOutBound) allows the JumpBox to reach the internet for
package downloads, NAT forwarding, etc.

---

## NSG 2 — nsg-private

Attached to: Each private VM's NIC (vm-private-01, 02, 03)

### Inbound Rules

| Priority | Name | Port | Protocol | Source | Action | Reason |
|---|---|---|---|---|---|---|
| 100 | Allow-SSH-From-JumpBox | 22 | TCP | 10.20.2.4 | Allow | Only JumpBox can SSH into private VMs |

Azure's default DenyAllInBound (priority 65500) blocks everything else.
No additional rules needed.

### Why this is secure

Private VMs have no public IP. There is no direct route from the internet
to their private IPs — they simply cannot be reached from outside the VNet.

The only machine that can initiate SSH to a private VM is 10.20.2.4
(the JumpBox). And the only machine that can reach the JumpBox is your
home IP. This creates two independent security layers:

```
Internet → blocked (no public IP on private VMs)
Your home → JumpBox (port 22, home IP only)
JumpBox → Private VMs (port 22, 10.20.2.4 only)
```

This is called defence in depth — each layer independently protects
the private VMs even if another layer were compromised.

### Outbound Rules

No custom outbound rules. Azure's default allows all outbound — private VMs
need to send packets to the JumpBox (10.20.2.4) for NAT forwarding, and
this is outbound traffic from their perspective, which is allowed by default.

---

## Attaching NSGs to NICs

After creating each VM:

Portal → VM → Networking → Network Interface (click the NIC name)
→ Network Security Group → Edit → Select the relevant NSG → Save

The VM does not need to be restarted. NSG changes take effect within seconds.

---

## Verification

To confirm NSG rules are working correctly, use the
**Network Watcher → IP Flow Verify** tool in Azure:

Portal → Network Watcher → IP Flow Verify

Test 1: Source = your home IP, Dest = JumpBox public IP, Port 22 → Should show **Allow**
Test 2: Source = 8.8.8.8 (random internet), Dest = JumpBox public IP, Port 22 → Should show **Deny**
Test 3: Source = 10.20.2.4, Dest = 10.20.1.5, Port 22 → Should show **Allow**

Screenshot: `screenshots/02-nsg-jumpbox-rules.png`
Screenshot: `screenshots/02-nsg-private-rules.png`
