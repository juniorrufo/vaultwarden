# Vaultwarden Infrastructure — Project Context

## 1. Purpose

This repository documents and stores the reproducible configuration for a
self-hosted Vaultwarden deployment running in a dedicated virtual machine
inside a private home network.

The primary goals are:

- Secure self-hosted password management.
- Minimal attack surface.
- Reproducible infrastructure.
- Documented operational procedures.
- Reliable local and off-site backups.
- Tested disaster recovery.
- Explicit configuration instead of hidden/manual changes.
- Ability to rebuild the environment from documentation + versioned configuration
  + external backups.

This is a personal/family password manager deployment, not a large-scale
multi-tenant service.

---

# 2. Current Architecture

## High-level topology

```text
                            INTERNET
                                |
                                v
                       +----------------+
                       |   Cloudflare   |
                       | DNS / TLS      |
                       +-------+--------+
                               |
                               v
                    Cloudflare Tunnel
                               |
                               v
                    192.168.15.253
                     Cloudflared LXC
                               |
                         HTTP :8080
                               |
                               v
                    192.168.15.200
                     Vaultwarden VM
                               |
                         Docker Engine
                               |
                               v
                     Vaultwarden :80
                               |
                               v
                           SQLite
```

## Virtualization

Hypervisor:

- Proxmox VE

Guest:

- Dedicated VM for Vaultwarden

The Vaultwarden application is intentionally isolated in its own VM.

---

# 3. VM Information

Hostname:

```text
vaultwarden
```

Operating System:

```text
Debian GNU/Linux 13 (trixie)
```

Architecture:

```text
x86-64 / amd64
```

Virtualization:

```text
KVM
```

Current kernel at deployment:

```text
Linux 6.12.107+deb13-amd64
```

CPU:

```text
2 vCPU
```

Memory:

```text
~2 GB
```

Disk:

```text
~32 GB
```

Main network interface:

```text
ens18
```

IPv4:

```text
192.168.15.200/24
```

Default gateway:

```text
192.168.15.1
```

IPv6 is enabled and functional.

The VM currently has a globally routable IPv6 address assigned by the
network. IPv6 SSH access is intentionally blocked by the host firewall.

---

# 4. Network Design

LAN:

```text
192.168.15.0/24
```

Important hosts:

```text
Vaultwarden VM:
192.168.15.200

Cloudflare Tunnel LXC:
192.168.15.253

Nginx Proxy Manager:
192.168.15.251
```

Nginx Proxy Manager exists elsewhere in the network but is NOT used by
Vaultwarden.

## Why NPM is not used

The Vaultwarden deployment uses:

```text
Cloudflare
    |
Cloudflare Tunnel
    |
Vaultwarden
```

instead of:

```text
Cloudflare
    |
Cloudflare Tunnel
    |
NPM
    |
Vaultwarden
```

Reasons:

- Avoid an unnecessary additional reverse-proxy dependency.
- Reduce operational complexity.
- Reduce attack surface.
- Avoid making Vaultwarden dependent on NPM availability.
- Cloudflare already provides the public HTTPS endpoint.
- The Cloudflare Tunnel can directly forward requests to the Vaultwarden VM.

NPM remains available for other services in the network.

---

# 5. Public URL

Production URL:

```text
https://vault.rufonex.com.br
```

The public HTTPS endpoint is handled by Cloudflare.

Origin:

```text
http://192.168.15.200:8080
```

The Cloudflare Tunnel connects to the internal HTTP endpoint.

There is intentionally no direct Internet port forwarding to the Vaultwarden
VM.

---

# 6. Cloudflare Tunnel

The Cloudflare Tunnel runs in a dedicated LXC.

Cloudflared host:

```text
192.168.15.253
```

The tunnel is managed remotely by Cloudflare.

Current local configuration model:

```text
/etc/cloudflared/token
```

No local `cert.pem` is required for the runtime configuration.

There is currently no:

```text
/etc/cloudflared/config.yml
```

Tunnel management is performed from the Cloudflare Dashboard.

IMPORTANT:

Never commit the tunnel token or any Cloudflare secret to Git.

The public route configured in Cloudflare is:

```text
vault.rufonex.com.br
    ->
http://192.168.15.200:8080
```

