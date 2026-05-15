---
name: nas
description: NAS server management with Docker Compose. Covers host environment setup, Docker installation and configuration, storage and network planning, user and permission management. Use when building a NAS, deploying self-hosted services, configuring a Docker home server, or initializing a new server.
metadata:
  author: 孟庆贺 https://mengqinghe.com
  version: "0.1.0"
---

# NAS Management Skill

Provides the agent with NAS server management capabilities.

## Overview

A complete, secure, best-practice NAS management plan. Services are deployed via Docker Compose to preserve host stability and service maintainability.

## When to Activate

Activate this skill when the user asks to:

- Initialize or prepare a new NAS / server
- Install Docker and Docker Compose
- Plan storage directory layout
- Configure server networking and firewall
- Create service accounts and plan permissions
- Deploy any self-hosted service with Docker Compose

## Work Cycle Rules

### Execution Principles
- **Idempotent-first**: All operations must be repeatable without side effects. Prefer check-then-execute patterns.
- **Least privilege**: Use `sudo` only when necessary; operate with the minimum required permissions.
- **Check before change**: Verify current state before any mutation to avoid duplicate installs or overwriting existing configs.
- **Security baseline**: Operations involving network port exposure, key file permissions, or inter-container communication isolation must pass a security review before proceeding.

### Phase Progression
- Execute phases strictly in order. Do not advance until all steps in the current phase pass verification.
- Every sub-task must produce explicit verification output (exit code, file existence, service status, etc.).

### Verification Rules
- Check results immediately after each operation.
- Service operations: confirm running state via `docker compose ps` or `docker ps`.
- Compose files: validate syntax with `docker compose config --quiet`.
- Files/directories: confirm existence via `ls` / `stat` / `test`.
- Port listening: confirm via `ss -tlnp`.
- Sensitive files (keys, certs, env files): confirm permissions via `stat -c %a`, must not exceed `600`.

### Error Handling
- Stop the current phase immediately on single-step failure; output the failure reason and context.
- When dependencies are missing, fix the dependency first — do not skip.
- For errors that cannot be auto-resolved, clearly inform the user with manual remediation steps.
- Mutating operations (config files, compose files) must retain an original backup for fast rollback.

### State Tracking
- After each phase completes, record the phase name and completion time in `/opt/nas-stack/.state`.
- On resume, read the state file to skip completed phases, enabling checkpoint-restart.

## Functional Modules

### Host Environment Management

Prepare the host OS before deploying any containerized services. Execute in order; verify each step before proceeding.

#### 1. OS Base Preparation
- System update: `apt update && apt upgrade` (Debian/Ubuntu) or `yum update` (CentOS/RHEL)
- Enable automatic security updates: `unattended-upgrades` (Debian/Ubuntu) or `yum-cron` (CentOS/RHEL)
- Install base dependencies: `curl`, `wget`, `git`, `htop`, `vim`
- Set timezone and locale

**Verify**: `apt list --upgradable 2>/dev/null | wc -l` or `yum check-update` — confirm no pending updates.

#### 2. Docker Runtime
- Install Docker Engine (community edition) per official docs
- Install Docker Compose v2 plugin
- Configure Docker daemon (`/etc/docker/daemon.json`):
  - Log rotation (`max-size: 10m`, `max-file: 3`)
  - Storage driver (`overlay2` recommended)
  - Registry mirror (if in mainland China)
- Security hardening:
  - Enable user namespace isolation (`userns-remap: default`)
  - Restrict inter-container communication (`icc: false`)
  - Limit default capabilities
- Add management user to `docker` group: `sudo usermod -aG docker $USER`

**Verify**: `docker info` and `docker compose version` produce normal output.

#### 3. Storage Layout
- Confirm system disk is separate from data disk
- Create core directory structure:
  - `/data/docker` — container persistent data
  - `/data/media` — media library
  - `/data/backup` — backups
  - `/opt/nas-stack` — Compose manifests and working directory
- Optional: plan LVM / RAID / ZFS

**Verify**: `test -d /data/docker && test -d /opt/nas-stack && echo "OK"`

#### 4. Network Foundation
- Configure a static IP (DHCP drift will break service reachability)
- Firewall rules: default-deny inbound, allow only required NAS service ports
- Create a Docker custom bridge network for inter-service communication

**Verify**: `ip addr show` confirms static IP; `ss -tlnp` confirms no unexpected open ports.

#### 5. Users & Permissions
- Create a dedicated service account (e.g. `nas-svc`)
- Plan UID/GID mappings to keep SMB/NFS permissions consistent across services
- Configure SSH key authentication; disable password login

**Verify**: `id nas-svc` confirms the user exists; `stat -c %a ~/.ssh/authorized_keys` confirms permissions are `600`.

#### 6. Basic Guarantees
- Enable Docker at boot: `systemctl enable docker`
- Configure disk space and resource baseline monitoring
- Reserve periodic snapshot / backup strategy

**Verify**: `systemctl is-enabled docker` returns `enabled`.

## Security Checklist

After completing host environment management, confirm each item:

- [ ] Firewall default policy is DROP/deny
- [ ] Only necessary service ports are open
- [ ] SSH password authentication is disabled
- [ ] Docker daemon has `userns-remap` enabled
- [ ] All sensitive file permissions are ≤ 600
- [ ] Service account has no `sudo` privileges
- [ ] Docker inter-container communication is restricted (`icc: false`)
