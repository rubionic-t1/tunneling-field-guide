# Tunneling Field Guide

**Author:** Rubab Fatima (rubionic-t1)  
**Version:** v1.1  
**Purpose:** A practical, auditable, no-nonsense tunneling guide for pentesters, red teamers, IR, and SecOps.

> **If you can reach the bastion, you can reach what it can see.**  
> **Tunnels are plumbing. Logs are proof.**  
> **The commands are easy — the discipline is the skill.**

---

## 📌 Table of Contents

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

## 🎯 Guiding Principles

- **No ticket, no tunnel** - If you can't show who approved it, don't run anything
- **Logs are your proof** - Your job is to make logs defensible, not assume they exist
- **Tunnels are temporary plumbing** - Tear down when done, document everything
- **Expect logging** - Verify it exists and capture it
- **Communicate proactively** - SOC should never be surprised by your traffic

---

## ⚡ 10-Second Decision Tree

| Scenario | Tool | Command |
|----------|------|---------|
| **One internal service** | `ssh -L` | `ssh -N -L 127.0.0.1:8443:target:8443 user@bastion` |
| **Browse many hosts (Burp)** | `ssh -D` (SOCKS) | `ssh -N -D 127.0.0.1:1080 user@bastion` |
| **Target must call out / NAT exists** | `ssh -R` or Ligolo/Chisel | `ssh -N -R 9000:127.0.0.1:3000 user@bastion` |
| **SSH forwarding blocked** | Chisel via proxy | `chisel client --proxy http://proxy.corp:8080 server.com:8080 R:8888:target:3389` |

---

## 🔒 Before You Tunnel (Real-World First Step)

**Complete this checklist BEFORE establishing any tunnel:**

- [ ] Engagement ticket/SOW exists with tunneling authorization
- [ ] Named approver identified and reachable during tunnel window
- [ ] Scope explicitly defined (IPs, ports, protocols)
- [ ] **SOC notified** if required by client policy
- [ ] Logging destination confirmed (`/var/log/secure`, `auth.log`, etc.)
- [ ] Teardown time agreed upon
- [ ] Fallback plan documented if SSH forwarding is blocked

---

## 📋 Ticket Checklist

**Paste this into your engagement ticket:**

```

## 📄 Evidence & Report Template (Paste-Ready)

**Use this template for all tunnel documentation in reports and tickets:**

```
### TUNNEL ACTIVITY - EVIDENCE SUMMARY

**Purpose**: Vendor admin access / Internal testing / IR data collection  
**Ticket**: INC-12345 | **Approver**: Jane Doe (Security Manager)  
**Window**: 2023-10-05T14:00–16:00 UTC  
**Operator**: [Your Name]

**Command Executed**:
```bash
ssh -N -L 127.0.0.1:8443:10.1.5.20:8443 user@bastion
```

**Evidence Collected**:
- ✅ Ticket screenshot (ID & approver visible)
- ✅ Local listener: `ss -tnlp | grep 8443` → `LISTEN 0 128 127.0.0.1:8443`
- ✅ Service connectivity: `curl -Ik https://127.0.0.1:8443/` → HTTP/1.1 200 OK
- ✅ Bastion auth.log session start: `Accepted publickey for user from [IP]`
- ✅ Bastion auth.log session end: `Received disconnect from [IP]`
- ✅ Teardown confirmed: 2023-10-05T16:00 UTC

**Archive Path**: `/engagements/client/logs/tunnel_20231005_1400/`
- `tunnel_command.txt` - Exact command executed
- `local_verification.txt` - ss/curl output
- `bastion_logs.txt` - Session start/stop logs
- `ticket_approval.png` - Approval screenshot

**Signed**: 
[Operator Name] / [Approver Name]
```

### Daily Status Template
```
**Tunnel Activity Summary - 2023-10-05**
- Established SSH tunnel through bastion-02 to app-server-05:8443
- Purpose: Vendor application testing (INC-12345)
- Window: 14:00-16:00 UTC
- Status: Tunnel torn down at 16:00 UTC as scheduled
- Logs archived: /engagements/client/logs/tunnel_20231005_1400
- No incidents or unexpected alerts
```

### Quick Evidence Capture Commands
```bash
# Capture local listener state
ss -tnlp | grep <PORT> > local_listener_$(date +%Y%m%d_%H%M).txt

