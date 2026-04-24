Linux Server Hardening Guide
> Production Ubuntu 24.04 LTS hardening process applied to a live VPS running Docker-based services.
This is not a theoretical guide. Every step documented here was applied to a real production server running n8n, Traefik, and PostgreSQL. The goal was to reduce the attack surface while keeping all services operational.
---
Environment
Component	Version / Detail
OS	Ubuntu 24.04.4 LTS (Noble)
Kernel	6.8.0-107-generic
Server type	KVM VPS — 2 vCPU, 8 GB RAM, 100 GB SSD
Docker	29.04.0
Services running	Traefik (reverse proxy), n8n (automation), PostgreSQL 15
SSH port	2222 (changed from default 22)
---
Architecture Overview
```
Internet
    │
    ▼
Cloudflare (SSL termination + DDoS protection + proxy)
    │
    ▼
UFW Firewall (ports: 80, 443, 2222 — everything else denied)
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
1. Initial System Update
Before any hardening, update all packages and the kernel:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
sudo reboot
```
After reboot, verify the kernel version:
```bash
uname -r
# Expected: 6.8.0-107-generic or newer
```
Why: Running outdated packages means running known vulnerabilities. This is step zero before anything else.
---
2. User Management
Create a dedicated sudo user (never work as root)
```bash
# Create new user
adduser user

# Add to sudo group
usermod -aG sudo user

# Verify
groups user
# Expected output: USER : USER sudo
```
Why: Root login should be disabled. All privileged operations go through a named sudo user, which creates an audit trail and reduces the blast radius of a compromise.
---
3. SSH Hardening
SSH is the primary remote access vector and the most targeted service on any internet-facing server.
3.1 Generate ED25519 key pair (on your local machine)
```bash
# Run this on your LOCAL machine (Windows PowerShell or Linux terminal)
ssh-keygen -t ed25519 -C "your-identifier"

# This creates:
# ~/.ssh/id_ed25519       (private key — never share this)
# ~/.ssh/id_ed25519.pub   (public key — goes to server)
```
Why ED25519 over RSA: ED25519 is faster, shorter, and considered more secure than RSA-2048 for modern use cases.
3.2 Copy public key to server
```bash
# From local machine
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 user@YOUR-SERVER-IP

# Or manually append to authorized_keys on the server
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
3.3 Harden SSH configuration
Edit `/etc/ssh/sshd_config`:
```bash
sudo nano /etc/ssh/sshd_config
```
Apply these settings:
```bash
# Change default port
Port 2222

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
AllowUsers user
```
Also check `/etc/ssh/sshd_config.d/` for any override files and apply the same settings there if needed.
Restart SSH:
```bash
sudo systemctl restart sshd
```
Before closing your current session, open a NEW terminal and verify you can connect on port 2222 with your key:
```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 muser@YOUR-SERVER-IP
```
Why: Changing the port reduces automated scanning noise significantly. Disabling password auth eliminates brute force attacks entirely. Never close your existing session until you confirm the new connection works.
---
4. UFW Firewall
UFW (Uncomplicated Firewall) provides host-based packet filtering.
4.1 Set default policies
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```
4.2 Allow only necessary ports
```bash
# HTTP — needed for Traefik / Cloudflare
sudo ufw allow 80/tcp

# HTTPS — needed for Traefik / Cloudflare
sudo ufw allow 443/tcp

# SSH on custom port
sudo ufw allow 2222/tcp

# Explicitly block direct access to n8n port
# (n8n should only be accessible via Traefik, not directly)
sudo ufw deny 5678/tcp
```
4.3 Enable UFW
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
2222/tcp          ALLOW IN    Anywhere
5678/tcp          DENY IN     Anywhere
```
Why: Default deny means anything not explicitly allowed is blocked. Denying 5678 directly ensures n8n is only reachable through Traefik — not exposed raw to the internet.
---
5. Fail2ban
Fail2ban monitors log files and bans IPs that show malicious behavior (repeated failed logins, etc.).
5.1 Install
```bash
sudo apt install fail2ban -y
```
5.2 Create jail.local (never edit jail.conf directly)
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
port     = 2222
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 7200
```
Important: Set `port = 2222` to match your custom SSH port. Default Fail2ban monitors port 22.
5.3 Enable and start
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```
5.4 Verify
```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```
Expected:
```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  `- Total failed: 2
`- Actions
   |- Currently banned: 0
   `- Total banned: 0
```
Why: Even with key-only SSH, Fail2ban adds a layer against scanners and bots that probe ports. With `maxretry = 3` on SSH, an attacker gets 3 attempts before being locked out for 2 hours.
---
6. HTTP Security Headers Audit
After deploying Traefik and your services, audit HTTP response headers using:
```bash
curl -I https://your-domain.com
```
Or use online tools:
securityheaders.com
observatory.mozilla.org
Headers to verify are present:
`X-Content-Type-Options: nosniff`
`X-Frame-Options: SAMEORIGIN`
`Referrer-Policy: strict-origin-when-cross-origin`
`Permissions-Policy`
These can be added as middleware in Traefik configuration.
---
7. Docker Security Considerations
7.1 Never expose internal service ports directly
In `docker-compose.yml`, avoid publishing ports that should be internal:
```yaml
# BAD — exposes PostgreSQL to the host network
ports:
  - "5432:5432"

# GOOD — PostgreSQL only reachable within Docker network
# (no ports: section)
```
7.2 Use read-only Docker socket mount for Traefik
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
```
The `:ro` flag ensures Traefik can read Docker events but cannot modify containers.
7.3 Use Docker networks for service isolation
Services that don't need to communicate should be on separate networks. Services that do communicate (n8n ↔ PostgreSQL) share an internal network, isolated from public-facing services.
---
8. Verification Checklist
After completing all steps, verify:
```bash
# SSH working on port 2222 with key
ssh -p 2222 -i ~/.ssh/id_ed25519 muser@YOUR-SERVER-IP

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
Current Security Status
Control	Status	Notes
OS up to date	✅	Kernel 6.8.0-107
Root login disabled	✅	SSH PermitRootLogin no
Password auth disabled	✅	Key-only SSH
ED25519 key pair	✅	Strong modern algorithm
Custom SSH port	✅	Port 2222
UFW active	✅	Default deny incoming
Fail2ban active	✅	SSH jail enabled
n8n not directly exposed	✅	Port 5678 denied in UFW
SSL/TLS	✅	Managed by Cloudflare
Docker socket read-only	✅	:ro mount on Traefik
jail.local configured	⚠️	Pending — using jail.conf defaults
Fail2ban port 2222	⚠️	Verify port matches SSH config
HTTP security headers	⚠️	Audit pending via Traefik middleware
Rate limiting	🔜	Planned via Traefik middleware
---
Work in Progress
[ ] Create `jail.local` with custom Fail2ban rules for port 2222
[ ] Add HTTP security headers middleware in Traefik
[ ] Configure rate limiting for public-facing routes
[ ] Set up log monitoring (Wazuh or similar)
[ ] Automate security updates with `unattended-upgrades`
---
References
Ubuntu Server Hardening Guide — CIS Benchmark
Fail2ban Documentation
UFW Documentation
Traefik Security Docs
Docker Security Best Practices
---
Maintained by Matheo M. | True Recovery | Paraguay
Infrastructure: Self-managed Ubuntu 24.04 VPS