---

# 7. Firewall Architecture

The host does NOT use a directly managed nftables ruleset.

Reason:

Docker integrates with iptables and expects the host firewall to cooperate
with Docker's iptables chains.

Current firewall implementation:

```text
iptables-nft
ip6tables-nft
```

Versions at deployment:

```text
iptables v1.8.11 (nf_tables)
ip6tables v1.8.11 (nf_tables)
```

The Debian `nftables.service` was intentionally disabled and masked.

Current state:

```text
nftables.service
    masked
    inactive
```

Persistent firewall:

```text
netfilter-persistent
```

is enabled.

---

# 8. Base Host Firewall Policy

IPv4:

```text
INPUT   DROP
FORWARD DROP
OUTPUT  ACCEPT
```

IPv6:

```text
INPUT   DROP
FORWARD DROP
OUTPUT  ACCEPT
```

IPv4 SSH:

```text
192.168.15.0/24 -> TCP/22 -> ALLOW
```

SSH from IPv6 is intentionally not allowed.

Allowed basic traffic includes:

- loopback
- established/related connections
- IPv4 ICMP
- required IPv6 ICMPv6 functionality
- SSH from the management LAN

The firewall configuration survives VM reboots.

A reboot test was performed successfully.

---

# 9. Docker Firewall Policy

Docker creates its standard chains including:

```text
DOCKER
DOCKER-BRIDGE
DOCKER-CT
DOCKER-FORWARD
DOCKER-INTERNAL
DOCKER-USER
```

The administrative firewall policy is implemented through:

```text
DOCKER-USER
    |
    v
VW-DOCKER
```

The dedicated chain is:

```text
VW-DOCKER
```

Current policy:

```text
ESTABLISHED,RELATED
    -> ACCEPT

192.168.15.253
    + original destination 192.168.15.200:8080
    -> ACCEPT

other sources
    + original destination 192.168.15.200:8080
    -> DROP

everything else
    -> RETURN
```

Filtering uses conntrack original-destination matching because Docker performs
DNAT before traffic reaches the administrative filtering point.

The effective rule set is:

```text
-A DOCKER-USER -j VW-DOCKER

-A VW-DOCKER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

-A VW-DOCKER -s 192.168.15.253/32 \
    -p tcp \
    -m conntrack \
    --ctorigdst 192.168.15.200 \
    --ctorigdstport 8080 \
    -j ACCEPT

-A VW-DOCKER \
    -p tcp \
    -m conntrack \
    --ctorigdst 192.168.15.200 \
    --ctorigdstport 8080 \
    -j DROP

-A VW-DOCKER -j RETURN
```

This means:

```text
Cloudflared LXC
192.168.15.253
        |
        | TCP/8080
        v
Vaultwarden
192.168.15.200
        |
        v
ALLOW
```

while other hosts cannot directly access the published Vaultwarden port.

This was tested successfully.

Test from Cloudflared:

```text
curl http://192.168.15.200:8080/alive
```

Result:

```text
HTTP 200
```

Test from another LAN origin:

```text
curl http://192.168.15.200:8080/alive
```

Result:

```text
connection timeout
```

The firewall policy is implemented by:

```text
/usr/local/sbin/vaultwarden-docker-firewall
```

and:

```text
/etc/systemd/system/vaultwarden-docker-firewall.service
```

The systemd service is enabled and reapplies the Docker-specific policy after
Docker becomes available.

The policy was tested after:

- Docker restart
- full VM reboot

and remained functional.

---

# 10. SSH Hardening

Administrative user:

```text
junior
```

SSH authentication is public-key only.

Current effective SSH configuration:

```text
UsePAM yes
MaxAuthTries 3
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
AllowUsers junior
```

SSH configuration was validated with:

```bash
sudo sshd -t
```

and the effective configuration was inspected with:

```bash
sudo sshd -T
```

Two public keys are authorized for the `junior` account:

```text
WSL key
Windows PowerShell key
```

Private keys MUST NOT be stored in Git.

The existing private keys are backed up separately in a secure offline location.

---

# 11. SSH Access

Management normally happens from:

```text
WSL
```

and:

```text
Windows PowerShell
```

Example:

