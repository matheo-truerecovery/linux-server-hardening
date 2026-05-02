# Linux Server Hardening Guide
Ubuntu 24.04 LTS hardening process applied to a VPS environment. Every step documented here was applied to a real server environment running Docker-based services. The goal was to reduce the attack surface while keeping all services operational.

## Environment
| Component | Detail |
|-----------|--------|
| OS | Ubuntu 24.04.4 LTS (Noble) |
| Kernel | 6.8.0-111-generic |
| Server type | KVM VPS |
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
```

Why: Running outdated packages means running known vulnerabilities. This is step zero before anything else.

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

Why: Root login should be disabled. All privileged operations go through a named sudo user, which creates an audit trail and reduces the blast radius of a compromise.

## 3. SSH Hardening

### 3.1 Generate ED25519 key pair (on your local machine)
```bash
# Run this on your LOCAL machine
ssh-keygen -t ed25519 -C "your-identifier"
```

Why ED25519 over RSA: ED25519 is faster, shorter, and considered more secure than RSA-2048 for modern use cases.

### 3.2 Copy public key to server
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 user@YOUR-SERVER-IP
```

### 3.3 Harden SSH configuration
Edit `/etc/ssh/sshd_config`:

```bash
Port XXXX
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no
AllowAgentForwarding no
TCPKeepAlive no
Compression no
AllowUsers your-user
```

### 3.4 Legal banner
```bash
sudo nano /etc/ssh/banner.txt
# Add your legal warning text
```

Add to `/etc/ssh/sshd_config`:
```
Banner /etc/ssh/banner.txt
```

Restart SSH:
```bash
sudo systemctl restart ssh
```

Why: Changing the port reduces automated scanning noise. Disabling password auth eliminates brute force attacks entirely. Legal banner provides deterrence and legal protection.

## 4. UFW Firewall

### 4.1 Set default policies
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 4.2 Allow only Cloudflare IPs on ports 80/443
```bash
for ip in 173.245.48.0/20 103.21.244.0/22 103.22.200.0/22 103.31.4.0/22 \
  141.101.64.0/18 108.162.192.0/18 190.93.240.0/20 188.114.96.0/20 \
  197.234.240.0/22 198.41.128.0/17 162.158.0.0/15 104.16.0.0/13 \
  104.24.0.0/14 172.64.0.0/13 131.0.72.0/22; do
  sudo ufw allow from $ip to any port 80
  sudo ufw allow from $ip to any port 443
done

# SSH on custom port
sudo ufw allow XXXX/tcp

# Explicitly block direct service ports
sudo ufw deny 5678/tcp
```

### 4.3 Enable UFW
```bash
sudo ufw enable
sudo ufw status verbose
```

Why: Restricting 80/443 to Cloudflare IPs only means attackers cannot bypass Cloudflare to reach your server directly.

## 5. Fail2ban
```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Add:
```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
ignoreip = 127.0.0.1/8

[sshd]
enabled  = true
port     = XXXX
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 7200
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Why: Always use jail.local — never edit jail.conf directly, as updates will overwrite it.

## 6. HTTP Security Headers
Headers configured as Traefik middleware:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: SAMEORIGIN`
- `Referrer-Policy: same-origin`
- `Strict-Transport-Security: max-age=15552000; includeSubDomains; preload`

## 7. Docker Security

```yaml
# Never expose internal ports
# BAD
ports:
  - "5432:5432"

# GOOD — no ports mapping for internal services

# Read-only Docker socket for Traefik
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro

# Secure .env files
sudo chmod 600 /path/to/.env
```

## 8. Automated Backups
```bash
# n8n data — 3am daily
0 3 * * * tar -zcf /home/user/backups/n8n-backup-$(date +\%F).tar.gz /var/lib/docker/volumes/n8n_data

# Docker config — 2am daily
0 2 * * * tar -zcf /home/user/backups/docker-config-$(date +\%F).tar.gz /docker

# Sync to Google Drive — 3:30am daily
30 3 * * * rclone copy /home/user/backups/ gdrive:backups-server/

