# Windows CCDC Tools — Overview & Walkthrough

## The Big Picture

The Windows folder contains two layers of defense:

1. **PowerShell scripts** — run interactively on machines to harden them
2. **GPO backups** — Group Policy Objects you restore to a Domain Controller to push settings to all machines at once

The domain controller (DC) is the most critical machine. Whoever controls the DC controls the domain.

---

## CCDC Quick Start Checklist

```
[ ] 1. On the DC: run dchardening.ps1 (as Domain Admin)
[ ] 2. On the DC: run groupManagement.ps1 (if not called by dchardening)
[ ] 3. On the DC: restore and link GPOs via PowerShell loop
[ ] 4. On all member servers: run Local-Hardening.ps1
[ ] 5. On web servers: run ww-hardening.ps1
[ ] 6. On the DC: run Reset-KrbTgt (mode 6) to invalidate old Kerberos tickets
[ ] 7. Force GPO update: gpupdate /force on all machines (or use restart.ps1)
```

---

## Scripts

### 1. `dchardening.ps1` — Domain Controller Hardening
> **Run first on the DC.**

**What it does:**

- **Backups first:** Exports firewall rules, DNS zones, and group memberships before changing anything
- **Imports helper scripts:** Pulls in `Local-Hardening.ps1`, `ww-hardening.ps1`, `advancedAuditing.ps1`, and others
- **AD lockdown:** Calls `groupManagement.ps1` to strip unauthorized users from privileged groups
- **Firewall hardening:** Blocks dangerous ports, configures Windows Firewall using `ports.json`
- **Service hardening:** Disables unnecessary services (telnet, print spooler, etc.)
- **GPO creation:** Creates and links a hardening GPO to all machines
- **Kerberos attack mitigations:** Patches ASREPRoasting and Kerberoasting vulnerabilities
- **Splunk/Sysmon:** Hooks up centralized logging
- **Patching:** Downloads and applies the EternalBlue (MS17-010) patch

**Requirements:** Must be run as **Domain Admin** on a DC.

**Gotchas:**
- Calls other scripts — keep the full repo present, or it falls back to GitHub
- Some sections are interactive and pause for input
- The backup steps at the top are critical — do not skip them

---

### 2. `Local-Hardening.ps1` — General Windows Host Hardening
> **Run on all non-DC machines.**

**What it does:**

- Disables NetBIOS and IPv6 (reduces attack surface for LLMNR/NBT-NS poisoning)
- Hardens RDP settings
- Configures Windows Firewall using `ports.json` to define allowed ports
- Disables unnecessary Windows services
- Enables UAC
- Installs patches (reads URLs from `patchURLs.json`)
- Enables advanced auditing (calls `advancedAuditing.ps1`)
- Configures Splunk/Sysmon for log forwarding
- Manages local user accounts (removes/disables competition-irrelevant accounts)

**Requirements:** Run as **local Administrator**.

**Gotchas:**
- Needs `ports.json`, `advancedAuditing.ps1`, and `patchURLs.json` — looks in the local directory first, then SYSVOL, then GitHub
- Firewall hardening **blocks** ports not listed in `ports.json` — verify scored services are listed **before** running

---

### 3. `ww-hardening.ps1` — Web Server Hardening + System Enumeration
> **Run on IIS/web machines. The enumeration section is useful on any machine.**

**IIS Hardening:**

- Sets all application pools to run with minimal privileges (`ApplicationPoolIdentity`)
- Disables directory browsing on all IIS sites
- Disables anonymous authentication on all sites
- Deletes custom error pages (prevents information leakage)

**System Enumeration (runs on any machine):**

- Prints OS version, domain membership, whether it's a DC
- Shows firewall status (Domain, Private, Public profiles)
- Lists Windows Defender status and any exclusions (exclusions are red flags)
- Shows active network adapters and IP addresses
- Lists non-default SMB shares
- Shows installed software
- Checks ~25+ common services/ports (IIS, FTP, RDP, SQL, MySQL, etc.)

**Optional monitors:**

| Flag | Behavior |
|------|----------|
| `-RunProcessMonitor` | Alerts on every new process; lets you kill or allow it |
| `-RunServiceMonitor` | Polls every second; alerts when a new service starts |

**Requirements:** Run as **Administrator**. IIS module must be installed for the hardening section.

**Gotchas:**
- Process/service monitors run in infinite loops — only use them with a dedicated terminal
- IIS section will error on non-IIS machines (non-fatal)

---

### 4. `groupManagement.ps1` — AD Privileged Group Cleanup
> **Called by `dchardening.ps1`, or run standalone on the DC.**

**What it does:**

Iterates through ~20 high-privilege AD groups and removes anyone who shouldn't be there:

| Group | Kept Members |
|-------|-------------|
| Administrators | `Administrator`, `Domain Admins`, `Enterprise Admins` |
| Domain Admins | `Administrator` only |
| Enterprise Admins / Schema Admins | `Administrator` only |
| DNS Admins | Cleared |
| Remote Desktop Users | Cleared |
| Pre-Windows 2000 Compatible Access | Removes `ANONYMOUS LOGON` and `Everyone` |
| All SMB shares | Removes `ANONYMOUS LOGON` |

