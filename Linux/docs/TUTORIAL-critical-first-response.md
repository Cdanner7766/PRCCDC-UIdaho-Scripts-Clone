# Tutorial: Critical First Response on Linux

**Target audience:** Beginners — assumes basic Linux terminal familiarity but explains every step.

This tutorial walks through the four most important scripts to run **within the first 15 minutes** of a CCDC round on a Linux machine.

---

## Before You Start

### What you need
- SSH access to the target machine (or console access)
- Your own account on the machine
- `sudo` privileges

### Golden rule
**Never close your only SSH terminal while making SSH or firewall changes.** Open a second terminal and test connectivity before closing anything.

---

## Step 0: Get the Scripts onto the Machine

```bash
# Option A: Clone from GitHub
git clone <repo-url>
cd PRCCDC-UIdaho-Scripts-Clone/Linux

# Option B: Copy from USB/shared drive (if no internet)
# Just scp the whole Linux/ folder to the machine

# Make all scripts executable
chmod +x modules/*.sh main.sh
```

---

## Step 1: Lock All Accounts — `lockAccounts.sh`

### Why do this first?
The moment you connect to a machine in CCDC, attackers may already have credentials for other accounts. Locking every account except yours prevents them from logging in while you work.

### What the script does
1. Creates a **new admin account** (you name it during the script)
2. Locks every other real user account using:
   - `passwd -l` — locks the password
   - `chage -E 0` — expires the account, which also blocks SSH key logins

### How to run it
```bash
sudo bash modules/lockAccounts.sh
```

### What to expect
```
=== Locking All Accounts Except Current User and a New Admin Account ===
[+] Current user: alice
----------------------------------------
Enter the username for a NEW admin account to create: ccdc_admin
[+] Creating user: ccdc_admin
New password:           <-- type a strong password
Retype new password:    <-- confirm it
[+] Admin privileges granted to 'ccdc_admin'

[+] Locking all accounts except:
    - alice
    - ccdc_admin
    - root
----------------------------------------
[-] Locking account: bob
[-] Locking account: charlie
[-] Locking account: mallory

[✓] All other user accounts have been locked.
```

### After running
- Only **you**, **root**, and the **new admin account** can log in
- To unlock an account later (e.g., for a service account you need):
  ```bash
  sudo passwd -u <username>       # re-enable the password
  sudo chage -E -1 <username>     # remove expiry
  ```

### Common issues
- **"No username entered. Exiting."** — You pressed Enter without typing a name. Run again.
- **Script skips a user** — That user has a non-interactive shell (`/sbin/nologin`, `/bin/false`). That is correct; those are system accounts that do not need locking.

---

## Step 2: Harden SSH — `sshConfig.sh`

### Why do this early?
SSH is your lifeline to the machine. Hardening it:
- Prevents brute-force logins
- Cuts off root login over SSH
- Forces stronger encryption

### What the script changes
| Setting | Old value (typical) | New value |
|---------|-------------------|-----------|
| `PermitRootLogin` | yes | no |
| `MaxAuthTries` | 6 | 3 |
| `MaxSessions` | 10 | 3 |
| `LoginGraceTime` | 120 | 30 seconds |
| `X11Forwarding` | yes | no |
| `AllowTcpForwarding` | yes | no |
| `Ciphers` | many weak ones included | only strong ones |
| `LogLevel` | INFO | VERBOSE |

The script also:
- **Backs up** your original config to `/etc/ssh/sshd_config.backup.<timestamp>`
- **Validates** the new config with `sshd -t` before restarting
- **Fixes** `.ssh/` directory permissions for all users

### How to run it

> **BEFORE YOU RUN:** Open a second SSH terminal to the same machine. Keep it open until the script finishes.

```bash
sudo bash modules/sshConfig.sh
```

### What to expect
```
=== SSH Server Hardening ===
Host: myserver
----------------------------------------
[+] Backing up current SSH configuration...
    Backup saved to: /etc/ssh/sshd_config.backup.2025-03-05_14-22-01

[+] Applying SSH hardening configuration...
    [✓] Root login disabled

[+] Validating SSH configuration...
    [✓] Configuration syntax is valid

[+] Restarting SSH service...
    [✓] SSH service restarted (systemd)

...

[✓] SSH Hardening Complete!

IMPORTANT REMINDERS:
  1. TEST your SSH connection in a NEW terminal before closing this one!
  2. If locked out, restore with: sudo cp /etc/ssh/sshd_config.backup.2025-03-05_14-22-01 /etc/ssh/sshd_config
```

### Test your connection
In your **second terminal**, try connecting again:
```bash
ssh youruser@machine-ip
```
If it works, you are good. If it fails, use your first terminal to restore:
```bash
sudo cp /etc/ssh/sshd_config.backup.<timestamp> /etc/ssh/sshd_config
sudo systemctl restart sshd
```