# Clean backups older than 14 days
0 5 * * * find /home/user/backups -name "*.tar.gz" -mtime +14 -delete
```

Why: Backups follow the 3-2-1 rule — 3 copies, 2 different media, 1 offsite.

## 9. SSH Login Alerts
```bash
#!/bin/bash
# /usr/local/bin/ssh-alert.sh
(
echo "To: your-email@gmail.com"
echo "From: your-email@gmail.com"
echo "Subject: SSH Access Alert"
echo ""
echo "Date: $(date)"
echo "User: $PAM_USER"
echo "IP: $PAM_RHOST"
echo "Type: $PAM_TYPE"
) | msmtp your-email@gmail.com
```

Add to `/etc/pam.d/sshd`:
```
session optional pam_exec.so /usr/local/bin/ssh-alert.sh
```

## 10. Cloudflare WAF Rules
- Block traffic from high-risk countries
- IP allowlist for sensitive subdomains
- Rate limiting on public-facing routes

## 11. Advanced Hardening (Lynis-based)

### Kernel parameters (sysctl)
```bash
sudo nano /etc/sysctl.conf
```

Add:
```
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.log_martians = 1
kernel.core_uses_pid = 1
fs.suid_dumpable = 0
kernel.randomize_va_space = 2
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
net.ipv4.ip_forward = 1
```

### Disable unnecessary protocols
```bash
echo "install dccp /bin/false" | sudo tee -a /etc/modprobe.d/disable-protocols.conf
echo "install sctp /bin/false" | sudo tee -a /etc/modprobe.d/disable-protocols.conf
echo "install rds /bin/false" | sudo tee -a /etc/modprobe.d/disable-protocols.conf
echo "install tipc /bin/false" | sudo tee -a /etc/modprobe.d/disable-protocols.conf
echo "install usb-storage /bin/false" | sudo tee -a /etc/modprobe.d/disable-protocols.conf
```

### Auditd
```bash
sudo apt install auditd -y
sudo nano /etc/audit/rules.d/server.rules
```

Add:
```
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/ssh/sshd_config -p wa -k sshd
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands
-w /var/log -p wa -k logs
```

### AIDE (File integrity monitoring)
```bash
sudo apt install aide -y
sudo aideinit
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

Add to root crontab:
```
0 6 * * * /usr/bin/aide --check | msmtp your-email@gmail.com
```

### Password policy
```
PASS_MAX_DAYS  90
PASS_MIN_DAYS  1
PASS_WARN_AGE  14
```

## 12. Daily Health Check
Script at `/usr/local/bin/health-check.sh` that emails daily:
- Docker container status
- Disk, RAM, CPU usage
- Failed login attempts with geolocation
- Fail2ban banned IPs
- Backup status
- Pending system updates
- Docker image versions and dates

## Verification Checklist
```bash
# SSH working on custom port with key
ssh -p XXXX -i ~/.ssh/id_ed25519 user@YOUR-SERVER-IP

# UFW active with correct rules
sudo ufw status verbose

# Fail2ban monitoring sshd
sudo fail2ban-client status sshd

# Docker containers running
docker ps

# No unauthorized open ports
sudo ss -tlnp

# Pending updates
sudo apt list --upgradable 2>/dev/null

# Lynis score
sudo lynis audit system 2>/dev/null | grep "Hardening index"
```

## Security Status
| Control | Status | Notes |
|---------|--------|-------|
| OS up to date | ✅ | Auto-updates configured |
| Root login disabled | ✅ | SSH PermitRootLogin no |
| Password auth disabled | ✅ | Key-only SSH |
| ED25519 key pair | ✅ | Strong modern algorithm |
| Custom SSH port | ✅ | Changed from default 22 |
| MaxAuthTries 3 | ✅ | Limits brute force |
| LoginGraceTime 30 | ✅ | Closes idle connections |
| Legal SSH banner | ✅ | Legal deterrence |
| UFW active | ✅ | Cloudflare IPs only on 80/443 |
| Fail2ban active | ✅ | jail.local configured |
| SSL/TLS | ✅ | Managed by Cloudflare |
| Docker socket read-only | ✅ | :ro mount on Traefik |
| HTTP security headers | ✅ | Via Traefik middleware |
| Automated backups | ✅ | Daily + Google Drive sync |
| SSH login alerts | ✅ | Email on every session |
| Cloudflare WAF | ✅ | Country blocking + IP allowlist |
| Kernel hardening | ✅ | sysctl parameters |
| Unnecessary protocols disabled | ✅ | dccp, sctp, rds, tipc, usb |
| Auditd active | ✅ | Custom rules |
| AIDE active | ✅ | Daily integrity check |
| Lynis score | ✅ | 73/100 |
| Core dumps disabled | ✅ | |
| Password policy | ✅ | 90 day expiry |

## Work in Progress
- [ ] WireGuard VPN for remote access
- [ ] Cloudflare Access (Zero Trust) authentication layer
- [ ] mTLS in Traefik
- [ ] Centralized log monitoring

## References
- CIS Ubuntu Benchmark
- NIST SP 800-123
- Fail2ban Documentation
- UFW Documentation
- Traefik Security Docs
- Docker Security Best Practices
- OWASP Top 10

---
Maintained by M.M.
