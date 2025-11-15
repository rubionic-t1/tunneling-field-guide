# Tunneling Field Guide

**Author:** Rubab Fatima (rubionic-t1)  
**Version:** v1.1  
**Purpose:** A practical, auditable, no-nonsense tunneling guide for pentesters, red teamers, IR, and SecOps.

> **If you can reach the bastion, you can reach what it can see.**  
> **Tunnels are plumbing. Logs are proof.**  
> **The commands are easy — the discipline is the skill.**

---

# 📌 Table of Contents

1. [Guiding Principles](#guiding-principles)
2. [10-Second Decision Tree](#10-second-decision-tree)
3. [Before You Tunnel (Real-World First Step)](#before-you-tunnel-real-world-first-step)
4. [Ticket Checklist](#ticket-checklist)
5. [Evidence Checklist](#evidence-checklist)
6. [Core Tunneling Recipes](#core-tunneling-recipes)
7. [What -N and -R Actually Do](#what--n-and--r-actually-do)
8. [SOC Interaction Scripts](#soc-interaction-scripts)
9. [Drills (Skill Builders)](#drills-skill-builders)
10. [Practical Notes & Gotchas](#practical-notes--gotchas)
11. [Appendix](#appendix)

---

# Guiding Principles

### 1. No ticket = No tunnel.
If you cannot show an approver, a scope, and a teardown time, do not run anything.

### 2. If you can reach the bastion, you can reach what it can see.
This is the mental model behind all tunneling.

### 3. One tunnel ≠ one service.
Local forward = one service.  
SOCKS = browse everything.  
Reverse = target calls you.

### 4. Logs before, during, after.
Assume every tunnel will be questioned later.  
Make the proof effortless.

---

# 10-Second Decision Tree

| Goal | Use | Command | Why This Matters |
|------|-----|---------|------------------|
| Access one internal service | Local forward | `ssh -N -L 127.0.0.1:<LPORT>:<RIP>:<RPORT> user@bastion` | Clean & firewall-friendly |
| Browse many internal hosts | SOCKS | `ssh -N -D 127.0.0.1:1080 -C -q user@bastion` | Avoid 10+ tunnels |
| Host must call back / NAT / outbound-only | Reverse | `ssh -N -R <RPORT>:127.0.0.1:<LPORT> user@bastion` | When host can't be reached |
| SSH forwarding blocked | Chisel | `chisel client ...` | Strict environments |
| Outbound-only & multiplexing needed | Ligolo | agent + proxy | Red-team grade tunneling |

---

# Before You Tunnel (Real-World First Step)

In labs (OSCP, HTB) you tunnel **from Kali**.

In real life:

- You **SSH into the client bastion**
- You run the tunnel from **your laptop through the bastion**
- The bastion reaches the internal network, *not* your Kali

Real workflow:

```
Your Laptop → SSH → Bastion → Internal Network
```

You do NOT pty-spawn shells on the bastion unless:
- it's a compromised host (red team), or
- you're stuck in a restricted shell.

---

# Ticket Checklist

Every tunnel must include:

- Ticket ID  
- Named Approver  
- Exact Tunnel Command  
- Scope (IPs/ports)  
- TTL / Teardown  
- Log plan & archive path  

Example:

```
INC-4451 | Approver: Jane Doe
Scope: 10.1.5.20:8443
Command: ssh -N -L 127.0.0.1:8443:10.1.5.20:8443 ruby@bastion
TTL: 2 hours
Logs: /engagements/client/logs/tunnel_20251112
```

---

# Evidence Checklist

### 1. Listener
```
ss -tnlp | grep <port>
```

### 2. Service Response
```
curl -Ik https://127.0.0.1:<port>
```

### 3. Bastion Log Proof
```
journalctl -u ssh -S "<start>" -U "<end>" | grep "$(whoami)@"
```

---

# Core Tunneling Recipes

## ⭐ 1. Local Forward
```
ssh -N -L 127.0.0.1:8443:10.1.5.20:8443 user@bastion
```

ASCII:
```
Your Browser → localhost:8443 → SSH → Bastion → Internal:8443
```

---

## ⭐ 2. SOCKS Proxy
```
ssh -N -D 127.0.0.1:1080 -C -q user@bastion
```

Configure:
```
network.proxy.socks_remote_dns=true
```

ASCII:
```
Browser → Burp → SOCKS(1080) → SSH → Bastion → Internal Hosts
```

---

## ⭐ 3. Reverse Tunnel
```
ssh -N -R 9000:127.0.0.1:3000 user@bastion
```

ASCII:
```
Client(local:3000) ← ssh -R 9000 → Bastion:9000
```

---

## ⭐ 4. Background Tunnel
```
ssh -N -f -L 127.0.0.1:8443:10.1.5.20:8443 user@bastion
```

---

## ⭐ 5. Chisel (SSH blocked)

Server:
```
chisel server -p 8080 --reverse
```

Client:
```
chisel client --proxy http://proxy.corp:8080 your-server.com:8080 R:8888:10.1.5.20:3389
```

---

## ⭐ 6. Ligolo
Server:
```
./proxy -selfcert -laddr 0.0.0.0:443
```

Agent:
```
./agent -connect your-server.com:443 -ignore-cert
```

---

# What -N and -R Actually Do

### -N
No shell. Tunnel only.

```
ssh -N -L ...
```

### -R
Remote host listens and sends back to you.

```
ssh -N -R 9000:127.0.0.1:3000 user@bastion
```

Notes:
- `-N -f` backgrounds the tunnel  
- `GatewayPorts` controls remote exposure  

---

# SOC Interaction Scripts

If SOC flags you:
```
"Authorized tunneling for engagement <ENG>. Ticket <INC>. Approved by <Name>. Evidence archived. Want me to pause?"
```

If they ask about Chisel/Ligolo:
```
"Technique approved under ROE 4.2 for outbound-only. I can pause immediately for review."
```

---

# Drills (Skill Builders)

### Vendor Drill (15m)
`ssh -N -L` → ss → curl → journalctl → archive

### SOCKS Drill (20m)
`ssh -D` → browse 3 hosts

### Lockdown Drill (25m)
Chisel → map RDP → collect logs

### Cleanup Drill (5m)
List → kill → verify no listeners

---

# Practical Notes & Gotchas

- Prefer `-N` for pure tunnels  
- Backgrounded `-f` tunnels get forgotten  
- Never expose remote binds without approval  
- Use `ssh -v` for debugging  
- Assume all tunnels are logged  

---

# Appendix

```
systemd-run --user --unit=tunnel5432 --on-active=30m \
  ssh -o ExitOnForwardFailure=yes -N -f \
  -L 5432:db.prod.local:5432 user@bastion
```

```
Your Laptop → SSH → Bastion → Internal Network
```

---


