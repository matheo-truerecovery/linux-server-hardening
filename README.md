# Linux Server Hardening Guide

Production Ubuntu 24.04 LTS hardening process applied to a live VPS running Docker-based services. This is not a theoretical guide. Every step documented here was applied to a real production server running n8n, Traefik, and PostgreSQL. The goal was to reduce the attack surface while keeping all services operational.

---

## Environment

| Component | Detail |
|-----------|--------|
| OS | Ubuntu 24.04.4 LTS (Noble) |
| Kernel | 6.8.0-110-generic (latest) |
| Server type | KVM VPS — resources omitted for security |
| Docker | 29.04.0 |
| Services running | Traefik (reverse proxy), n8n (automation), PostgreSQL 15 |
| SSH port | Custom port (changed from default 22) |



## Architecture Overview

```
Internet
    │
    ▼
Cloudflare (SSL termination + DDoS protection + WAF + IP blocking)
    │
    ▼
UFW Firewall (ports: 80, 443, custom SSH — everything else denied)
    │
    ▼
Traefik (reverse proxy — routes traffic to containers)
    │
    ├──▶ n8n (automation platform — internal only via Traefik)
    │
    └──▶ PostgreSQL (database — internal Docker network only)
```

SSL/TLS is managed externally by Cloudflare. Internal traffic between Cloudflare and Traefik runs over HTTP. This is a deliberate architecture decision — the public-facing connection is always HTTPS via Cloudflare proxy.

---

## 1. Initial System Update

Before any hardening, update all packages and the kernel:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
sudo reboot
```

After reboot, verify the kernel version:

```bash
uname -r
# Expected: 6.8.0-110-generic or newer
```

**Why:** Running outdated packages means running known vulnerabilities. This is step zero before anything else.

---

## 2. User Management

Create a dedicated sudo user (never work as root):

```bash
# Create new user
adduser mateo

# Add to sudo group
usermod -aG sudo mateo

# Verify
groups mateo
# Expected output: mateo : mateo sudo
```

**Why:** Root login should be disabled. All privileged operations go through a named sudo user, which creates an audit trail and reduces the blast radius of a compromise.

---

## 3. SSH Hardening

SSH is the primary remote access vector and the most targeted service on any internet-facing server.

### 3.1 Generate ED25519 key pair (on your local machine)

```bash
# Run this on your LOCAL machine (Windows PowerShell or Linux terminal)
ssh-keygen -t ed25519 -C "your-identifier"

# This creates:
# ~/.ssh/id_ed25519       (private key — never share this)
# ~/.ssh/id_ed25519.pub   (public key — goes to server)
```

**Why ED25519 over RSA:** ED25519 is faster, shorter, and considered more secure than RSA-2048 for modern use cases.

### 3.2 Copy public key to server

```bash
# From local machine
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 user@YOUR-SERVER-IP

# Or manually append to authorized_keys on the server
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 3.3 Harden SSH configuration

Edit `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

Apply these settings:

```bash
# Change default port (use a custom port)
Port XXXX

# Disable root login
PermitRootLogin no

# Disable password authentication (key-only)
PasswordAuthentication no
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no

# Limit authentication attempts
MaxAuthTries 3

# Disable unused authentication methods
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no

# Set login grace time
LoginGraceTime 30

# Allow only specific users
AllowUsers mateo
```

Also check `/etc/ssh/sshd_config.d/` for any override files and apply the same settings there if needed.

Restart SSH:

```bash
sudo systemctl restart sshd
```

**Before closing your current session**, open a NEW terminal and verify you can connect on your custom port with your key:

```bash
ssh -p XXXX -i ~/.ssh/id_ed25519 mateo@YOUR-SERVER-IP
```

**Why:** Changing the port reduces automated scanning noise significantly. Disabling password auth eliminates brute force attacks entirely. Never close your existing session until you confirm the new connection works.

---

## 4. UFW Firewall

UFW (Uncomplicated Firewall) provides host-based packet filtering.

### 4.1 Set default policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

### 4.2 Allow only necessary ports

```bash
# HTTP — needed for Traefik / Cloudflare
sudo ufw allow 80/tcp

# HTTPS — needed for Traefik / Cloudflare
sudo ufw allow 443/tcp

# SSH on custom port
sudo ufw allow XXXX/tcp

