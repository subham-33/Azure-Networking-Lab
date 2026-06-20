# 03 — UDR & IP Forwarding

## Background: How Azure Routes Traffic by Default

Every Azure VNet has a built-in system route table that is invisible in the
portal but always active. By default it contains:

| Destination | Next Hop | Meaning |
|---|---|---|
| 10.20.0.0/16 | VNet local | All VNet traffic stays within the VNet |
| 0.0.0.0/0 | Internet | Everything else goes to the internet directly |

The `0.0.0.0/0 → Internet` default route is the problem for our lab.
If private VMs can reach the internet directly via Azure's default gateway,
they bypass the JumpBox entirely — our NAT setup is pointless.

A User Defined Route (UDR) overrides this default.

---

## What is a User Defined Route?

A UDR is a custom route you define inside a Route Table resource and then
associate with a subnet. When a packet's destination matches a UDR, Azure
uses your custom next hop instead of the system default.

Azure uses **longest prefix match** to pick between routes. A more specific
route always wins over a less specific one, regardless of whether it is a
system route or a UDR.

Example:

```
System route:  0.0.0.0/0    → Internet    (least specific — matches everything)
UDR:           0.0.0.0/0    → 10.20.2.4   (same prefix — UDR wins over system)
System route:  10.20.0.0/16 → VNet local  (more specific — always wins for VNet traffic)
```

So traffic to other VMs (10.20.x.x) still goes through the VNet directly.
Only internet-bound traffic is redirected to the JumpBox.

---

## What is IP Forwarding?

By default, Azure's SDN layer drops any packet that arrives at a NIC whose
destination IP does not match that NIC's own IP address. This is a security
measure — a VM should not be accepting and forwarding traffic meant for
someone else.

Enabling IP Forwarding on the JumpBox's NIC tells Azure's SDN:
> "This NIC is allowed to receive and pass along packets even if the
> destination IP is not its own."

This must be enabled at two levels:
1. **Azure portal (NIC level)** — tells Azure's SDN layer to stop dropping
   foreign-destination packets at the virtual switch
2. **Linux kernel (OS level)** — tells the OS to route packets between
   interfaces (covered in the iptables doc)

Both must be enabled. Azure IP Forwarding alone doesn't make Linux route
the packets. Linux ip_forward alone doesn't help because Azure drops the
packets before they reach Linux.

---

## Creation Steps

### Step 1 — Enable IP Forwarding on the JumpBox NIC

Portal → `vm-jumpbox` → Networking → Network Interface (click the NIC name)
→ IP Configurations → IP Forwarding toggle → **Enabled** → Save

The change takes effect immediately. No VM restart needed.

Screenshot: `screenshots/03-ip-forwarding-enabled.png`

---

### Step 2 — Create the Route Table

Portal → Create a resource → Route Table

| Field | Value |
|---|---|
| Name | rt-private-subnet |
| Region | Central India (same as VNet) |
| Resource Group | rg-private-lab |
| Propagate gateway routes | No |

Click **Review + Create → Create**.

---

### Step 3 — Add the Route

Portal → `rt-private-subnet` → Routes → Add

| Field | Value |
|---|---|
| Route name | route-internet-via-jumpbox |
| Destination type | IP Addresses |
| Destination IP | 0.0.0.0/0 |
| Next hop type | Virtual appliance |
| Next hop address | 10.20.2.4 |

Click **Add**.

### Why "Virtual Appliance" as the next hop type?

Azure uses the term Virtual Appliance for any VM acting as a network
device — router, firewall, NAT box, or proxy. Even though the JumpBox
is a standard Ubuntu VM, it qualifies when used as a NAT router.

This next hop type also enforces a requirement: the target VM's NIC must
have IP Forwarding enabled. If it doesn't, Azure will accept the route
but packets will still be dropped at the NIC. This is why Step 1
(enabling IP Forwarding) must come before this step.

---

### Step 4 — Associate the Route Table with snet-private ONLY

Portal → `rt-private-subnet` → Subnets → Associate

| Field | Value |
|---|---|
| Virtual Network | vnet-lab |
| Subnet | snet-private |

Click **OK**.

### Critical: Do NOT associate with snet-jumpbox

This was the first major mistake made in this lab (see troubleshooting doc).

Route tables apply to every VM in a subnet. If you associate this route
table with `snet-jumpbox`, the JumpBox's own traffic also gets routed to
10.20.2.4 (itself) for internet-bound packets. The JumpBox enters a
routing loop and SSH access dies instantly.

The fix is simple: only associate the route table with the subnet that
contains the private VMs. The JumpBox's subnet must have no route table,
so its traffic uses Azure's default internet route.

Screenshot: `screenshots/03-route-table-config.png`
Screenshot: `screenshots/03-subnet-association.png`

---

## How Traffic Flows After This Configuration

```
vm-private-01 wants to reach 8.8.8.8 (Google DNS)
        │
        │  Packet: [SRC: 10.20.1.5] [DST: 8.8.8.8]
        │
        ▼
Azure SDN checks route table for snet-private:
  - 10.20.0.0/16? No match (8.8.8.8 is not in this range)
  - 0.0.0.0/0? Match → Next hop: Virtual Appliance → 10.20.2.4
        │
        ▼
Packet delivered to JumpBox NIC (10.20.2.4)
Azure SDN allows it because IP Forwarding is ON
        │
        ▼
Linux kernel receives it (ip_forward=1)
iptables MASQUERADE rewrites source IP
        │
        ▼
Packet exits JumpBox's eth0:
  [SRC: JumpBox public IP] [DST: 8.8.8.8]
        │
        ▼
Google receives it, replies to JumpBox public IP
iptables connection tracking reverses the NAT
Reply delivered to vm-private-01
```

---

## Verification

From inside a private VM:

```bash
# Test routing — should NOT time out
ping 8.8.8.8

# Confirm NAT is working — should show JumpBox's PUBLIC IP, not the VM's private IP
curl ifconfig.me
```

If `curl ifconfig.me` returns the JumpBox's public IP, the full chain is
working: UDR → IP Forwarding → iptables NAT.

From the JumpBox, confirm packets are being forwarded:

```bash
sudo iptables -L FORWARD -v -n
# pkts counter should be non-zero after the private VM makes requests
```

Screenshot: `screenshots/03-curl-ifconfig-output.png`