# Test service connectivity  
curl -Ik https://127.0.0.1:8443/ > service_test_$(date +%Y%m%d_%H%M).txt

# Capture bastion logs (if accessible)
ssh user@bastion "sudo grep $(whoami) /var/log/auth.log" > bastion_logs_$(date +%Y%m%d_%H%M).txt

# Verify tunnel teardown
ps aux | grep ssh > process_check_$(date +%Y%m%d_%H%M).txt
ss -tnlp | grep <PORT> > listener_check_$(date +%Y%m%d_%H%M).txt
```

---

## 🔍 Evidence Checklist

**Capture this for every tunnel session:**

- [ ] Ticket screenshot with ID and approver visible
- [ ] Local listener verification: `ss -tnlp | grep <port>`
- [ ] Service connectivity test: `curl -Ik`, `nc -zv`, or equivalent
- [ ] Bastion authentication logs (session start)
- [ ] Bastion session logs (activity if available)
- [ ] Teardown verification (session end logs)
- [ ] Archive path documented: `/engagements/<client>/logs/tunnel_YYYYMMDD_HHMM`

---

## 🛠️ Core Tunneling Recipes

### 1. SSH Local Forward (Single Service)
```
Vendor laptop (browser)
┌────────────────────────────────────────────────────────┐
│ https://127.0.0.1:8443  ──▶ 127.0.0.1:8443 (local sock)│
└────────────────────────────────────────────────────────┘
                       │
                       ▼
                SSH client (your laptop)
                       │ (encrypted SSH tunnel)
                       ▼
                Bastion / Jump host
                       │
                       ▼
                Internal App 10.1.5.20:8443
```

```bash
ssh -N -L 127.0.0.1:8443:10.1.5.20:8443 user@bastion
```
**Use case**: Vendor access to specific internal service

### 2. SSH Dynamic SOCKS (Multi-Service Browsing)
```
Browser -> Burp (127.0.0.1:8080)
    │
    ▼
Burp → SOCKS proxy (127.0.0.1:1080)
    │
    ▼
SSH client → SSH tunnel → Bastion → internal web servers
```

```bash
ssh -N -D 127.0.0.1:1080 -C -q user@bastion
```
**Burp Config**: SOCKS proxy → `127.0.0.1:1080`  
**Firefox**: `network.proxy.socks_remote_dns = true`

### 3. SSH Reverse Tunnel (Callback)
```
Your local service (127.0.0.1:3000)
   ↑
ssh -R 9000:127.0.0.1:3000  (your side)  ──▶  Bastion:9000 listens
                                               │
                                               ▼
                                         curl http://127.0.0.1:9000  (on bastion)
```

```bash
ssh -N -R 9000:127.0.0.1:3000 user@bastion
```
**Verify**: On bastion: `ss -tlnp | grep 9000`

### 4. Chisel (SSH Forwarding Blocked)
```
Your public server (chisel server) :8080
   ▲
   │  outbound CONNECT via corporate proxy
   │
Corporate Proxy (proxy.corp:8080)
   │
Internal machine (chisel client) - reverse map:
R:8888:10.1.5.20:3389
```

```bash
# Server (your VPS)
chisel server -p 8080 --reverse

# Client (through corporate proxy)
chisel client --proxy http://proxy.corp:8080 your-server.com:8080 R:8888:10.1.5.20:3389
```

### 5. Ligolo (Outbound-Only Access)
```
Agent (target) ──TLS outbound──▶ your-server:443 (ligolo)
                        ▲
                    Ligolo server
                        │
                        ▼
            multiplexed channels → your local ports
```

```bash
# Server (your VPS)
./proxy -selfcert -laddr 0.0.0.0:443

# Agent (target)
.\agent.exe -connect your-server.com:443 -ignore-cert
```

---

## 🔧 What -N and -R Actually Do

### The `-N` Flag
```bash
ssh -N -L 127.0.0.1:8443:10.1.5.20:8443 user@bastion
```
- **Creates forwarding only** - no remote shell session
- **More ephemeral** - easier to track and tear down
- **Recommended** for all tunnel-only connections

### The `-R` Flag & GatewayPorts
```bash
# Default (loopback only on remote)
ssh -N -R 9000:127.0.0.1:3000 user@bastion