```bash
ssh -o IdentitiesOnly=yes \
    -i ~/.ssh/id_ed25519 \
    junior@192.168.15.200
```

PowerShell equivalent uses the Windows private key.

---

# 12. Proxmox QEMU Guest Agent

QEMU Guest Agent is installed and operational.

Guest communication channel:

```text
/dev/virtio-ports/org.qemu.guest_agent.0
```

The service is:

```text
qemu-guest-agent.service
```

and is intentionally installed as a static systemd service according to the
Debian packaging model.

Verified status:

```text
active (running)
```

The Proxmox VM has QEMU Guest Agent enabled.

---

# 13. Time Synchronization

Timezone:

```text
America/Sao_Paulo
```

NTP:

```text
enabled
```

System clock:

```text
synchronized
```

The VM uses network time synchronization.

Correct time is important for:

- TOTP
- TLS
- logs
- token validation
- scheduled backups

---

# 14. Docker Installation

Docker was installed from the official Docker Debian repository.

Current components at deployment:

```text
Docker Engine: 29.8.1
Docker Compose: 5.5.1
containerd: 2.3.5
runc: 1.5.1
Buildx: 0.37.1
```

Docker uses:

```text
Storage Driver:
overlayfs

Cgroup Driver:
systemd

Cgroup Version:
2

Firewall Backend:
iptables
```

Docker root directory:

```text
/var/lib/docker
```

Docker service:

```text
enabled
active
```

---

# 15. Docker Daemon Configuration

File:

```text
/etc/docker/daemon.json
```

Current configuration:

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "20m",
    "max-file": "5"
  },
  "live-restore": true
}
```

Purpose:

- use controlled local Docker logging
- prevent unbounded container log growth
- retain a limited number of log files
- keep supported containers running during certain Docker daemon restarts

Docker configuration was validated using:

```bash
dockerd --validate --config-file=/etc/docker/daemon.json
```

---

# 16. Docker Privilege Model

The administrative Linux user `junior` is NOT a member of the `docker` group.

Docker commands are intentionally executed with:

```bash
sudo docker ...
```

Reason:

Membership in the Docker group effectively grants high privileges over the
Docker daemon and therefore over the host.

Do not add the user to the Docker group without a specific architectural
reason.

---

# 17. Vaultwarden Container

Application:

```text
Vaultwarden
```

Current server version:

```text
1.37.3
```

Container:

```text
vaultwarden
```

Image is pinned by digest.

Current image digest:

```text
sha256:4ecafc9049c7d878c7717d1ce4f9059d706758c78b8fa42e5ead21f4b2dfc770
```

Platform:

```text
linux/amd64
```

Do not replace the image reference with:

```text
latest
```

without a deliberate upgrade process.

Image upgrades must be intentional and documented.

---

# 18. Vaultwarden Docker Compose

Current Compose file:

```text
/opt/vaultwarden/docker-compose.yaml
```

The deployment intentionally uses a single application container.

There is NO:

- PostgreSQL
- Redis
- Nginx
- Nginx Proxy Manager
- Portainer
- Watchtower
- separate Cloudflared container

The architecture intentionally avoids unnecessary components.

---

# 19. Vaultwarden Docker Network

Compose creates:

```text
vaultwarden_net
```

Network type:

```text
bridge
```

The network is declared in Compose rather than created manually.

Current design:

```text
vaultwarden_net
    |
    +-- vaultwarden
```

This keeps the Compose deployment reproducible.

---

# 20. Vaultwarden Port Publishing

Current production binding:

```text
192.168.15.200:8080 -> container:80
```

The service is NOT published on:

```text
0.0.0.0:8080
```

The Docker firewall policy restricts access to this published port so that only
the Cloudflare Tunnel LXC can reach it.

---

# 21. Vaultwarden Container Security Settings

Current container configuration includes:

```yaml
restart: unless-stopped
```

and:

```yaml
security_opt:
  - no-new-privileges:true
