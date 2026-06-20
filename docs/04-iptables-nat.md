# 04 — iptables NAT Configuration

## How to Access the JumpBox

```bash
ssh -i ~/.ssh/your_key azureuser@<pip-jumpbox-public-ip>
```

All commands in this document are run on the JumpBox VM unless stated
otherwise.

---

## Part A — Linux Kernel IP Forwarding (sysctl)

### What is sysctl?

`sysctl` is a tool for reading and modifying Linux kernel parameters at
runtime without rebooting. The kernel exposes its tunable settings through
a virtual filesystem at `/proc/sys/`. Every file in that directory is a
kernel parameter.

`sysctl` is just a cleaner interface to read and write those files.
The two commands below do exactly the same thing:

```bash
# Using sysctl
sudo sysctl -w net.ipv4.ip_forward=1

# Writing directly to the /proc/sys file
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

The dotted name maps directly to the file path — slashes become dots:
```
/proc/sys/net/ipv4/ip_forward
            net.ipv4.ip_forward
```

### What does net.ipv4.ip_forward do?

By default (value = 0), the Linux kernel only processes packets whose
destination IP matches one of its own interfaces. Any other packet is
silently dropped at the kernel level — before iptables even sees it.

Setting it to 1 tells the kernel:
> "If a packet arrives whose destination is not my own IP, but I have
> a route for that destination via another interface — forward it."

This is the OS-level counterpart to Azure's IP Forwarding toggle.
Both must be enabled. Azure's toggle stops the virtual switch from
dropping the packets. This sysctl stops the OS from dropping them.

### Commands

```bash
# Check current value
sysctl net.ipv4.ip_forward
# Output: net.ipv4.ip_forward = 0  (disabled by default)

# Enable at runtime (resets on reboot)
sudo sysctl -w net.ipv4.ip_forward=1

# Make it permanent across reboots
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Apply the config file immediately
sudo sysctl -p

# Verify
sysctl net.ipv4.ip_forward
# Output: net.ipv4.ip_forward = 1
```

---

## Part B — iptables

### What is iptables?

iptables is Linux's built-in packet filtering and NAT framework. It sits
inside the Linux kernel (in the Netfilter subsystem) and intercepts every
packet that enters, leaves, or passes through the machine.

Rules in iptables decide what to do with each packet: accept it, drop it,
reject it, or modify it (such as rewriting IP addresses for NAT).

### Tables and Chains

iptables organises rules into tables, and each table contains chains.

```
Tables:
├── filter   ← Default. Controls what traffic is ALLOWED or DROPPED.
├── nat      ← Handles Network Address Translation (rewriting IPs/ports).
├── mangle   ← Modifies packet headers (TTL etc.). Rarely used.
└── raw      ← Very early processing. Rarely used.
```

Each table has chains — ordered lists of rules that packets flow through:

```
filter table:
├── INPUT      ← Packets destined FOR this machine
├── OUTPUT     ← Packets originating FROM this machine
└── FORWARD    ← Packets passing THROUGH this machine (our NAT scenario)

nat table:
├── PREROUTING    ← Packets arriving, before routing decision
├── POSTROUTING   ← Packets leaving, after routing decision ← we use this
└── OUTPUT        ← Locally generated packets
```

### Packet flow through the system

```
Packet arrives from private VM
        │
        ▼
nat: PREROUTING      (nothing for us here)
        │
        ▼
Routing decision     (is this for me? No → forward it)
        │
        ▼
filter: FORWARD      (is forwarding allowed? policy ACCEPT → yes)
        │
        ▼
nat: POSTROUTING     ← MASQUERADE rule fires here
        │
        ▼
Packet exits through eth0 with rewritten source IP
```

### Rule syntax

```bash
iptables -t [TABLE] -[OPERATION] [CHAIN] [MATCH] -j [TARGET]
```

| Part | Meaning |
|---|---|
| `-t nat` | Which table to work in |
| `-A POSTROUTING` | Append rule to POSTROUTING chain |
| `-L` | List all rules in the chain |
| `-v` | Verbose — shows packet and byte counters |
| `-n` | Numeric — don't reverse-lookup IP addresses |
| `-o eth0` | Match packets going OUT through interface eth0 |
| `-i eth0` | Match packets coming IN through eth0 |
| `-s 10.20.1.0/24` | Match source IP range |
| `-p tcp` | Match protocol |
| `--dport 22` | Match destination port |
| `-j ACCEPT` | Target: accept the packet |
| `-j DROP` | Target: silently drop |
| `-j MASQUERADE` | Target: NAT masquerade (rewrite source IP) |

---

## Part C — The NAT Configuration

### Step 1 — Enable kernel IP forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Step 2 — Add the MASQUERADE rule

First, find the correct network interface name:

```bash
ip link show
# Look for the primary interface — usually eth0 on Azure VMs
```

Then add the rule:

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Breaking this down word by word:
- `-t nat` → work in the NAT table
- `-A POSTROUTING` → append to the POSTROUTING chain
- `-o eth0` → only match packets going out through eth0
- `-j MASQUERADE` → rewrite the source IP to this machine's own IP on eth0

### What MASQUERADE actually does — packet by packet

```
Step 1: vm-private-01 (10.20.1.5) sends a packet
        [SRC: 10.20.1.5]  →  [DST: 142.250.77.14 (Google)]

