# Linux Port Debugging — Apache (httpd) on CentOS

> Study notes: Firewall, Port Conflicts, Sendmail, iptables  
> Environment: CentOS (LXC Container) | Jump Host | stapp01/02/03

---

## Context & Environment

The lab consists of an **Ubuntu Host** running several **CentOS containers** (stapp01, stapp02, stapp03). Connection is made via a **Jump Host**.

```
Ubuntu Host
└── LXC Containers
    ├── stapp01 (CentOS) ← target
    ├── stapp02 (CentOS)
    └── stapp03 (CentOS)
```

> `uname -a` always shows the **host kernel** (Ubuntu), not the container OS.  
> Always use `cat /etc/os-release` to identify the actual OS.

---

## Tools: telnet & ss/netstat

### telnet

```bash
telnet <host> <port>

# Examples
telnet localhost 8088
telnet stapp01 8088
```

| Output                 | Meaning                           |
|------------------------|-----------------------------------|
| `Connected to ...`     | Port open, service responding     |
| `Connection refused`   | Port closed, no process listening |
| `Connection timed out` | Firewall blocking                 |

Exit connection: `Ctrl + ]` then `quit`

---

### ss (Socket Status)

```bash
# All listening TCP ports
sudo ss -tlnp

# Filter by specific port
sudo ss -tlnp | grep 8088
```

| Flag | Meaning |
|------|---------|
| `-t` | TCP |
| `-l` | listening only |
| `-n` | numeric (no DNS lookup) |
| `-p` | show process/PID |

**Reading the output:**
```
LISTEN 0  10  127.0.0.1:8088  0.0.0.0:*  users:(("sendmail",pid=8778,fd=4))
```

| Part             | Meaning                     |
|------------------|-----------------------------|
| `127.0.0.1:8088` | listening on localhost only |
| `0.0.0.0:8088`   | listening on all interfaces |
| `sendmail`       | process holding the port    |
| `pid=8778`       | process ID                  |

---

## iptables — Firewall on CentOS

### Configuration file

```bash
sudo cat /etc/sysconfig/iptables
```

### Default configuration explained

```
*filter                          # Work in the filter table
:INPUT ACCEPT [0:0]              # Default: allow incoming
:FORWARD ACCEPT [0:0]            # Default: allow forwarding
:OUTPUT ACCEPT [0:0]             # Default: allow outgoing

-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT   # Allow established connections
-A INPUT -p icmp -j ACCEPT                                # Allow ping
-A INPUT -i lo -j ACCEPT                                  # Allow loopback
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT  # Allow SSH
-A INPUT -j REJECT --reject-with icmp-host-prohibited     # Block everything else
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```

### A rule broken down

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8088 -j ACCEPT
```

| Part           | Meaning                                |
|----------------|----------------------------------------|
| `-A INPUT`     | append rule to INPUT chain             |
| `-p tcp`       | protocol TCP                           |
| `-m state`     | load connection state module           |
| `--state NEW`  | new connections only                   |
| `-m tcp`       | load TCP module (required for --dport) |
| `--dport 8088` | destination port 8088                  |
| `-j ACCEPT`    | allow the packet                       |

### Opening port 8088 (persistent)

```bash
# 1. Edit the configuration file
sudo vim /etc/sysconfig/iptables

# Add the following line BEFORE the REJECT line:
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8088 -j ACCEPT

# 2. Reload iptables
sudo systemctl restart iptables

