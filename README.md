# Azure-Networking-Lab
Azure Networking Lab — Private VM Isolation, IP Forwarding &amp; iptables NAT

A hands-on Azure networking lab demonstrating private VM isolation,
IP forwarding, iptables NAT, and UDR-based routing.

## Architecture

![Architecture Diagram](architecture.png)

## What this lab covers

- VNet and subnet design (isolating JumpBox from private VMs)
- Network Security Groups (per-NIC rules, layered defence)
- Static private IP assignment
- Azure IP Forwarding on a NIC
- Linux iptables MASQUERADE (NAT)
- User Defined Routes — overriding Azure's default gateway
- Jumpbox / bastion host pattern

## Lab topology

| Resource | Value |
|---|---|
| VNet | 10.20.0.0/16 |
| JumpBox subnet | 10.20.2.0/24 |
| Private VM subnet | 10.20.1.0/24 |
| JumpBox private IP | 10.20.2.4 (static, IP Forwarding ON) |
| Private VMs | 10.20.1.5 / .6 / .7 — no public IP |

## Docs

- [VNet & Subnet Setup](docs/01-vnet-setup.md)
- [NSG Configuration](docs/02-nsg-config.md)
- [UDR & IP Forwarding](docs/03-udr-ip-forwarding.md)
- [iptables NAT](docs/04-iptables-nat.md)
- [Troubleshooting & Issues Found](docs/05-troubleshooting.md)

## Key troubleshooting lessons

Two real issues hit during this lab and resolved — see
[docs/05-troubleshooting.md](docs/05-troubleshooting.md)

## Tools used

Azure Portal · Ubuntu 24.04 · iptables · sysctl

## Status

✅ Completed and verified — private VMs reach internet via JumpBox NAT