# With GatewayPorts (binds to all interfaces)
ssh -N -R *:9000:127.0.0.1:3000 user@bastion
```
**Warning**: `GatewayPorts` exposes the port to the entire network. Use only with explicit approval and firewall restrictions.

---

## 🎙️ SOC Interaction Scripts

### For SSH Forwarding Alerts
> "Authorized tunneling for engagement ENG-XXX. Ticket INC-12345, approved by [Name]. Session logs available on request. Would you like me to pause for review?"

### For Ligolo/Chisel Alerts  
> "Technique authorized under ROE section 4.2 for outbound-only scenarios. I can provide artifact hashes and pause immediately if needed."

### Daily Status Update
> "Established SSH tunnel through bastion-02 to app-server-05 for vendor access (INC-12345). Tunnel active from 14:00-16:00 UTC. Logs archived at /engagements/client/logs/tunnel_20231005."

---

##  Drills (Skill Builders)

### Vendor Drill (15 minutes)
1. Establish SSH local forward to test service
2. Verify connectivity with `curl -Ik https://127.0.0.1:8443/`
3. Capture: `ss -tnlp`, bastion auth logs, teardown confirmation
4. Build complete evidence package

### SOCKS Drill (20 minutes)  
1. Create SOCKS proxy with `ssh -D`
2. Configure Burp to use proxy
3. Intercept traffic from 3 different internal hosts
4. Export and archive Burp history

### Lockdown Drill (25 minutes)
1. Set up Chisel server on VPS
2. Configure client through simulated corporate proxy
3. Map RDP service through tunnel
4. Collect and document all connection logs

### Cleanup Drill (5 minutes)
1. Create two different tunnel types
2. Identify with `ps aux | grep ssh` and `ss -tnlp`
3. Kill all tunnel processes
4. Verify no listeners remain

---

## 💡 Practical Notes & Gotchas

### SSH Configuration Issues
```bash
# Check if forwarding is allowed
ssh -v user@bastion
# Look for: channel_setup_fwd_listener: denied

# If blocked, ask ops to run:
ssh -N -L 127.0.0.1:8443:target:8443 user@bastion
```

### Common Problems & Solutions
- **Background tunnels forgotten**: Always document `-f` tunnels immediately
- **GatewayPorts not enabled**: Use SSH jump host or Chisel as fallback
- **Corporate proxy blocking**: Chisel with `--proxy` flag often works
- **Session timeouts**: Use `autossh` for persistent tunnels (with approval)

### Security Considerations
- **Never** connect client environments directly to hosts you don't own
- **Always** use loopback binding (`127.0.0.1`) unless explicitly authorized
- **Assume all tunnels are logged** - make your activity defensible
- **Clean up immediately** after testing window ends

---

## 📖 Appendix

### Quick Reference Flags
| Flag | Purpose | Example |
|------|---------|---------|
| `-L` | Local forward | `-L local:remote` |
| `-D` | SOCKS proxy | `-D 127.0.0.1:1080` |
| `-R` | Remote forward | `-R remote:local` |
| `-N` | No remote command | Tunnel only |
| `-f` | Background | Run in background |
| `-C` | Compression | Useful for slow links |

### Network Diagrams
```
# Local Forward
You → SSH Tunnel → Bastion → Internal Service

# SOCKS Proxy  
Browser → Burp → SOCKS → SSH Tunnel → Bastion → Multiple Services

# Reverse Tunnel
Your Service ← SSH Reverse ← Bastion Listener ← Internal Client

# Chisel/Ligolo
Internal Host → Outbound TLS → Your Server → You
```

### One-Liner for Ticket Requests
> "Request: Allow local SSH forward - `ssh -N -L 127.0.0.1:8443:10.1.5.20:8443 user@bastion` - 2-hour window. If run on bastion, paste exact command and teardown confirmation. Logs to: /engagements/client/logs/tunnel_date."

---

## 🎯 Final Rule

> **Tunnels are professional plumbing, not magic.**  
> The difference between repeat work and incidents isn't technical skill—it's process discipline.  
> No ticket = no tunnel. Don't be the incident.

**Proof beats posture, every time.**