# Explicitly block direct access to n8n port
# (n8n should only be accessible via Traefik, not directly)
sudo ufw deny 5678/tcp
```

### 4.3 Enable UFW

```bash
sudo ufw enable
sudo ufw status verbose
```

Expected output:

```
Status: active
Default: deny (incoming), allow (outgoing), deny (routed)

To                Action      From
--                ------      ----
80/tcp            ALLOW IN    Anywhere
443/tcp           ALLOW IN    Anywhere
XXXX/tcp          ALLOW IN    Anywhere
5678/tcp          DENY IN     Anywhere
```

**Why:** Default deny means anything not explicitly allowed is blocked. Denying 5678 directly ensures n8n is only reachable through Traefik — not exposed raw to the internet.

---

## 5. Fail2ban

Fail2ban monitors log files and bans IPs that show malicious behavior (repeated failed logins, etc.).

### 5.1 Install

```bash
sudo apt install fail2ban -y
```

### 5.2 Create jail.local (never edit jail.conf directly)

```bash
sudo nano /etc/fail2ban/jail.local
```

Add:

```ini
[DEFAULT]
# Ban duration in seconds (1 hour)
bantime  = 3600

# Time window to count failures
findtime = 600

# Number of failures before ban
maxretry = 5

# Ignore local traffic
ignoreip = 127.0.0.1/8

[sshd]
enabled  = true
port     = XXXX
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 7200
```

**Important:** Set `port = XXXX` to match your custom SSH port. Default Fail2ban monitors port 22.

### 5.3 Enable and start

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### 5.4 Verify

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

**Why:** Even with key-only SSH, Fail2ban adds a layer against scanners and bots that probe ports. With `maxretry = 3` on SSH, an attacker gets 3 attempts before being locked out for 2 hours.

---

## 6. HTTP Security Headers Audit

After deploying Traefik and your services, audit HTTP response headers using:

```bash
curl -I https://your-domain.com
```

Or use online tools:
- securityheaders.com
- observatory.mozilla.org

Headers verified present:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: SAMEORIGIN`
- `Referrer-Policy: same-origin`
- `Strict-Transport-Security: max-age=15552000; includeSubDomains; preload`

These are configured as middleware in Traefik.

---

## 7. Docker Security Considerations

### 7.1 Never expose internal service ports directly

In `docker-compose.yml`, avoid publishing ports that should be internal:

```yaml
# BAD — exposes PostgreSQL to the host network
ports:
  - "5432:5432"

# GOOD — PostgreSQL only reachable within Docker network
# (no ports: section)
```

### 7.2 Use read-only Docker socket mount for Traefik

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
```

The `:ro` flag ensures Traefik can read Docker events but cannot modify containers.

### 7.3 Use Docker networks for service isolation

Services that don't need to communicate should be on separate networks. Services that do communicate (n8n ↔ PostgreSQL) share an internal network, isolated from public-facing services.

### 7.4 Secure environment files

```bash
# Restrict .env file permissions — only owner can read
sudo chmod 600 /path/to/.env
```

**Why:** `.env` files contain sensitive credentials. Default permissions allow any system user to read them.

---

## 8. Automated Backups

Automated daily backups using cron jobs running as root:

```bash
# n8n data backup — runs at 3am daily
0 3 * * * tar -zcf /home/user/backups/n8n-backup-$(date +\%F).tar.gz /var/lib/docker/volumes/n8n_data

# Docker config backup — runs at 2am daily
0 2 * * * tar -zcf /home/user/backups/docker-config-$(date +\%F).tar.gz /docker

# Sync backups to Google Drive — runs at 3:30am daily
30 3 * * * rclone copy /home/user/backups/ gdrive:backups-server/

# Clean backups older than 14 days
0 5 * * * find /home/user/backups -name "*.tar.gz" -mtime +14 -delete
```

**Why:** Backups follow the 3-2-1 rule — 3 copies, 2 different media, 1 offsite (Google Drive). If the server is fully compromised, data is recoverable from Google Drive.

---

## 9. SSH Login Alerts

Automated email alert on every SSH session open/close using PAM:

```bash
# /usr/local/bin/ssh-alert.sh
#!/bin/bash
(
echo "To: your-email@gmail.com"
echo "From: your-email@gmail.com"
echo "Subject: SSH Access - Server Alert"
echo ""
echo "Date: $(date)"
echo "User: $PAM_USER"
echo "IP: $PAM_RHOST"
echo "Type: $PAM_TYPE"
) | msmtp --file=/etc/msmtprc your-email@gmail.com
```

Add to `/etc/pam.d/sshd`:

```
session optional pam_exec.so /usr/local/bin/ssh-alert.sh
```

**Why:** Immediate notification of any SSH access — legitimate or not. If an unexpected IP appears, you know instantly.

---

## 10. Cloudflare WAF Rules

Additional protection at the edge:

- Block traffic from high-risk countries (China, Hong Kong, etc.)
- IP allowlist for sensitive subdomains
- Rate limiting on public-facing routes

**Why:** Blocking known attack sources at Cloudflare level means malicious requests never reach the server.

---

## Verification Checklist

```bash
# SSH working on custom port with key
ssh -p XXXX -i ~/.ssh/id_ed25519 mateo@YOUR-SERVER-IP