Also clears the `dSHeuristics` AD attribute to block anonymous LDAP binds.

**Requirements:** `ActiveDirectory` PowerShell module, run on a DC as Domain Admin.

**Gotchas:**
- Red teams frequently pre-plant accounts in privileged groups — run this early
- Some groups may not exist in all environments; errors are caught and reported as warnings

---

### 5. `advancedAuditing.ps1` — Enable Security Event Logging
> **Called automatically by `Local-Hardening.ps1` and `dchardening.ps1`. Can also be run standalone.**

**What it does:**

Enables detailed Windows Security Event Log auditing:

- Logon / logoff events (success + failure)
- Account logon and account management events
- Directory Service Access (AD object access)
- Object access (files, registry)
- Process creation and termination tracking
- Policy change auditing
- Privilege use auditing
- File system audit rules (creation events on all drives)
- Firewall rule change auditing
- Checks for .NET Framework 4.7+ and installs if missing

**Gotchas:**
- File system audit rules set "Everyone/CreateFiles/Success" on all local drives — may generate high log volume

---

### 6. `Reset-KrbTgt-Password-For-RWDCs-And-RODCs.ps1` — Kerberos Ticket Invalidation
> **Run on the DC after initial lockdown to invalidate stolen/pre-existing Kerberos tickets.**

**Why this matters:** Any attacker with a valid Kerberos ticket (e.g., a Golden Ticket) can continue operating even after you change AD passwords. Resetting the `krbtgt` account password invalidates all outstanding tickets.

**Important:** Must be reset **twice** to fully invalidate all tickets (Kerberos tickets have up to a 10-hour lifetime).

**Modes:**

| Mode | What it does |
|------|-------------|
| 1 | Informational only — shows DC status, no changes |
| 2 | Canary replication test — no changes |
| 3 | Check TEST krbtgt status |
| 4 | Reset TEST krbtgt (safe to practice with) |
| 5 | Check PROD krbtgt status |
| **6** | **Reset PROD krbtgt** — the real action |
| 8 | Create TEST krbtgt accounts |
| 9 | Delete TEST krbtgt accounts |

**Recommended CCDC workflow:**
1. Run **Mode 1** — verify all DCs are reachable
2. Run **Mode 6** — reset the production krbtgt
3. If time allows, run **Mode 6 again** after ~10 minutes

**Requirements:** Run on a DC as Domain Admin. This is an interactive script.

---

### 7. `restart.ps1` — Remote Bulk Reboot
> **Utility — use after GPO pushes or hardening to force clean reboots across machines.**

**Current state:** The `$computers` variable is not defined in the script. Define it before running:

```powershell
$computers = @("SERVER01", "WORKSTATION01", "DC02")
# Then dot-source and run the script
. .\restart.ps1
```

**Requirements:** WinRM must be enabled on target machines (`Enable-PSRemoting -Force`) and you need admin rights on each.

---

### 8. `startup.ps1` — Disable NetBIOS and IPv6
> **Called by `Local-Hardening.ps1`. Can be run standalone on any machine.**

**What it does:**

1. Disables NetBIOS over TCP/IP on all network adapters — blocks LLMNR/NBT-NS poisoning attacks
2. Fully disables IPv6 via registry (`DisabledComponents = 0xFF`)

---

## Data/Config Files

### `patchURLs.json`

Maps Windows OS versions to the correct MS17-010 (EternalBlue) patch download URL. Used by `Local-Hardening.ps1` to automatically download and apply the right patch. Covers Vista through Server 2012 R2. Windows 10/2016/2019+ are not listed (not vulnerable to the original EternalBlue).

### `ports.json`

Defines what ports should be allowed through Windows Firewall for various machine types. Used by `Local-Hardening.ps1` when building firewall rules. **Review and update this before running firewall hardening** to ensure scored services aren't blocked.

---

## GPO Folder (`gpos/`) — 28 Group Policy Object Backups

Pre-built security policies that can be restored to your AD domain and pushed to all machines instantly.

### How to restore and apply GPOs

```powershell
Import-Module GroupPolicy

$BackupPath = "C:\path\to\Windows\gpos"
$Domain = "DC=yourdomain,DC=local"  # Update to match your domain

# List available GPO backups
Get-ChildItem $BackupPath -Directory | ForEach-Object {
    Get-GPOBackupInfo -Path $_.FullName -ErrorAction SilentlyContinue
}

# Restore a single GPO by name (example: "baseline")
$GpoName = "baseline"
New-GPO -Name $GpoName -ErrorAction SilentlyContinue
Import-GPO -BackupGpoName $GpoName -Path $BackupPath -TargetName $GpoName
New-GPLink -Name $GpoName -Target $Domain -ErrorAction SilentlyContinue

# Force all machines to apply immediately
gpupdate /force
```

### Recommended GPO application order for CCDC

1. `laps` — randomize local admin passwords
2. `baseline` — core security settings
3. `basefirewall` — enable Windows Firewall on all machines
4. `basedefender` — enable and configure Windows Defender
5. `applocker` — restrict what executables can run
6. `dc_gpo` + `dc_firewall` — additional DC-specific lockdown
7. Role-based firewall GPOs (web, database, mail servers)
8. Credential Guard / VBS — if hardware supports it
