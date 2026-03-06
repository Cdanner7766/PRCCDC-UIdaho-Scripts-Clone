# Linux Hardening Suite

A modular bash hardening toolkit designed for fast deployment during CCDC competitions.
Supports Debian/Ubuntu, RHEL/CentOS/Fedora, Arch, Alpine, and openSUSE.

---

## How to Use

### Run a single module
```bash
cd Linux/
sudo bash main.sh --module sshConfig
```

### List all available modules
```bash
bash main.sh --list
```

### Run all modules at once
```bash
sudo bash main.sh --all
```

> **Warning:** `--all` runs every module including ones that wipe cron jobs and remove packages.
> Only use `--all` if you know the machine's role and have reviewed each module.

### Run a module directly
```bash
sudo bash modules/lockAccounts.sh
```

---

## Module Overview

### Priority Order for CCDC

Run modules in this order at the start of a round:

| Order | Module | Why |
|-------|--------|-----|
| 1 | `lockAccounts.sh` | Lock out attackers from other accounts immediately |
| 2 | `sshConfig.sh` | Harden your remote access before doing anything else |
| 3 | `firewallRules.sh` | Cut off unwanted network access |
| 4 | `rotatePasswords.sh` | Change all passwords before attackers can use them |
| 5 | `auditServices.sh` | Understand what is running on the system |
| 6 | `sudoCheck.sh` | Know who has elevated privileges |
| 7 | `groupMemberships.sh` | Audit sensitive group membership |
| 8 | `hardenSysctl.sh` | Harden the kernel |
| 9 | `patchPrivEsc.sh` | Patch known local privilege escalation CVEs |
| 10 | `removeUnusedPackages.sh` | Remove attacker tools |
| 11 | `bulkDisableServices.sh` | Disable unnecessary/dangerous services |
| 12 | `cronControl.sh` | Remove attacker persistence via cron |
| 13 | `auditStartupTasks.sh` | Check for other persistence mechanisms |
| 14 | `permissionAudit.sh` | Fix critical file permissions |
| 15 | `checkPermissions.sh` | General permission check |
| 16 | `checkPackageIntegrity.sh` | Detect tampered files |
| 17 | `configureAppArmor.sh` | Enable mandatory access control |
| 18 | `updatePackages.sh` | Patch vulnerabilities |
| 19 | `searchSSN.sh` | Find PII (scored in CCDC) |

---

## All Modules — Detailed Descriptions

### lockAccounts.sh
Locks every human user account on the system **except**:
- The user running the script (you)
- A new admin account you create during the script

Uses `passwd -l` to lock passwords and `chage -E 0` to expire the account, which also blocks SSH key login. To unlock a user later:
```bash
sudo passwd -u <username>
sudo chage -E -1 <username>
```

---

### sshConfig.sh
Applies a hardened SSH server configuration to `/etc/ssh/sshd_config`. Automatically backs up the original config before making any changes.

**What it changes:**
- Disables root login
- Sets max auth tries to 3, max sessions to 3
- Disconnects idle clients after 10 minutes (300s × 2 keepalives)
- Disables X11 forwarding, agent forwarding, TCP forwarding
- Restricts to strong ciphers (ChaCha20, AES-256-GCM), MACs, and key exchange algorithms
- Enables verbose logging to auth.log
- Installs a legal warning banner
- Sets correct permissions on all `.ssh/` directories

**Recovery (if locked out):**
```bash
sudo cp /etc/ssh/sshd_config.backup.<timestamp> /etc/ssh/sshd_config
sudo systemctl restart sshd
```

---

### firewallRules.sh
Configures `iptables` with a **default-DROP** policy — blocks everything not explicitly allowed.

**What is allowed (outbound):**
- SSH (port 22) — incoming and established
- DNS (UDP/TCP port 53)
- HTTP (port 80) and HTTPS (port 443)
- ICMP outbound
- Loopback interface

**What is blocked:**
- All other inbound and outbound traffic

> **Important:** If the machine runs services (web server, database, custom app), you must add rules before running this script or services will stop working. See the tutorial for adding custom rules.

---

### rotatePasswords.sh
Walks through every user with a real shell (bash/sh/zsh) and UID >= 1000, plus root. For each user, prompts you to change their password interactively.

Must be run as root:
```bash
sudo bash modules/rotatePasswords.sh
```

---

### hardenSysctl.sh
Appends kernel hardening settings to `/etc/sysctl.conf` and applies them immediately with `sysctl -p`.

