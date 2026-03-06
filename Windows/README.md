# Windows / Active Directory Tools

This directory contains Group Policy Object (GPO) backups for the PRCCDC Active Directory domain (`prccdc.htb`) and a PowerShell remote reboot utility.

---

## Contents

```
Windows/
├── restart.ps1        # Remote bulk reboot for multiple Windows machines
└── gpos/              # 28 GPO backups — restore to harden an AD domain
    └── {GUID}/        # One folder per GPO, named by GUID
        ├── Backup.xml         # GPO metadata and name
        ├── bkupInfo.xml       # Backup timestamps
        ├── gpreport.xml       # Detailed settings report
        └── DomainSysvol/GPO/ # Policy files (registry.pol, security templates, etc.)
```

---

## GPO Inventory

The `gpos/` folder contains 28 backed-up Group Policy Objects. The table below maps each GUID folder to its GPO name.

| GPO Name | GUID | Purpose |
|----------|------|---------|
| Default Domain Policy | `{49F7F4EA-...}` | Base domain-wide settings |
| Default Domain Controllers Policy | `{08DADFC5-...}` | DC-specific baseline |
| baseline | `{107D1D1D-...}` | Custom baseline security policy |
| Good-GPO | `{04381F4C-...}` | Reference "good" security state |
| baseperms | `{F1BFF74D-...}` | Baseline file/registry permissions |
| basefirewall | `{F0E432E8-...}` | Windows Firewall baseline rules |
| basedefender | `{52B110AB-...}` | Windows Defender baseline |
| applocker | `{8E184663-...}` | AppLocker allow/deny rules |
| default_applocker | `{57C27367-...}` | Default AppLocker configuration |
| laps | `{AF073574-...}` | Local Administrator Password Solution |
| dc_gpo | `{3D313A74-...}` | Domain Controller hardening |
| dc_firewall | `{36ED4CBC-...}` | Domain Controller firewall rules |
| enable_rdp | `{056201C4-...}` | Enable Remote Desktop for domain machines |
| Workstations RDP Enable | `{79CCDE22-...}` | RDP enable policy for workstations |
| Web Servers Firewall Policy | `{B5B2BD0E-...}` | Firewall rules for web server role |
| Databases Firewall Policy | `{20A98617-...}` | Firewall rules for database server role |
| Mail Servers Firewall Policy | `{F11F1678-...}` | Firewall rules for mail server role |
| Domain GPO Update Interval | `{FA74DD20-...}` | Controls how often GPOs refresh |
| Domain Splomk | `{F2C8F452-...}` | Custom domain policy |
| MSFT Windows Server 2022 - Member Server | `{5EEC8804-...}` | Microsoft security baseline for member servers |
| MSFT Windows Server 2022 - Domain Controller | `{9085A343-...}` | Microsoft security baseline for DCs |
| MSFT Windows Server 2022 - Domain Security | `{82C5DFD2-...}` | Microsoft domain security settings |
| MSFT Windows Server 2022 - Defender Antivirus | `{79146F7B-...}` | Microsoft Defender configuration |
| MSFT Windows Server 2022 - Member Server Credential Guard | `{28E6CAA8-...}` | Credential Guard for member servers |
| MSFT Windows Server 2022 - DC Virtualization Based Security | `{FB1609CA-...}` | VBS/HVCI for domain controllers |
| MSFT Internet Explorer 11 - Computer | `{815FCABF-...}` | IE 11 computer settings |
| MSFT Internet Explorer 11 - User | `{4D378C73-...}` | IE 11 user settings |
| New Group Policy Object | `{A59031A2-...}` | Unnamed/placeholder GPO |

### GPO Categories

**Security Baselines** — Apply these first to establish a secure foundation:
- `baseline`
- `MSFT Windows Server 2022 - Member Server`
- `MSFT Windows Server 2022 - Domain Controller`
- `MSFT Windows Server 2022 - Domain Security`

**Access Control:**
- `applocker` / `default_applocker` — restrict what executables can run
- `laps` — randomize local administrator passwords across machines
- `baseperms` — file and registry permissions

**Firewall Policies** — Apply based on each machine's role:
- `basefirewall` — all machines
- `dc_firewall` — domain controllers only
- `Web Servers Firewall Policy` — IIS/web servers
- `Databases Firewall Policy` — SQL Server machines
- `Mail Servers Firewall Policy` — Exchange/mail servers

**Remote Access:**
- `enable_rdp` — enables RDP for domain machines
- `Workstations RDP Enable` — specifically for workstations

**Malware Protection:**
- `basedefender` / `MSFT Windows Server 2022 - Defender Antivirus`

**Advanced Protections:**
- `MSFT Windows Server 2022 - Member Server Credential Guard` — prevents credential theft
- `MSFT Windows Server 2022 - DC Virtualization Based Security` — HVCI/VBS

---

## How to Restore GPOs

### Method 1: Group Policy Management Console (GPMC) — GUI

1. Log into a **Domain Controller** as Domain Admin
2. Open **Group Policy Management** (run `gpmc.msc`)
3. In the left pane, expand your domain
4. Right-click **Group Policy Objects** → **Import Settings...**
5. When the wizard asks for a backup location, point it to the `gpos/` folder
6. Select the GPO you want to restore by name
7. Complete the wizard
8. **Link the GPO** to the appropriate OU or domain root (right-click the OU → "Link an Existing GPO")

### Method 2: PowerShell — Command Line