### Known limitation
The script leaves `PubkeyAuthentication no` — this disables SSH key authentication entirely and requires passwords. If your team relies on SSH keys, you will need to re-enable this manually:
```bash
sudo sed -i 's/^PubkeyAuthentication no/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl reload sshd
```

---

## Step 3: Apply the Firewall — `firewallRules.sh`

### Why apply a firewall?
By default, most Linux systems have an open firewall. This means attackers can reach any port. This script locks down outbound and inbound traffic to only what is needed.

### How iptables works (quick explainer)

`iptables` is Linux's built-in firewall. Traffic goes through three "chains":
- **INPUT** — traffic coming INTO the machine
- **OUTPUT** — traffic leaving the machine
- **FORWARD** — traffic passing through (used for routing; we block this)

The **policy** for each chain decides what happens to traffic that does not match any rule. This script sets the policy to **DROP** (block everything by default).

### What the script allows
| Direction | Type | Port | Why |
|-----------|------|------|-----|
| INPUT | TCP | 22 | SSH (your access) |
| INPUT | Any | — | Established connections (replies to your requests) |
| INPUT | Any | lo | Loopback (internal machine communication) |
| OUTPUT | TCP | 22 | SSH established replies |
| OUTPUT | UDP/TCP | 53 | DNS (so the machine can look up hostnames) |
| OUTPUT | TCP | 80 | HTTP (package updates, web) |
| OUTPUT | TCP | 443 | HTTPS |
| OUTPUT | ICMP | — | Ping outbound |
| OUTPUT | Any | — | Established connections |

### What the script blocks
- All inbound traffic except SSH and established connections
- All outbound traffic except the services listed above

### How to run it
```bash
sudo bash modules/firewallRules.sh
```

There is no interactive prompt — it applies immediately.

### Verify the rules are active
```bash
sudo iptables -L -v -n
```
You should see `DROP` policies for INPUT, FORWARD, OUTPUT, and your allow rules listed.

### If a service breaks

If the machine runs a service that suddenly stopped working (web app, database, custom scoring service), you need to add a rule to allow it.

**Example: Allow inbound traffic on port 8080 (web app)**
```bash
sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
```

**Example: Allow outbound on port 5432 (PostgreSQL talking to another machine)**
```bash
sudo iptables -I OUTPUT -p tcp --dport 5432 -j ACCEPT
```

**Emergency: Remove all firewall rules**
```bash
sudo iptables -F
sudo iptables -P INPUT ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
```

> **Note:** iptables rules are **not persistent by default** — they are lost on reboot.
> To save them on Debian/Ubuntu:
> ```bash
> sudo apt install iptables-persistent
> sudo netfilter-persistent save
> ```
> On RHEL/CentOS:
> ```bash
> sudo service iptables save
> ```

---

## Step 4: Rotate Passwords — `rotatePasswords.sh`

### Why rotate passwords?
If attackers already have the current passwords (common in CCDC — default creds, credential dumps, etc.), changing them immediately cuts that access.

### What the script does
Finds every user with a real shell (bash, sh, zsh) and UID >= 1000, plus root. For each one, asks you whether to change the password.

### How to run it
```bash
sudo bash modules/rotatePasswords.sh
```

### What to expect
```
[INFO] Starting password rotation process
[INFO] ==================================
[INFO] Found 4 user(s) to process

[User 1/4] Processing user: root
Do you want to change password for 'root'? (y/n): y
[INFO] Changing password for root...
New password:
Retype new password:
passwd: password updated successfully
[INFO] Password changed successfully for root

[User 2/4] Processing user: alice
Do you want to change password for 'alice'? (y/n): y
...
```

### Tips
- Press `n` to skip users you do not want to change (e.g., service accounts you already locked)
- Use **long, random passwords** — at least 16 characters
- Consider a password manager or a shared password doc for your team

---

## Summary Checklist

After running all four scripts, verify:

- [ ] `sudo bash modules/lockAccounts.sh` — ran successfully, accounts locked
- [ ] SSH still works in a second terminal after `sshConfig.sh`
- [ ] `sudo iptables -L` shows DROP policies after `firewallRules.sh`
- [ ] All key user passwords changed with `rotatePasswords.sh`
- [ ] You know the backup location for sshd_config (shown in script output)
- [ ] You know how to add iptables rules if a scored service breaks

---

## Next Steps

After completing the critical first response, move on to:

1. `auditServices.sh` — understand what is running
2. `sudoCheck.sh` — verify who has sudo
3. `hardenSysctl.sh` — kernel hardening
4. `patchPrivEsc.sh` — patch local privilege escalation
5. `removeUnusedPackages.sh` — remove attacker tools

See [MODULE-REFERENCE.md](MODULE-REFERENCE.md) for all modules.