Step 2: JumpBox receives it via UDR, forwards it to POSTROUTING
        ip_forward=1 lets the kernel route it

Step 3: MASQUERADE fires
        Rewrites source:
        [SRC: 10.20.1.5]  →  [SRC: 20.x.x.x (JumpBox public IP)]
        iptables records the original mapping in a connection tracking table:
        {public_IP:port ↔ 10.20.1.5:port}

Step 4: Packet exits JumpBox
        [SRC: 20.x.x.x]  →  [DST: 142.250.77.14]
        Google sees the JumpBox's public IP as the requester

Step 5: Google replies
        [SRC: 142.250.77.14]  →  [DST: 20.x.x.x]

Step 6: Reply arrives at JumpBox
        iptables checks the connection tracking table
        Finds the mapping: 20.x.x.x:port → 10.20.1.5:port
        Rewrites destination back to the private VM

Step 7: Packet delivered to vm-private-01
        The VM has no idea its IP was rewritten — it just sees the reply
```

This is exactly how your home router works. Your laptop has 192.168.x.x
but the internet sees your ISP's public IP. MASQUERADE is the same
mechanism.

### MASQUERADE vs SNAT

| | MASQUERADE | SNAT |
|---|---|---|
| Source IP | Automatically uses current IP of outgoing interface | You hardcode a specific IP |
| Use case | Dynamic or unknown public IP | Static public IP |
| Performance | Slightly slower (IP lookup per packet) | Slightly faster |
| Command | `-j MASQUERADE` | `-j SNAT --to-source x.x.x.x` |

For this lab, MASQUERADE is simpler and works correctly since Azure
VMs have a stable public IP on their interface.

### Step 3 — Make the rule persistent

iptables rules live in kernel memory — they are lost on reboot:

```bash
sudo apt install iptables-persistent -y
# During install, say Yes to save current rules

# To save manually at any time:
sudo netfilter-persistent save

# Rules are saved to:
# /etc/iptables/rules.v4   (IPv4 rules)
# /etc/iptables/rules.v6   (IPv6 rules)
# Loaded automatically on boot by netfilter-persistent service
```

---

## Part D — Verification Commands

### On the JumpBox

```bash
# Confirm ip_forward is active
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1

# View NAT table — confirm MASQUERADE rule exists
sudo iptables -t nat -L POSTROUTING -v -n
# Expected: one MASQUERADE rule on eth0

# View FORWARD chain — after private VM makes requests, pkts should be non-zero
sudo iptables -L FORWARD -v -n
# Expected: policy ACCEPT, pkts counter > 0 after testing

# Watch NAT in action in real time
sudo iptables -t nat -L -v -n -w 1
```

### From a Private VM

SSH in via JumpBox first:

```bash
# From JumpBox:
ssh azureuser@10.20.1.5

# Test 1: can the VM reach internet by IP?
ping 8.8.8.8
# Expected: replies from 8.8.8.8

# Test 2: confirm traffic is exiting via JumpBox's public IP
curl ifconfig.me
# Expected: shows JumpBox's PUBLIC IP — not the VM's private IP 10.20.1.5
# This is the definitive proof that NAT is working

# Test 3: DNS resolution works
curl https://google.com
# Expected: HTTP redirect response from Google
```

If `curl ifconfig.me` shows the JumpBox's public IP from inside vm-private-01,
the entire chain is confirmed working end to end:

```
UDR → IP Forwarding (Azure) → ip_forward (kernel) → FORWARD chain → MASQUERADE → Internet
```

Screenshot: `screenshots/04-iptables-rules.png`
Screenshot: `screenshots/04-curl-ifconfig-from-private-vm.png`