# UFW active with correct rules
sudo ufw status verbose

# Fail2ban monitoring sshd
sudo fail2ban-client status sshd

# Docker containers running
docker ps

# No unauthorized open ports
sudo ss -tlnp
```

---

## Current Security Status

| Control | Status | Notes |
|---------|--------|-------|
| OS up to date | ✅ | Kernel 6.8.0-110, auto-updates daily |
| Root login disabled | ✅ | SSH PermitRootLogin no |
| Password auth disabled | ✅ | Key-only SSH |
| ED25519 key pair | ✅ | Strong modern algorithm |
| Custom SSH port | ✅ | Changed from default 22 |
| AllowUsers configured | ✅ | Only named user allowed |
| UFW active | ✅ | Default deny incoming |
| Fail2ban active | ✅ | SSH jail enabled, custom port |
| jail.local configured | ✅ | Custom rules applied |
| n8n not directly exposed | ✅ | Port 5678 denied in UFW |
| SSL/TLS | ✅ | Managed by Cloudflare |
| Docker socket read-only | ✅ | :ro mount on Traefik |
| HTTP security headers | ✅ | Configured via Traefik middleware |
| .env files secured | ✅ | chmod 600 applied |
| Automated backups | ✅ | Daily, synced to Google Drive |
| SSH login alerts | ✅ | Email notification on every session |
| Cloudflare WAF | ✅ | Country blocking + IP allowlist |
| PostgreSQL password | ✅ | Strong random key (32 bytes) |
| n8n JWT secret | ✅ | Strong random key (32 bytes) |

---

## Work in Progress
## Advanced Hardening Applied (May 2026)

### Lynis Security Audit
- Hardening score improved: 62 → 73/100
- 263 security tests performed

### Implemented Controls
- Auditd with custom rules (monitors /etc/passwd, /etc/shadow, /etc/sudoers, SSH config, Docker)
- AIDE file integrity monitoring (detects unauthorized file changes)
- Kernel hardening via sysctl (redirects disabled, martians logged, ASLR enabled)
- Unnecessary network protocols disabled (dccp, sctp, rds, tipc)
- USB storage disabled
- Core dumps disabled
- Process accounting enabled (acct)
- Sysstat enabled for performance monitoring
- SSH legal banner configured (/etc/issue.net)
- SSH additional hardening (MaxAuthTries 3, LoginGraceTime 30, compression disabled)
- Password policy enforced (90 day expiry, complexity requirements via libpam-pwquality)
- Fail2ban jail.local configured
- Cloudflare Access (Zero Trust) as authentication layer before n8n
- UFW restricted to Cloudflare IP ranges only (ports 80/443)
- Web published on Cloudflare Pages (truesolutionsgestion.com)
- [ ] mTLS implementation in Traefik for client certificate authentication
- [ ] Rate limiting middleware in Traefik
- [ ] Centralized log monitoring (Wazuh or similar)
- [ ] Wireguard VPN for remote access

---

## References

- Ubuntu Server Hardening Guide — CIS Benchmark
- Fail2ban Documentation
- UFW Documentation
- Traefik Security Docs
- Docker Security Best Practices
- OWASP Top 10

## Advanced Hardening (2026)
- Lynis hardening score: 73/100
- Auditd with custom rules
- AIDE file integrity monitoring
- Kernel parameters hardening (sysctl)
- Unnecessary protocols disabled
- Cloudflare Access (Zero Trust) for n8n
- SSH legal banner
- Process accounting enabled

*Maintained by M.M. | True Recovery | Paraguay*
*Infrastructure: Self-managed Ubuntu 24.04 VPS*
