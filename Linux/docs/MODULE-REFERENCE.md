# Linux Module Quick Reference

Fast lookup card for all 20 modules. Use this during a competition.

---

## One-Liner Reference

```bash
# All commands assume you are in Linux/
sudo bash modules/<module>.sh
```

| Module | One-Line Summary | Requires Input? | Destructive? |
|--------|-----------------|-----------------|--------------|
| `lockAccounts.sh` | Lock all users except you + new admin | Yes (new admin name + password) | Yes |
| `sshConfig.sh` | Harden SSH config + restart sshd | No | No (auto-backs up) |
| `firewallRules.sh` | Default-DROP iptables firewall | No | Yes (breaks open ports) |
| `rotatePasswords.sh` | Change passwords for all real users | Yes (per-user prompts) | No |
| `hardenSysctl.sh` | Apply kernel hardening settings | No | No |
| `patchPrivEsc.sh` | Patch pwnkit + CVE-2023-32233 | No | No |
| `auditServices.sh` | Report running/enabled services | No | No (read-only) |
| `auditStartupTasks.sh` | Check persistence locations | No | No (read-only) |
| `bulkDisableServices.sh` | Stop/disable risky legacy services | No | Yes |
| `checkPackageIntegrity.sh` | Verify package file checksums | No | No (read-only) |
| `checkPermissions.sh` | Verify + fix file permissions | No | No |
| `configureAppArmor.sh` | Install + enforce AppArmor profiles | No | No |
| `.configureSELinux.sh` | SELinux setup (RHEL only) | No | No |
| `cronControl.sh` | Delete ALL cron jobs | No | Yes (irreversible) |
| `groupMemberships.sh` | Audit sudo/wheel/adm groups | No | Optional |
| `permissionAudit.sh` | Fix /etc/passwd, shadow, group perms | No | No |
| `removeUnusedPackages.sh` | Remove netcat, gcc, telnet, etc. | No | Yes (removes packages) |
| `searchSSN.sh` | Scan /home for SSN patterns | No | No (read-only) |
| `sudoCheck.sh` | List all sudo users | No | No (read-only) |
| `updatePackages.sh` | Update all system packages | No | No |

---

## Safe vs. Destructive

### Safe to run anytime (read-only, no side effects)
- `auditServices.sh`
- `auditStartupTasks.sh`
- `checkPackageIntegrity.sh`
- `searchSSN.sh`
- `sudoCheck.sh`

### Modifies configs (reversible)
- `sshConfig.sh` — backup at `/etc/ssh/sshd_config.backup.<timestamp>`
- `hardenSysctl.sh` — revert with `sysctl -w <key>=<original-value>`
- `checkPermissions.sh`
- `permissionAudit.sh`
- `groupMemberships.sh`

### High impact — review before running
- `lockAccounts.sh` — locks other users; use `passwd -u` + `chage -E -1` to reverse
- `firewallRules.sh` — breaks any port not explicitly allowed; flush with `iptables -F`
- `bulkDisableServices.sh` — stops services; re-enable with `systemctl enable --now <svc>`
- `removeUnusedPackages.sh` — removes packages; reinstall with your package manager
- `cronControl.sh` — **irreversible** — deletes all cron jobs with no backup

---

## Script Compatibility

| Module | Debian/Ubuntu | RHEL/CentOS/Fedora | Arch | Alpine | Notes |
|--------|:---:|:---:|:---:|:---:|-------|
| `lockAccounts.sh` | ✓ | ✓ | ✓ | ✓ | Universal |
| `sshConfig.sh` | ✓ | ✓ | ✓ | ✓ | Requires sshd installed |
| `firewallRules.sh` | ✓ | ✓ | ✓ | ✓ | Requires iptables |
| `rotatePasswords.sh` | ✓ | ✓ | ✓ | ✓ | Universal |
| `hardenSysctl.sh` | ✓ | ✓ | ✓ | ✓ | Universal |
| `patchPrivEsc.sh` | ✓ | ✓ | ✓ | ✓ | Skips if files absent |
| `auditServices.sh` | ✓ | ✓ | ✓ | Partial | Requires systemd |
| `configureAppArmor.sh` | ✓ | Partial | Partial | — | Best on Ubuntu/Debian |
| `.configureSELinux.sh` | — | ✓ | — | — | RHEL/CentOS only |
| `checkPackageIntegrity.sh` | ✓ | ✓ | ✓ | ✓ | Uses native pkg manager |
| `updatePackages.sh` | ✓ | ✓ | ✓ | ✓ | Auto-detects pkg manager |
| `removeUnusedPackages.sh` | ✓ | ✓ | — | ✓ | No Arch support |