Run these commands on a Domain Controller as Domain Admin:

```powershell
# Import the GroupPolicy module
Import-Module GroupPolicy

# Define where your backup folder is
$BackupPath = "C:\path\to\PRCCDC-UIdaho-Scripts-Clone\Windows\gpos"

# Restore a single GPO by name (it must already exist in AD — create it first if needed)
# Step 1: Create a new empty GPO
New-GPO -Name "baseline"

# Step 2: Import settings from backup into it
Import-GPO -BackupGpoName "baseline" -Path $BackupPath -TargetName "baseline"

# Step 3: Link it to the domain root (or a specific OU)
New-GPLink -Name "baseline" -Target "DC=prccdc,DC=htb"
```

### Method 3: Restore All GPOs at Once (PowerShell Loop)

```powershell
Import-Module GroupPolicy

$BackupPath = "C:\path\to\PRCCDC-UIdaho-Scripts-Clone\Windows\gpos"

# List of GPOs to restore — edit as needed
$GPONames = @(
    "baseline",
    "baseperms",
    "basefirewall",
    "basedefender",
    "dc_gpo",
    "dc_firewall",
    "laps"
)

foreach ($GPOName in $GPONames) {
    try {
        # Create GPO if it doesn't exist
        if (-not (Get-GPO -Name $GPOName -ErrorAction SilentlyContinue)) {
            New-GPO -Name $GPOName | Out-Null
            Write-Host "[+] Created GPO: $GPOName"
        }
        # Import from backup
        Import-GPO -BackupGpoName $GPOName -Path $BackupPath -TargetName $GPOName
        Write-Host "[OK] Imported: $GPOName"
    } catch {
        Write-Warning "[FAIL] Could not import $GPOName`: $_"
    }
}
```

### Force GPO Update on All Machines

After linking GPOs, push them immediately without waiting for the default refresh:

```powershell
# Force update on a single remote machine
Invoke-GPUpdate -Computer "WORKSTATION01" -Force -RandomDelayInMinutes 0

# Force update on all computers in an OU
Get-ADComputer -Filter * -SearchBase "OU=Workstations,DC=prccdc,DC=htb" |
    ForEach-Object {
        Invoke-GPUpdate -Computer $_.Name -Force -RandomDelayInMinutes 0
    }

# Or just run on the local machine
gpupdate /force
```

---

## restart.ps1 — Remote Bulk Reboot

### What it does
Sends an immediate reboot command (`shutdown /r /t 0`) to a list of remote computers using PowerShell Remoting.

### Current state
The script is **incomplete** — it references a `$computers` variable that must be defined before the loop runs. You need to add the computer list before using it.

### How to use it

```powershell
# Add your computer list at the top of the script, then run it:

$computers = @(
    "WORKSTATION01",
    "WORKSTATION02",
    "SERVER01",
    "DC01"
)

# Then run the rest of the script, or paste both blocks into one PowerShell session
```

**Requirements:**
- WinRM (PowerShell Remoting) must be enabled on target machines
- You must have admin rights on the remote machines
- Run from a machine with network access to targets

**Enable WinRM on remote machines (if not already on):**
```powershell
# Run on each target, or push via GPO
Enable-PSRemoting -Force
```

---

## Recommended CCDC GPO Deployment Order

When you get access to a Windows domain in CCDC, apply GPOs in this order:

1. **`laps`** — Randomize local admin passwords immediately
2. **`baseline`** — Apply base security settings
3. **`basefirewall`** — Enable Windows Firewall with deny-by-default
4. **`basedefender`** / Defender Antivirus — Enable and configure Defender
5. **`applocker`** — Restrict software execution
6. **`dc_gpo`** + **`dc_firewall`** — Harden domain controllers
7. Role-specific firewall GPOs based on each machine's function
8. **Credential Guard** / **VBS** — After confirming hardware supports it

### Linking GPOs to the Right Level

| GPO | Link Target |
|-----|------------|
| `Default Domain Policy` | Domain root |
| `baseline`, `basefirewall`, `basedefender` | Domain root or top-level OU |
| `dc_gpo`, `dc_firewall` | Domain Controllers OU |
| `Web Servers Firewall Policy` | Web Servers OU |
| `Databases Firewall Policy` | Database Servers OU |
| `laps` | All computer OUs |
| `applocker` | Workstations OU (test carefully) |

---

## Useful AD/GPO Commands

```powershell
# List all GPOs in the domain
Get-GPO -All | Select-Object DisplayName, GpoStatus, CreationTime

# Check which GPOs are linked to a specific OU
Get-GPInheritance -Target "OU=Workstations,DC=prccdc,DC=htb"

# Back up all GPOs (useful to save your hardened state mid-competition)
$timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm"
Backup-GPO -All -Path "C:\GPO-Backup-$timestamp"

# Check GPO application status on a machine
gpresult /r

# Detailed GPO report for a machine
gpresult /h C:\gpo-report.html /f

# Check who has admin rights across domain
Get-ADGroupMember "Domain Admins" | Select-Object Name, SamAccountName

# List all users with admin rights
Get-ADGroupMember "Administrators" -Recursive | Select-Object Name

# Force password change on next login for a user
Set-ADUser -Identity <username> -ChangePasswordAtLogon $true

# Disable a user account immediately
Disable-ADAccount -Identity <username>

# See all enabled user accounts
Get-ADUser -Filter {Enabled -eq $true} | Select-Object Name, SamAccountName
```