```

The container has:

```yaml
stop_grace_period: 30s
```

The image-provided healthcheck is used instead of a custom healthcheck.

Current health state has been verified as:

```text
healthy
```

The application health endpoint is:

```text
/alive
```

Example:

```bash
curl http://192.168.15.200:8080/alive
```

Expected successful response:

```text
HTTP 200
```

---

# 22. Vaultwarden Persistent Data

Host directory:

```text
/opt/vaultwarden/data
```

Container mount:

```text
/opt/vaultwarden/data:/data
```

The persistent data directory currently contains items such as:

```text
db.sqlite3
rsa_key.pem
icon_cache/
tmp/
```

Additional directories/files may appear as Vaultwarden features are used.

The `/data` directory is critical to disaster recovery.

Do not delete or recreate it without understanding the recovery impact.

---

# 23. Database

Database engine:

```text
SQLite
```

Primary database:

```text
/opt/vaultwarden/data/db.sqlite3
```

Reason for SQLite:

- personal deployment
- low operational complexity
- low resource consumption
- easy recovery
- no additional database service
- appropriate for current workload

Do NOT introduce PostgreSQL solely for the sake of making the
architecture appear more "professional".

The current architecture intentionally values reduced operational complexity.

---

# 24. Vaultwarden Account

The primary user account was created successfully.

The following operations were tested:

```text
create account
    ->
login
    ->
logout
    ->
login again
```

The Web Vault works through:

```text
https://vault.rufonex.com.br
```

The official Bitwarden browser extension was also configured to use the
self-hosted environment rather than Bitwarden Cloud.

The browser extension was tested successfully.

---

# 25. Self-hosted Bitwarden Client Configuration

The browser extension must point to:

```text
https://vault.rufonex.com.br
```

Environment:

```text
Self-hosted
```

It must NOT remain pointed at:

```text
bitwarden.com
```

If the extension reports that the master password is invalid while the Web
Vault works, first verify the selected server environment.

---

# 26. Two-Factor Authentication

Current second factor:

```text
TOTP
```

TOTP was configured and successfully tested.

The recovery code is stored outside Vaultwarden.

The recovery code MUST NOT be:

- committed to Git
- stored in the Vaultwarden vault itself
- placed in the Docker `.env`
- stored in a public/shared location

The current deployment intentionally uses TOTP only.

WebAuthn is not required unless intentionally added later.

---

# 27. Vaultwarden Registration Policy

Current policy:

```text
SIGNUPS_ALLOWED=false
INVITATIONS_ALLOWED=false
```

The first account was created during the initial bootstrap while signup was
temporarily enabled.

After account creation, signup was disabled.

This prevents arbitrary users from registering on the public endpoint.

---

# 28. Vaultwarden Environment

Current environment file:

```text
/opt/vaultwarden/.env
```

This file is protected and is NOT intended to be committed if it contains
deployment secrets or environment-specific sensitive values.

At minimum, it contains configuration related to:

```text
DOMAIN
SIGNUPS_ALLOWED
BIND_ADDRESS
```

The production domain is:

```text
https://vault.rufonex.com.br
```

The environment file must be reviewed carefully before being added to Git.

Use an example file such as:

```text
.env.example
```

for version control.

---

# 29. Backup Strategy

Current backup architecture:

```text
Vaultwarden
    |
    +--> native SQLite backup
    |
    +--> complete /data archive
    |
    +--> SHA-256 checksum
    |
    +--> local backup storage
```

Current local backup directory:

```text
/var/backups/vaultwarden
```

Temporary staging directory:

```text
/var/lib/vaultwarden-backup
```

Backup script:

```text
/usr/local/sbin/vaultwarden-backup
```

---

# 30. Native Vaultwarden SQLite Backup

The Vaultwarden binary provides a native backup command:

```bash
docker exec vaultwarden /vaultwarden backup
```

This produces a timestamped SQLite backup inside `/data`.

Example:

```text
db_20260922_180023.sqlite3
```

The native backup mechanism is preferred over blindly copying the live SQLite
database while it is being modified.

---

# 31. Complete Backup Archive

The backup script creates a complete archive containing the important
persistent Vaultwarden state.

The archive currently preserves:

```text
db.sqlite3
rsa_key.pem
icon_cache/
```

and will also preserve additional persistent files/directories that may appear
in `/data` in the future, subject to the script's exclusion rules.

The archive intentionally excludes:

```text
db.sqlite3-wal
db.sqlite3-shm
db_*.sqlite3
tmp/
```

The database included in the final archive is the consistent SQLite copy
generated by Vaultwarden's native backup mechanism.

---

# 32. Backup Integrity

Each backup produces:

```text
vaultwarden_YYYYMMDD_HHMMSS.tar.gz
```

and:

```text
vaultwarden_YYYYMMDD_HHMMSS.tar.gz.sha256
```

Example:

```text
vaultwarden_20260922_155527.tar.gz
vaultwarden_20260922_155527.tar.gz.sha256
```

The archive is validated using:

```bash
tar -tzf archive.tar.gz
```

and:

```bash
sha256sum -c archive.tar.gz.sha256
```

A real backup test produced:

```text
TAR OK
checksum OK
```

---

# 33. Backup Restore Test

A real restore test was already performed.

Process:

```text
production backup
        |
        v
