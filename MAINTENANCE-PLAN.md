# Ghost Server Maintenance Plan

Server: kithara (158.69.201.160)
Created: 2026-02-02

## Current Status

| Item | Status | Notes |
|------|--------|-------|
| Ubuntu | 25.04 (EOL) | Needs upgrade to 25.10 |
| Disk | 79.9% used | Needs cleanup before upgrade |
| Ghost | Running | Healthy, logs now persistent |
| Backups | None automated | Manual only |

---

## 1. Backup Strategy

### Immediate (before any maintenance)

```bash
# Full Ghost data backup
cd /home/dev/ghost-docker
tar czvf ~/ghost-backup-$(date +%Y%m%d).tar.gz data/

# Copy off-server (run from local machine)
scp ubuntu@kithara:~/ghost-backup-*.tar.gz ./
```

### Ongoing Strategy

**Option A: Simple cron backup to external storage**
```bash
# Daily backup script at /home/dev/ghost-docker/scripts/backup.sh
#!/bin/bash
BACKUP_DIR="/home/dev/backups"
DATE=$(date +%Y%m%d)
mkdir -p $BACKUP_DIR
tar czvf $BACKUP_DIR/ghost-$DATE.tar.gz /home/dev/ghost-docker/data/
# Keep only last 7 days
find $BACKUP_DIR -name "ghost-*.tar.gz" -mtime +7 -delete
```

**Option B: Restic to S3/B2 (recommended for production)**
- Encrypted, deduplicated, versioned
- Can restore individual files or full snapshots
- Integrates with Backblaze B2, S3, etc.

**What to back up:**
- `/home/dev/ghost-docker/data/ghost/` - content, themes, images
- `/home/dev/ghost-docker/data/mysql/` - database
- `/home/dev/ghost-docker/data/logs/` - application logs
- `/home/dev/ghost-docker/.env` - configuration
- `/home/dev/ghost-docker/compose.yml` - infrastructure

**MySQL dump (more portable than raw files):**
```bash
docker exec ghost-docker-db-1 mysqldump -u ghost -p"$PASSWORD" ghost > ghost-db-$(date +%Y%m%d).sql
```

---

## 2. Disk Cleanup

**Check what's using space:**
```bash
sudo du -h --max-depth=2 / 2>/dev/null | sort -hr | head -30
```

**Common culprits:**
```bash
# Old container images
docker image prune -a

# Old logs (system)
sudo journalctl --vacuum-time=7d

# Apt cache
sudo apt clean

# Old kernels (after upgrade)
sudo apt autoremove
```

**Target:** Get disk usage below 60% before OS upgrade

---

## 3. Ubuntu Upgrade (25.04 → 25.10)

### Pre-upgrade checklist
- [ ] Full backup completed and verified
- [ ] Backup copied off-server
- [ ] Disk usage below 70%
- [ ] VPS console access confirmed (not just SSH)
- [ ] Low-traffic time window
- [ ] Rollback plan understood

### Upgrade procedure
```bash
# As ubuntu user with sudo
sudo apt update && sudo apt upgrade -y
sudo reboot  # Apply any kernel updates first

# After reboot, verify Ghost is running
docker ps

# Then upgrade
sudo do-release-upgrade
```

### Post-upgrade checklist
- [ ] Server boots successfully
- [ ] SSH access works
- [ ] Ghost containers running (`docker ps`)
- [ ] Website accessible (https://newsletter.aisafetymontreal.org)
- [ ] Signup flow works (test with burner email)
- [ ] Logs still writing to `/home/dev/ghost-docker/data/logs/`

### Rollback plan
If upgrade fails mid-way:
1. Use VPS provider console to access server
2. Check `/var/log/dist-upgrade/` for errors
3. Worst case: restore from VPS snapshot or rebuild from backup

---

## 4. Security Review

### Check
- [ ] SSH key-only auth (no password login)
  ```bash
  grep PasswordAuthentication /etc/ssh/sshd_config
  ```
- [ ] Firewall enabled
  ```bash
  sudo ufw status
  ```
- [ ] Fail2ban installed and running
  ```bash
  sudo systemctl status fail2ban
  ```
- [ ] No unnecessary ports open
  ```bash
  sudo ss -tlnp
  ```
- [ ] Ghost admin only accessible via HTTPS
- [ ] Mailgun API key not exposed in logs
- [ ] `.env` file has restricted permissions (600)

### Harden
```bash
# If not already done
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable

# Install fail2ban if missing
sudo apt install fail2ban
sudo systemctl enable fail2ban
```

---

## 5. Reliability Review

### Current setup
- [x] Container restart policy: `always`
- [x] Podman restart service: enabled
- [x] User linger: enabled
- [x] Persistent logs: configured
- [x] MySQL healthcheck: configured

### Missing/Recommended
- [ ] **Uptime monitoring** - external service to alert if site goes down
  - Free options: UptimeRobot, Freshping, Healthchecks.io
  - Set up check for https://newsletter.aisafetymontreal.org

- [ ] **Log rotation** - prevent logs from filling disk
  ```yaml
  # Add to ghost service in compose.yml
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "3"
  ```

- [ ] **Database backups** - separate from file backup
  - Daily mysqldump to separate location

- [ ] **Disk space alerts** - get notified before 90%
  ```bash
  # Simple cron check
  df -h / | awk 'NR==2 {if ($5+0 > 85) print "Disk alert: "$5" used"}'
  ```

---

## 6. Additional Recommendations

### Ghost-specific
- [ ] Set `mail.from` config to remove warning in logs
- [ ] Fix theme NQL query error (`tags:[]` filter bug)
- [ ] Review Ghost version - currently 6.14, check for updates

### Infrastructure
- [ ] Consider VPS snapshots before major changes (provider feature)
- [ ] Document the full setup in README for bus factor
- [ ] Set up a staging environment for testing updates

### Monitoring
- [ ] Mailgun delivery monitoring / alerts
- [ ] Ghost error log monitoring (currently empty, good)
- [ ] Set up simple health endpoint check

---

## Execution Order

1. **Tonight (low traffic):**
   - Full backup + copy off-server
   - Disk cleanup
   - Ubuntu upgrade 25.04 → 25.10
   - Verify everything works

2. **This week:**
   - Security review
   - Set up uptime monitoring
   - Fix Ghost theme query error

3. **Soon:**
   - Automated backup strategy
   - Plan upgrade to 26.04 LTS (April 2026)

---

## Quick Reference

```bash
# SSH as ubuntu (has sudo)
ssh kithara-ubuntu-ts

# Containers
docker ps
docker logs ghost-docker-ghost-1

# Ghost logs (persistent)
tail -f /home/dev/ghost-docker/data/logs/https___newsletter_aisafetymontreal_org_production.log

# Restart Ghost only
cd /home/dev/ghost-docker && docker compose up -d ghost

# Full stack restart
cd /home/dev/ghost-docker && docker compose down && docker compose up -d

# Backup
tar czvf ~/ghost-backup-$(date +%Y%m%d).tar.gz /home/dev/ghost-docker/data/
```