# 3. Verify
sudo iptables -L -n | grep 8088
```

### Checking status

```bash
sudo systemctl status iptables
```

> `active (exited)` is **normal** for iptables. The process loads rules into the kernel and exits. The rules remain active.

---

## httpd (Apache) on CentOS

### Check configuration

```bash
sudo grep -i "listen" /etc/httpd/conf/httpd.conf
```

### Error: Address already in use


(98)Address already in use: AH00072: make_sock: could not bind to address
no listening sockets available, shutting down

Another process is already using the port httpd wants to bind to.

```bash
sudo ss -tlnp | grep 8083
sudo ps aux | grep httpd
```

---

## Sendmail — Resolving the Port Conflict

### Problem

Sendmail was configured on port 8088, blocking httpd from starting.

```bash
# Shows sendmail on port 8088
sudo ss -tlnp | grep 8088
# LISTEN  0  10  127.0.0.1:8088  0.0.0.0:*  users:(("sendmail",...))
```

### Understanding Sendmail configuration

| File          | Purpose                                       |
|---------------|-----------------------------------------------|
| `sendmail.mc` | Source file (human-readable)                  |
| `sendmail.cf` | Generated configuration (Sendmail reads this) |
| `m4`          | Compiler that transforms `.mc` into `.cf`     |

**Analogy:**
```
sendmail.mc  →  m4    →  sendmail.cf
.vue         →  Vite  →  bundle.js
.scss        →  sass  →  .css
```

> Never edit `sendmail.cf` directly. Changes will be overwritten on the next `m4` run.

### Changing the port (best practice)

```bash
# 1. Create backups
sudo cp /etc/mail/sendmail.mc /etc/mail/sendmail.mc.bak
sudo cp /etc/mail/sendmail.cf /etc/mail/sendmail.cf.bak

# 2. Find port in source file
grep -n "Port" /etc/mail/sendmail.mc

# 3. Edit source file
sudo vim /etc/mail/sendmail.mc
```

Change from:
```
DAEMON_OPTIONS(`Port=8088,Addr=127.0.0.1, Name=MTA')dnl
```

To (standard SMTP):
```
DAEMON_OPTIONS(`Port=25,Addr=127.0.0.1, Name=MTA')dnl
```

```bash
# 4. Restart sendmail
sudo systemctl restart sendmail

# 5. Verify
sudo ss -tlnp | grep "25"
```

### Standard SMTP port

| Port  | Usage                                    |
|-------|------------------------------------------|
| 25    | Standard SMTP (server to server)         |

### What is m4?

`m4` is a **macro processor** — it reads an input file, expands macros into their full content, and outputs the result. It is a general Unix tool used by many programs, not just Sendmail.

```
You write (readable)           m4 expands          Machine reads (complex)
────────────────────────────────────────────────────────────────────────
DAEMON_OPTIONS(`Port=25')      ---------->         O DaemonPortOptions=Port=25,...
```

### What is dnl?

`dnl` stands for **"delete to next line"**. Comment syntax in m4.

```
dnl DAEMON_OPTIONS(...)   <- this line is ignored / commented out
DAEMON_OPTIONS(...)       <- this line is active
```

---

## Diagnosing: Port not reachable

```bash
curl http://stapp01:8083
```

> curl: Failed to connect to stapp01 port 8088: Connection refused

```bash
sudo ss -tlnp | grep 8083
sudo systemctl status httpd
```

### Error messages compared

| Message                | Cause                            |
|------------------------|----------------------------------|
| `Connection refused`   | No process listening on the port |
| `Connection timed out` | Firewall blocking                |
| `No route to host`     | Network issue                    |

---

## Firewall on CentOS

```bash
# Check status
sudo systemctl status firewalld
sudo systemctl status iptables

# Find configuration file
sudo find / -type f -name iptables

# Edit configuration
sudo vim /etc/sysconfig/iptables
```

Added rule to `/etc/sysconfig/iptables` before the REJECT line:

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8083 -j ACCEPT
```

```bash
# Reload
sudo systemctl restart iptables
```

---

## Useful Commands Reference

```bash
# Identify OS
cat /etc/os-release
cat /etc/redhat-release

# Find process on a port
sudo ss -tlnp | grep 8088

# Search inside a file
grep -n "searchterm" /path/to/file

# Find files by name
sudo find /etc -type f -name "sendmail*"
```

---

## Summary

1. **Opened port 8088 in iptables** — added rule to `/etc/sysconfig/iptables`
2. **Discovered port conflict** — Sendmail was occupying port 8088
3. **Moved Sendmail** — changed port from 8088 to 25 in `sendmail.mc`
4. **Recompiled** — `sendmail.cf` was automatically updated on restart
5. **Started httpd** — Apache now running on port 8088
6. **Verified** with `ss`, `telnet`, `curl`