**Settings applied:**
- `tcp_syncookies = 1` — SYN flood protection
- `icmp_echo_ignore_all = 1` — machine does not respond to ping
- `accept_redirects = 0` — ignore ICMP redirects (prevents routing attacks)
- `randomize_va_space = 2` — full ASLR (address space randomization)
- `kptr_restrict = 2` — hide kernel pointers from unprivileged users
- `yama.ptrace_scope = 2` — restrict ptrace (debugging) to root only
- `protected_hardlinks/symlinks = 1` — prevent symlink/hardlink attacks
- `suid_dumpable = 0` — prevent core dumps from SUID programs
- `unprivileged_userns_clone = 0` — disable unprivileged user namespaces

---

### patchPrivEsc.sh
Patches two specific local privilege escalation vulnerabilities:

1. **Pwnkit (CVE-2021-4034)** — Removes SUID bit from `/usr/bin/pkexec`
   ```bash
   chmod 0755 /usr/bin/pkexec   # removes the s bit
   ```

2. **CVE-2023-32233** — Disables unprivileged user namespaces
   ```bash
   kernel.unprivileged_userns_clone = 0
   ```

---

### auditServices.sh
Generates a report at `/tmp/service_audit_<timestamp>.txt` showing:
- All currently running services
- All services enabled at boot
- Any of 27 flagged suspicious services (telnet, ftp, docker, bind9, etc.)
- All network ports currently listening (`ss -tulpn`)
- Any failed services

Run this **before** disabling services so you know what's on the machine.

---

### auditStartupTasks.sh
Checks all common places attackers hide persistence:
- `systemctl` enabled services
- `/etc/rc.local`
- `/etc/init.d/`
- User and system crontabs (`crontab -l`, `/var/spool/cron/`, `/etc/cron.*`)
- Shell initialization files (`.bashrc`, `.profile`, `.bash_profile`)

---

### bulkDisableServices.sh
Stops and disables a list of dangerous or unnecessary services:
`xinetd`, `rsh`, `rlogin`, `rexec`, `telnet`, `tftp`, `kdump`, `avahi-daemon`, and others.

---

### checkPackageIntegrity.sh
Verifies the integrity of installed packages by checking file checksums against the package manager's database. This can detect if an attacker has replaced system binaries.

| Distro | Command used |
|--------|-------------|
| Debian/Ubuntu | `debsums` |
| RHEL/Fedora | `rpm -Va` |
| Arch | `pacman -Qkk` |
| Alpine | `apk verify` |

---

### checkPermissions.sh
Verifies permissions and ownership on key files and directories. Fixes mismatches automatically.

---

### configureAppArmor.sh
Installs AppArmor (if missing), enables it in GRUB, and sets all profiles to enforcing mode.
Requires a reboot to take full effect on systems where AppArmor was not previously active.

---

### cronControl.sh
**Destructive — use carefully.** Removes all cron jobs for all users and clears `/etc/cron.*` directories.

Useful when you suspect attacker persistence via cron, but will also remove legitimate scheduled tasks.

---

### groupMemberships.sh
Audits membership in sensitive groups: `sudo`, `wheel`, `adm`. Lists all members and can optionally remove users.

---

### permissionAudit.sh
Checks and fixes permissions on:
- `/etc/passwd` — should be 644
- `/etc/shadow` — should be 640 (root:shadow)
- `/etc/group` — should be 644
- `/etc/gshadow` — should be 640

---

### removeUnusedPackages.sh
Removes: `netcat`, `nc`, `gcc`, `cmake`, `make`, `telnet`

These are tools attackers commonly use for reverse shells and compiling exploits. Supports apt, yum, and apk.

---

### searchSSN.sh
Scans all `.txt` and `.csv` files under `/home/` for Social Security Number patterns (`XXX-XX-XXXX`). Useful for the scored "find the PII" inject in CCDC.

---

### sudoCheck.sh
Lists all users with sudo privileges. Quick way to see who can escalate to root.

---

### updatePackages.sh
Updates all installed packages using the system's native package manager. Supports apt, dnf, yum, zypper, pacman, and apk.

---

## Troubleshooting

**Script says "permission denied"**
```bash
sudo bash modules/<script-name>.sh
# or
chmod +x modules/<script-name>.sh && sudo ./modules/<script-name>.sh
```

**Locked out after running sshConfig.sh**
If you have console access:
```bash
sudo cp /etc/ssh/sshd_config.backup.<timestamp> /etc/ssh/sshd_config
sudo systemctl restart sshd
```
The backup timestamp is shown in the script output.

**firewallRules.sh broke a service**
```bash
# Check current rules
sudo iptables -L -v -n

# Temporarily allow a port (e.g., port 8080 for a web app)
sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT

# Flush all rules (emergency reset — removes all protection)
sudo iptables -F && sudo iptables -P INPUT ACCEPT && sudo iptables -P OUTPUT ACCEPT
```

**hardenSysctl.sh blocked ping**
By design. `icmp_echo_ignore_all = 1` stops the machine from responding to pings.
To re-enable temporarily:
```bash
sudo sysctl -w net.ipv4.icmp_echo_ignore_all=0
```