separate restore directory
        |
        v
temporary Vaultwarden container
        |
        v
Vaultwarden started
        |
        v
Web Vault accessible
        |
        v
account data recovered
```

The restore instance successfully started and presented the Vaultwarden login
screen.

This proves that the backup is not merely syntactically valid; it is capable of
reconstructing a functional Vaultwarden instance.

The restore test instance was then removed.

Temporary restore resources must not remain on the production server.

---

# 34. Backup Script Behavior

The backup script performs approximately the following process:

```text
1. Verify Vaultwarden container exists.
2. Verify container is running.
3. Verify application is healthy.
4. Run the native Vaultwarden SQLite backup.
5. Stop Vaultwarden cleanly.
6. Copy required persistent non-live database data.
7. Insert the consistent SQLite backup as db.sqlite3.
8. Create compressed archive.
9. Generate SHA-256 checksum.
10. Validate archive content.
11. Remove temporary database backup files.
12. Start Vaultwarden.
13. Wait for Vaultwarden to become healthy.
14. Validate checksum.
15. Prune old backups older than RETENTION_DAYS=10 and matching .sha256 files.
16. Report success.
```

The script uses a lock to prevent concurrent backups.

The healthcheck wait logic was intentionally increased because the Vaultwarden
Docker image's healthcheck does not necessarily run immediately after container
startup.

The current health wait period has been adjusted to tolerate the image's
healthcheck interval.

---

# 35. Backup Lessons Learned

Important:

Do NOT assume:

```text
docker container running
```

means:

```text
backup completed correctly
```

Also do NOT assume:

```text
.tar.gz exists
```

means:

```text
backup is restorable
```

The correct validation chain is:

```text
backup
    ->
archive
    ->
checksum
    ->
archive content
    ->
restore
    ->
application startup
    ->
login test
```

---

# 36. Local Backup Retention

Local backups are stored in:

```text
/var/backups/vaultwarden
```

### Retention Policy:

- Configured retention window: **10 days** (`RETENTION_DAYS=10`).
- Pruning runs strictly **after** the new backup has been created and verified (`tar -tzf` and `sha256sum -c`).
- It scans `/var/backups/vaultwarden` for archives older than 10 days (`vaultwarden_*.tar.gz`).
- For each expired archive, both the archive file and its matching `.sha256` checksum file are removed.

### Controlled Validation Test:

- A controlled test was performed by creating a **FICTITIOUS** backup pair (`vaultwarden_*.tar.gz` and `.sha256`) with a simulated age of 15 days, created exclusively for validation purposes.
- A real execution of `vaultwarden-backup.service` was triggered via `systemctl start vaultwarden-backup.service`.
- The script successfully created the new backup and validated its SHA-256 checksum.
- The fictitious expired backup pair was successfully removed by `prune_old_backups`.
- All genuine real backups remained intact.
- Vaultwarden completed the execution in `running/healthy` state.

---

# 37. Backup Automation

Backup execution is automated using systemd timer and service units.

Architecture:

```text
vaultwarden-backup.timer
        ↓
vaultwarden-backup.service
        ↓
/usr/local/sbin/vaultwarden-backup
        ↓
native SQLite backup
        ↓
archive + SHA-256
        ↓