---

## Kernel Hardening Settings Applied by hardenSysctl.sh

| Setting | Value | Effect |
|---------|-------|--------|
| `net.ipv4.tcp_syncookies` | 1 | SYN flood protection |
| `net.ipv4.tcp_synack_retries` | 2 | Reduce SYN-ACK retries |
| `net.ipv4.tcp_rfc1337` | 1 | Protection against TIME_WAIT assassination |
| `net.ipv4.icmp_ignore_bogus_error_responses` | 1 | Ignore bad ICMP errors |
| `net.ipv4.conf.all.accept_redirects` | 0 | Ignore ICMP redirects |
| `net.ipv4.icmp_echo_ignore_all` | 1 | Ignore all pings |
| `kernel.core_uses_pid` | 1 | Core dump files include PID |
| `kernel.kptr_restrict` | 2 | Hide kernel addresses from all users |
| `kernel.perf_event_paranoid` | 2 | Restrict perf to root |
| `kernel.randomize_va_space` | 2 | Full ASLR |
| `kernel.sysrq` | 0 | Disable magic SysRq key |
| `kernel.yama.ptrace_scope` | 2 | Restrict ptrace to root only |
| `fs.protected_hardlinks` | 1 | Prevent hardlink attacks |
| `fs.protected_symlinks` | 1 | Prevent symlink attacks |
| `fs.suid_dumpable` | 0 | No core dumps from SUID programs |
| `kernel.unprivileged_userns_clone` | 0 | Block unprivileged user namespaces |
| `fs.protected_fifos` | 2 | Protect named pipes |
| `fs.protected_regular` | 2 | Protect regular files |

---

## SSH Settings Applied by sshConfig.sh

| Setting | Value |
|---------|-------|
| Port | 22 (unchanged) |
| Protocol | 2 |
| AddressFamily | inet (IPv4 only) |
| PermitRootLogin | no |
| PasswordAuthentication | yes |
| PermitEmptyPasswords | no |
| PubkeyAuthentication | no |
| HostbasedAuthentication | no |
| IgnoreRhosts | yes |
| LoginGraceTime | 30s |
| MaxAuthTries | 3 |
| MaxSessions | 3 |
| MaxStartups | 5:50:10 |
| ClientAliveInterval | 300 |
| ClientAliveCountMax | 2 |
| X11Forwarding | no |
| AllowAgentForwarding | no |
| AllowTcpForwarding | no |
| PermitTunnel | no |
| GatewayPorts | no |
| Compression | no |
| LogLevel | VERBOSE |
| StrictModes | yes |
| UsePAM | yes |

**Allowed Ciphers:** chacha20-poly1305, aes256-gcm, aes128-gcm, aes256-ctr, aes192-ctr, aes128-ctr

**Allowed MACs:** hmac-sha2-512-etm, hmac-sha2-256-etm, hmac-sha2-512, hmac-sha2-256

**Allowed Key Exchange:** curve25519-sha256, diffie-hellman-group-exchange-sha256

---

## Services Flagged by auditServices.sh

The script checks for these services and flags any that are running or enabled:

`telnet`, `rsh`, `rlogin`, `vsftpd`, `ftpd`, `apache2`, `httpd`, `nginx`, `mysql`, `mariadb`, `postgresql`, `samba`, `smbd`, `nmbd`, `nfs-server`, `rpcbind`, `snmpd`, `tftpd`, `xinetd`, `cups`, `avahi-daemon`, `bluetooth`, `docker`, `postfix`, `dovecot`, `bind9`, `named`, `squid`

> Note: These are not always bad — some may be required scored services. The audit flags them for review, not automatic removal.

---

## Useful Commands for CCDC

```bash
# See all logged-in users
who

# See recent login attempts
sudo last
sudo lastb   # failed logins

# Check auth log for SSH activity
sudo tail -50 /var/log/auth.log | grep sshd

# See all open network connections
sudo ss -tulpn

# List all processes with their users
ps aux

# Find all SUID binaries (potential escalation paths)
find / -perm -4000 -type f 2>/dev/null

# Check sudo rules
sudo cat /etc/sudoers
sudo ls /etc/sudoers.d/

# View iptables rules
sudo iptables -L -v -n --line-numbers

# Current iptables rules (save-friendly format)
sudo iptables-save
```
