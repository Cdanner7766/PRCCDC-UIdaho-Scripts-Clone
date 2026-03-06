# PRCCDC UIdaho Scripts — CCDC Toolkit

This repository is a collection of hardening scripts, Group Policy Objects (GPOs), and utilities built for the **Pacific Rim Collegiate Cyber Defense Competition (PRCCDC)**. It was originally developed by the University of Idaho CCDC team.

---

## Repository Inventory

```
PRCCDC-UIdaho-Scripts-Clone/
├── Linux/                          # Linux hardening suite (20 modules)
│   ├── main.sh                     # Master control script — run all or individual modules
│   ├── utils/
│   │   └── common.sh               # Shared logging functions
│   └── modules/
│       ├── sshConfig.sh            # SSH server hardening (CRITICAL — run first)
│       ├── lockAccounts.sh         # Lock all user accounts except yours (CRITICAL)
│       ├── firewallRules.sh        # iptables firewall with default-DROP (CRITICAL)
│       ├── rotatePasswords.sh      # Interactive password rotation for all users
│       ├── patchPrivEsc.sh         # Patch pwnkit + CVE-2023-32233
│       ├── hardenSysctl.sh         # Kernel hardening via sysctl
│       ├── auditServices.sh        # Audit running/enabled services
│       ├── auditStartupTasks.sh    # Audit cron, init.d, rc.local, shell configs
│       ├── bulkDisableServices.sh  # Disable dangerous legacy services
│       ├── checkPackageIntegrity.sh# Verify installed package checksums
│       ├── checkPermissions.sh     # Verify and fix file permissions
│       ├── configureAppArmor.sh    # Install and enable AppArmor
│       ├── .configureSELinux.sh    # SELinux setup (RHEL-based systems)
│       ├── cronControl.sh          # Wipe all cron jobs
│       ├── groupMemberships.sh     # Audit sudo/wheel/adm group membership
│       ├── permissionAudit.sh      # Audit /etc/passwd, shadow, group permissions
│       ├── removeUnusedPackages.sh # Remove netcat, gcc, telnet, etc.
│       ├── searchSSN.sh            # Scan /home for SSN patterns
│       ├── sudoCheck.sh            # List all users with sudo privileges
│       └── updatePackages.sh       # Update all system packages
├── Windows/
│   ├── restart.ps1                 # Remote bulk reboot script (PowerShell)
│   └── gpos/                       # 28 GPO backups for Active Directory
│       └── {GUID}/                 # Each GPO in its own folder
├── services/
│   └── sql-docker-compose.yml      # MySQL 8 Docker Compose config
└── broken/
    └── fail2ban.sh                 # Non-functional — do not use
```

---

## Quick-Start: First 15 Minutes of a CCDC Round

When you get access to a Linux box, run these **in order**:

```bash
# 1. Clone this repo (if not already on the machine)
git clone <this-repo-url>
cd PRCCDC-UIdaho-Scripts-Clone/Linux

# 2. Make scripts executable
chmod +x main.sh modules/*.sh

# 3. Lock down accounts FIRST (stops attackers from using other accounts)
sudo bash modules/lockAccounts.sh

# 4. Harden SSH (back up config, apply secure settings)
sudo bash modules/sshConfig.sh

# 5. Apply firewall rules
sudo bash modules/firewallRules.sh

# 6. Rotate all passwords
sudo bash modules/rotatePasswords.sh
```

> **WARNING:** Always keep your current SSH session open while testing SSH changes.
> Test in a second terminal before closing the first.

---

## Tool Categories

### Critical First Response (Run Immediately)

| Script | What It Does |
|--------|-------------|
| `lockAccounts.sh` | Locks every user except you + a new admin you create |
| `sshConfig.sh` | Hardens SSH: disables root login, limits ciphers, sets timeouts |
| `firewallRules.sh` | Default-DROP firewall — only allows SSH, DNS, HTTP/S |
| `rotatePasswords.sh` | Walks you through changing passwords for all real users |

### System Hardening (Run After First Response)

| Script | What It Does |
|--------|-------------|
| `hardenSysctl.sh` | Kernel hardening: ASLR, SYN cookies, ICMP protection, etc. |
| `patchPrivEsc.sh` | Patches pwnkit (CVE-2021-4034) and CVE-2023-32233 |
| `configureAppArmor.sh` | Installs AppArmor and enforces profiles |
| `removeUnusedPackages.sh` | Removes netcat, gcc, make, cmake, telnet |
| `bulkDisableServices.sh` | Disables xinetd, rsh, rlogin, tftp, and other risky services |
| `cronControl.sh` | Wipes all cron jobs (useful after compromise) |

### Auditing and Reconnaissance

| Script | What It Does |
|--------|-------------|
| `auditServices.sh` | Lists running/enabled services; flags suspicious ones |
| `auditStartupTasks.sh` | Checks cron, init.d, rc.local, shell profiles for persistence |
| `sudoCheck.sh` | Lists every user with sudo access |
| `groupMemberships.sh` | Audits who is in sudo/wheel/adm groups |
| `permissionAudit.sh` | Checks permissions on /etc/passwd, /etc/shadow, etc. |
| `checkPermissions.sh` | Verifies and fixes general file permissions |
| `checkPackageIntegrity.sh` | Verifies package checksums (detect file tampering) |
| `searchSSN.sh` | Scans /home for Social Security Numbers (PII check) |

### Maintenance

| Script | What It Does |
|--------|-------------|
| `updatePackages.sh` | Updates all packages (apt/dnf/yum/zypper/pacman/apk) |

### Windows / Active Directory

| Tool | What It Does |
|------|-------------|
| `gpos/` | 28 GPO backups covering security baselines, AppLocker, Defender, firewall, domain policy |
| `restart.ps1` | Remotely reboot multiple Windows machines via PowerShell |

---

## Detailed Documentation

- [Linux Hardening Suite](Linux/README.md)
- [Critical First-Response Tutorial (Beginner)](Linux/docs/TUTORIAL-critical-first-response.md)
- [Linux Module Reference](Linux/docs/MODULE-REFERENCE.md)
- [Windows GPO Guide](Windows/README.md)