/var/backups/vaultwarden/
```

### Components:

- Service Unit: `/etc/systemd/system/vaultwarden-backup.service`
  - `Type=oneshot`
  - `Requires=docker.service`
  - `After=docker.service`
  - `ExecStart=/usr/local/sbin/vaultwarden-backup`
  - `UMask=0077`
  - `NoNewPrivileges=true`
  - `TimeoutStartSec=20min`

- Timer Unit: `/etc/systemd/system/vaultwarden-backup.timer`
  - Schedule: `OnCalendar=*-*-* 03:00:00` (daily at 03:00)
  - `Persistent=true`
  - `WantedBy=timers.target`
  - State: enabled and active

### Validation Status:

- The backup script was previously validated.
- `vaultwarden-backup.service` was created as `Type=oneshot`, depending on `docker.service`.
- `vaultwarden-backup.timer` was created, enabled, and is active waiting for the next execution.
- Manual execution of the service was performed successfully (`systemctl start vaultwarden-backup.service`).
- The backup `vaultwarden_20260922_173509.tar.gz` was created successfully.
- The corresponding `.sha256` checksum file was created and validated.
- Vaultwarden finished the process in `running/healthy` state.
- The next timer execution was shown by systemd for 03:00 of 2026-09-23.
- Automatic execution at the scheduled time has NOT yet been observed, therefore it is not documented as a completed automatic test.

Important:

Currently, there is NO centralized monitoring server (such as Zabbix) deployed in production for this environment.
Operational verification relies directly on systemd service logs (`journalctl -u vaultwarden-backup.service`), timer inspection (`systemctl list-timers`), and verifying generated archive files and checksums in `/var/backups/vaultwarden/`.
The absence of proactive centralized alerting is a known limitation, and centralized backup monitoring via Zabbix is categorized as a future improvement / evolution, not a prerequisite for local backup.

---

# 38. Off-site Backup — Pending

Off-site backup is NOT yet finalized.

Target architecture:

```text
Vaultwarden
    |
    v
local backup archive
    |
    v
Restic
    |
    v
encrypted repository
    |
    v
OCI Object Storage
```

OCI will act as an independent off-site recovery location.

The off-site backup must be encrypted before data leaves the local
environment.

---

# 39. OCI Backup Requirements

When OCI backup is implemented:

- Use a dedicated Object Storage bucket.
- Bucket must remain private.
- Do not use public object access.
- Use a dedicated credential with minimum required permissions.
- Do not store OCI secrets in Git.
- Do not place OCI API keys in the repository.
- Do not place OCI secrets directly in documentation.
- Use client-side encrypted backup storage.
- Test restoring a backup directly from OCI.

The OCI backup implementation is NOT yet complete.

---

# 40. Disaster Recovery Objective

The intended disaster recovery path is:

```text
VM destroyed
    |
    v
Create new Debian VM
    |
    v
Configure hostname/network
    |
    v
Install Docker
    |
    v
Restore versioned configuration
    |
    v
Install/start Vaultwarden
    |
    v
Restore backup from OCI
    |
    v
Restore persistent data
    |
    v
Configure Cloudflare Tunnel
    |
    v
https://vault.rufonex.com.br
    |
    v
Login + TOTP
```

The system should NOT depend on the original VM surviving.

---

# 41. Proxmox Backup Strategy

A Proxmox/PBS backup of the VM was taken before Docker/Vaultwarden production
deployment.

That backup represents the baseline VM state after:

- Debian installation
- system updates
- network configuration
- firewall hardening
- SSH hardening
- QEMU Guest Agent

The Proxmox/PBS backup was validated successfully.

This gives a second recovery layer:

```text
Application backup
    +
