# 01 — VNet & Subnet Setup

## What is a VNet?

A Virtual Network (VNet) is a logically isolated network inside Azure. It is
completely private — nothing enters or leaves unless you explicitly allow it.
Think of it as your own private data centre network, but software-defined and
managed by Azure's infrastructure.

No two customers share a VNet. It doesn't matter if someone else in Azure uses
the same address range as you — Azure's SDN layer keeps all VNets fully
isolated from each other.

---

## Address Space Planning

Before creating anything in the portal, plan your IP ranges on paper.
Overlapping address spaces break routing — Azure cannot decide which path to
send traffic on if two networks share the same range.

| Resource | CIDR | Notes |
|---|---|---|
| VNet | 10.20.0.0/16 | Parent range — 65,536 IPs |
| snet-jumpbox | 10.20.2.0/24 | JumpBox only — no route table |
| snet-vms | 10.20.1.0/24 | Private VMs — route table applied here |

### Why 10.20.0.0/16?

It doesn't overlap with common home/VMware ranges (192.168.x.x, 172.16.x.x).
The /16 gives us room to carve multiple /24 subnets without running out of
space — useful as the lab grows.

### Understanding CIDR notation

The `/` number is the subnet mask expressed in bits.

- `/16` = 65,536 total IPs. Used for the VNet (parent container).
- `/24` = 256 total IPs, 251 usable after Azure's reservations.

The VNet's address space must always be larger than the subnets inside it.
A /24 subnet cannot exist inside a /28 VNet.

### Azure's 5 reserved IPs per subnet

Azure takes 5 IPs from every subnet automatically. You cannot assign these
to VMs:

| Reserved IP | Purpose |
|---|---|
| .0 | Network address |
| .1 | Default gateway (Azure managed) |
| .2 | Azure DNS |
| .3 | Azure DNS |
| .255 | Broadcast |

So a /24 subnet gives you 256 − 5 = **251 usable IPs**.
The first assignable IP is always `.4`.

---

## Creation Steps

### Step 1 — Create the Resource Group

Portal → Resource Groups → Create

| Field | Value |
|---|---|
| Name | rg-private-lab |
| Region | Central India |

A resource group is a logical container — every Azure resource must live
inside one. When the lab is done, deleting the resource group removes
everything inside it in one operation. Never delete resources one by one
in a lab environment.

---

### Step 2 — Create the VNet

Portal → Create a resource → Virtual Network

**Basics tab**

| Field | Value |
|---|---|
| Resource Group | rg-private-lab |
| Name | vnet-lab |
| Region | Central India |

**IP Addresses tab**

- Delete the default address space Azure pre-fills
- Add: `10.20.0.0/16`
- Delete the default subnet Azure creates
- Add subnet 1:
  - Name: `snet-jumpbox`
  - Range: `10.20.2.0/24`
- Add subnet 2:
  - Name: `snet-vms`
  - Range: `10.20.1.0/24`

Click **Review + Create → Create**.

---

## Why Two Subnets?

This is the most important design decision in the entire lab.

The route table we apply later sends all internet traffic (`0.0.0.0/0`) to
the JumpBox as the next hop. Route tables apply at the **subnet level** — they
affect every VM in that subnet, including the JumpBox itself.

If the JumpBox is in the same subnet as the private VMs, the route table also
applies to it. The JumpBox would try to send its own traffic to itself as
the next hop — a routing loop. SSH access dies immediately.

Separating them into two subnets solves this cleanly:
- `snet-jumpbox` has no route table → JumpBox routes normally to the internet
- `snet-vms` has the route table → private VMs are forced through JumpBox

This is also the correct real-world pattern. In production environments,
bastion hosts and NAT gateways always sit in their own dedicated subnets,
separate from the workloads they serve.

---

## Verification

After creation, confirm the following in the portal:

Portal → `vnet-lab` → Subnets

You should see both subnets listed with their correct address ranges and no
overlapping space.

```
snet-jumpbox    10.20.2.0/24
snet-vms    10.20.1.0/24
```

Screenshot: `screenshots/01-vnet-subnets.png`
