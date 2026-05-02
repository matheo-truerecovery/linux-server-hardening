Linux Server Hardening Guide
Ubuntu 24.04 LTS hardening process applied to a VPS environment. Every step documented here was applied to a real server environment running Docker-based services. The goal was to reduce the attack surface while keeping all services operational.
Environment
Component
Detail
OS
Ubuntu 24.04.4 LTS (Noble)
Kernel
6.8.0-111-generic
Server type
KVM VPS
Docker
29.04.0
Services running
Traefik (reverse proxy), n8n (automation), PostgreSQL 15
SSH port
Custom port (changed from default 22)
Architecture Overview
Código
SSL/TLS is managed externally by Cloudflare. Internal traffic between Cloudflare and Traefik runs over HTTP. This is a deliberate architecture decision — the public-facing connection is always HTTPS via Cloudflare proxy.
1. Initial System Update
Before any hardening, update all packages and the kernel:
Bash
After reboot, verify the kernel version:
Bash
Why: Running outdated packages means running known vulnerabilities. This is step zero before anything else.
2. User Management
Create a dedicated sudo user (never work as root):
Bash
Why: Root login should be disabled. All privileged operations go through a named sudo user, which creates an audit trail and reduces the blast radius of a compromise.
3. SSH Hardening
3.1 Generate ED25519 key pair (on your local machine)
Bash
Why ED25519 over RSA: ED25519 is faster, shorter, and considered more secure than RSA-2048 for modern use cases.
3.2 Copy public key to server
Bash
3.3 Harden SSH configuration
Edit /etc/ssh/sshd_config:
Bash
3.4 Legal banner
Bash
Add to /etc/ssh/sshd_config:
Código
Restart SSH:
Bash
Why: Changing the port reduces automated scanning noise. Disabling password auth eliminates brute force attacks entirely. Legal banner provides deterrence and legal protection.
4. UFW Firewall
4.1 Set default policies
Bash
4.2 Allow only Cloudflare IPs on ports 80/443
Bash
4.3 Enable UFW
Bash
Why: Restricting 80/443 to Cloudflare IPs only means attackers cannot bypass Cloudflare to reach your server directly.
5. Fail2ban
Bash
Add:
Ini
Bash
Why: Always use jail.local — never edit jail.conf directly, as updates will overwrite it.
6. HTTP Security Headers
Headers configured as Traefik middleware:
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Referrer-Policy: same-origin
Strict-Transport-Security: max-age=15552000; includeSubDomains; preload
7. Docker Security
Yaml
8. Automated Backups
Bash
Why: Backups follow the 3-2-1 rule — 3 copies, 2 different media, 1 offsite.
9. SSH Login Alerts
Bash
Add to /etc/pam.d/sshd:
Código
10. Cloudflare WAF Rules
Block traffic from high-risk countries
IP allowlist for sensitive subdomains
Rate limiting on public-facing routes
11. Advanced Hardening (Lynis-based)
Kernel parameters (sysctl)
Bash
Add:
Código
Disable unnecessary protocols
Bash
Auditd
Bash
Add:
Código
AIDE (File integrity monitoring)
Bash
Add to root crontab:
Código
Password policy
Código
12. Daily Health Check
Script at /usr/local/bin/health-check.sh that emails daily:
Docker container status
Disk, RAM, CPU usage
Failed login attempts with geolocation
Fail2ban banned IPs
Backup status
Pending system updates
Docker image versions and dates
Verification Checklist
Bash
Security Status
Control
Status
Notes
OS up to date
✅
Auto-updates configured
Root login disabled
✅
SSH PermitRootLogin no
Password auth disabled
✅
Key-only SSH
ED25519 key pair
✅
Strong modern algorithm
Custom SSH port
✅
Changed from default 22
MaxAuthTries 3
✅
Limits brute force
LoginGraceTime 30
✅
Closes idle connections
Legal SSH banner
✅
Legal deterrence
UFW active
✅
Cloudflare IPs only on 80/443
Fail2ban active
✅
jail.local configured
SSL/TLS
✅
Managed by Cloudflare
Docker socket read-only
✅
:ro mount on Traefik
HTTP security headers
✅
Via Traefik middleware
Automated backups
✅
Daily + Google Drive sync
SSH login alerts
✅
Email on every session
Cloudflare WAF
✅
Country blocking + IP allowlist
Kernel hardening
✅
sysctl parameters
Unnecessary protocols disabled
✅
dccp, sctp, rds, tipc, usb
Auditd active
✅
Custom rules
AIDE active
✅
Daily integrity check
Lynis score
✅
73/100
Core dumps disabled
✅

Password policy
✅
90 day expiry
Work in Progress
[ ] WireGuard VPN for remote access
[ ] Cloudflare Access (Zero Trust) authentication layer
[ ] mTLS in Traefik
[ ] Centralized log monitoring
References
CIS Ubuntu Benchmark
NIST SP 800-123
Fail2ban Documentation
UFW Documentation
Traefik Security Docs
Docker Security Best Practices
OWASP Top 10
Maintained by M.M.