VM backup
```

These are complementary.

---

# 42. Recovery Layers

The intended recovery hierarchy is:

### Layer 1 — Application recovery

Restore:

```text
Vaultwarden /data
```

from the application backup.

### Layer 2 — VM recovery

Restore the complete VM from Proxmox/PBS.

### Layer 3 — Site disaster

Rebuild the VM elsewhere and restore the encrypted off-site backup.

---

# 43. Security Principles

This deployment follows these principles:

1. No direct Internet port-forwarding to Vaultwarden.
2. Cloudflare Tunnel is the public ingress mechanism.
3. Vaultwarden has a dedicated VM.
4. Host firewall defaults to DROP on inbound traffic.
5. SSH uses public-key authentication.
6. Root SSH login is disabled.
7. Password authentication over SSH is disabled.
8. Only the management LAN can access SSH.
9. Only Cloudflared can access the published Vaultwarden port.
10. Public signup is disabled.
11. Invitations are disabled.
12. TOTP is enabled.
13. Docker uses `no-new-privileges`.
14. Docker logs have controlled rotation.
15. Vaultwarden image is pinned by digest.
16. Persistent application data is stored outside the container filesystem.
17. Backups are validated.
18. Restore has been tested.
19. Secrets are not stored in Git.

---

# 44. Git Security Rules

NEVER commit:

```text
.env
```

when it contains secrets or environment-specific sensitive values.

NEVER commit:

```text
*.pem
*.key
*.secret
```

when they contain private keys or credentials.

NEVER commit:

```text
db.sqlite3
*.sqlite3
*.sqlite3-wal
*.sqlite3-shm
```

NEVER commit:

```text
*.tar.gz
```

when they are Vaultwarden backups.

NEVER commit:

- Cloudflare tunnel tokens
- OCI access keys
- OCI secret keys
- TOTP secrets
- TOTP recovery codes
- Vaultwarden recovery codes
- SSH private keys
- real user credentials

Public configuration templates are acceptable.

---

# 45. Recommended Git Structure

Target repository structure:

```text
vaultwarden-infra/
|
├── README.md
├── PROJECT_CONTEXT.md
├── ARCHITECTURE.md
├── DECISIONS.md
├── SECURITY.md
|
├── docker/
│   ├── docker-compose.yaml
│   ├── daemon.json
│   └── .env.example
|
├── firewall/
│   ├── vaultwarden-docker-firewall
│   └── vaultwarden-docker-firewall.service
|
├── backup/
│   ├── vaultwarden-backup
│   ├── vaultwarden-backup.service
│   └── vaultwarden-backup.timer
|
├── docs/
│   ├── 01-vm-provisioning.md
│   ├── 02-network.md
│   ├── 03-firewall.md
│   ├── 04-ssh-hardening.md
│   ├── 05-docker.md
│   ├── 06-vaultwarden.md
│   ├── 07-cloudflare.md
│   ├── 08-backup.md
│   ├── 09-restore.md
│   ├── 10-maintenance.md
│   └── 11-disaster-recovery.md
|
└── diagrams/
    └── architecture.mmd
```

The exact structure may evolve, but secrets and production data must remain
outside Git.

---

# 46. Current Production State

## Completed

```text
Debian 13 VM                         DONE
Static/reserved IP                   DONE
SSH access                           DONE
SSH key authentication               DONE
SSH hardening                        DONE
Root SSH disabled                    DONE
Firewall base                        DONE
Firewall persistence                 DONE
Firewall reboot test                 DONE
QEMU Guest Agent                     DONE
System time synchronization          DONE
Docker Engine                        DONE
Docker Compose                       DONE
Docker daemon hardening              DONE
Docker firewall integration          DONE
Vaultwarden container                DONE
SQLite                               DONE
Persistent /data                     DONE
Image digest pinning                 DONE
Cloudflare Tunnel                    DONE
Public HTTPS domain                  DONE
Vaultwarden account                  DONE
Login/logout test                    DONE
Bitwarden browser extension         DONE
TOTP                                  DONE
Signup disabled                      DONE
Backup script                        DONE
Backup integrity test                DONE
Backup restore test                  DONE
Systemd backup timer                 DONE
Backup retention automation          DONE
Proxmox/PBS baseline backup          DONE
```

---

# 47. Remaining Work / Future Improvements

The local backup is already implemented, scheduled via systemd, validated, with 10-day retention and restore tested.
Centralized monitoring via Zabbix is NOT a prerequisite for local backup completion and is classified as a future improvement / evolution, as there is currently no Zabbix server in production for this environment.

Future improvements / evolutions:

```text
Backup monitoring via Zabbix (future evolution; no Zabbix server currently exists)
Off-site backup
OCI Object Storage repository
Encrypted backup outside the local environment
Restore from off-site / OCI
Full disaster recovery test
OCI credential hardening
Regular Vaultwarden update procedure
Regular restore verification procedure
Formal maintenance runbook
```

Do not document these as completed until they are actually implemented and
tested.

---

# 48. Operational Rules for Future Changes

Before changing production:

```text
1. Read this PROJECT_CONTEXT.md.
2. Check the current production state.
3. Create a backup when appropriate.
4. Make the smallest required change.
5. Validate configuration before restarting services.
6. Test functionality.
7. Check logs.
8. Check health status.
9. Update documentation.
10. Commit configuration changes to Git.
```

Never make undocumented manual changes that cannot be reproduced.

---

# 49. Version Upgrade Policy

Vaultwarden upgrades must be intentional.

Do NOT blindly run:

```bash
docker compose pull
```

against an unpinned `latest` image.

Upgrade process:

```text
1. Review Vaultwarden release notes.
2. Check whether the version contains security fixes.
3. Create a current application backup.
4. Confirm backup integrity.
5. Update the image digest intentionally.
6. Pull the new image.
7. Recreate the container.
8. Wait for healthcheck.
9. Verify Web Vault.
10. Verify browser extension.
11. Verify login/TOTP.
12. Check logs.
13. Document the version change.
```

Rollback must remain possible through the previous known-good image digest
and backup.

---

# 50. Troubleshooting Principles

When diagnosing Vaultwarden:

Check in this order:

```text
Network
    ->
Cloudflare Tunnel
    ->
host port
    ->
DOCKER-USER
    ->
Docker port mapping
    ->
container
    ->
application health
    ->
SQLite
```

Useful commands:

```bash
sudo docker compose -f /opt/vaultwarden/docker-compose.yaml ps
```

```bash
sudo docker logs --tail=100 vaultwarden
```

```bash
sudo docker inspect vaultwarden \
  --format 'Status={{.State.Status}} Health={{.State.Health.Status}}'
```

```bash
sudo ss -lntp
```

```bash
sudo iptables -S
```

```bash
sudo iptables -S DOCKER-USER
```

```bash
sudo iptables -S VW-DOCKER
```

```bash
curl -i http://192.168.15.200:8080/alive
```

---

# 51. Important Operational Distinction

These are different concepts:

```text
VM availability
Docker availability
Container availability
Application health
External reachability
Authentication availability
Backup availability
Restore availability
```

A successful healthcheck does not prove the external Cloudflare route works.

A successful Cloudflare request does not prove the backup works.

A successful backup creation does not prove restore works.

Each layer must be tested independently.

---

# 52. Current Mental Model

The most important mental model for this infrastructure is:

```text
                       PUBLIC ACCESS
                            |
                      Cloudflare TLS
                            |
                     Cloudflare Tunnel
                            |
                       .253 LXC
                            |
                       HTTP :8080
                            |
                       .200 VM
                            |
                        Docker
                            |
                     DOCKER-USER
                            |
                     VW-DOCKER
                            |
                      Vaultwarden
                            |
                          SQLite
                            |
                        /data
                            |
                  +---------+---------+
                  |                   |
             local backup         off-site backup
                                      |
                                     OCI
```

The security boundary exists at several layers:

```text
Cloudflare
    +
network
    +
host firewall
    +
Docker firewall
    +
container
    +
Vaultwarden authentication
    +
TOTP
    +
encrypted vault
```

---

# 53. OpenCode Instructions

When working on this repository, OpenCode must follow these rules:

- Treat `PROJECT_CONTEXT.md` as the authoritative project context.
- Do not invent infrastructure that is not documented.
- Do not replace existing architecture without explicit justification.
- Do not introduce additional services merely for convention or appearance.
- Do not remove security controls without documenting why.
- Do not store secrets in the repository.
- Never generate or commit private keys.
- Never generate or commit real `.env` files containing secrets.
- Never generate or commit real Vaultwarden database files.
- Never generate or commit real backups.
- Clearly distinguish CURRENT, PLANNED, and DEPRECATED configuration.
- When modifying production configuration, first validate the change.
- Prefer reproducible configuration over manual procedures.
- Keep production configuration and documentation synchronized.
- Update this project context when an architectural decision changes.
- Never claim a backup, restore test, security control, or monitoring system is
  implemented unless it has actually been tested.
- When uncertain about the current state, inspect the actual host/configuration
  rather than guessing.
