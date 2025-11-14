Security Hardening, Misconfiguration Detection, and Remediation

1. Project Overview

This project involves creating a secure user/group model on an Ubuntu server, applying least-privilege access controls, enabling auditing, and analyzing a deliberately misconfigured VM to identify and fix at least three security issues.

Tools used:

Ubuntu Lab VM (target system)

Kali/Attacker VM (optional testing)

sudo privileges on the lab VM

POSIX permissions, ACLs, sudoers, auditd



---

2. Baseline Security Policy

Team Roles

Role	Description	Required Privileges

admin	System administrator	Full sudo (password required), user mgmt, package mgmt
dev	Developer team	No sudo; write access to project folder
auditor	Security reviewer	Read-only access to logs & project folder; no sudo


Access Requirements

Resource	admin	dev	auditor	others

/srv/project	RWX	RWX (group)	R– (ACL)	–
/etc/sudoers	RW via sudo	–	R	–
Audit logs (/var/log/audit/audit.log)	RW	–	R	–


Sudo Policy

Only admin can run system management commands.

No one receives blanket NOPASSWD.

Developers receive zero sudo permissions.

Auditors get only read permissions (no sudo).



---

3. User & Group Setup

Create Groups

sudo groupadd admin
sudo groupadd dev
sudo groupadd auditor

Create Users

sudo useradd -m alice -G admin
sudo useradd -m bob -G dev
sudo useradd -m carol -G auditor
sudo passwd alice
sudo passwd bob
sudo passwd carol


---

4. Configure Least-Privilege Sudo Rules

Open sudoers safely:

sudo visudo

Add:

%admin ALL=(ALL) ALL
%dev   ALL=(ALL) !ALL
%auditor ALL=(ALL) !ALL

✔ No passwordless sudo
✔ No wildcard command permissions
✔ Only admin can run privileged commands


---

5. Secure Shared Project Directory

Create the directory

sudo mkdir -p /srv/project
sudo chown :dev /srv/project
sudo chmod 2775 /srv/project     # g+s for group inheritance

ACLs for auditor read-only access

sudo setfacl -m g:auditor:rx /srv/project

Verify:

getfacl /srv/project


---

6. Enable Auditing With auditd

Install:

sudo apt install auditd audispd-plugins -y

Add audit rules:

sudo auditctl -w /etc/sudoers -p wa -k sudoers_changes
sudo auditctl -w /etc/passwd -p wa -k passwd_changes

Persist rules:

sudo nano /etc/audit/rules.d/user_mgmt.rules

Add:

-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/passwd -p wa -k passwd_changes

Restart:

sudo systemctl restart auditd


---

7. Vulnerable Snapshot – Misconfigurations Found

Misconfiguration #1: World-Writable /etc/cron.d

drwxrwxrwx  root root /etc/cron.d

Impact: Any user can add cron jobs → privilege escalation.

Fix:

sudo chmod 755 /etc/cron.d


---

Misconfiguration #2: Sudoers Has NOPASSWD for ALL

In /etc/sudoers:

%dev ALL=(ALL) NOPASSWD: ALL

Impact: Developers get full root access without authentication.

Fix:

sudo visudo

Change to:

%dev ALL=(ALL) !ALL


---

Misconfiguration #3: Sensitive File World-Readable

Example:

-rw-r--r--  1 root root /etc/shadow.backup

Impact: Password hashes are exposed.

Fix:

sudo chmod 600 /etc/shadow.backup
sudo chown root:root /etc/shadow.backup


---

8. Before/After Evidence

Example

Before:

-rw-r--r-- 1 root root /etc/shadow.backup

After:

-rw------- 1 root root /etc/shadow.backup

Sudoers Change Logged:

sudo ausearch -k sudoers_changes

ACL Verification:

# file: /srv/project
# owner: root
# group: dev
group:auditor:rx


---

9. Remediation Checklist

User/Group Management

[x] Remove unused accounts

[x] Enforce strong passwords

[x] Assign users to least-privilege groups

[x] Disable password login for service accounts


File Permissions

[x] Restrict sensitive files (600 or stricter)

[x] Remove world-writable directories except /tmp

[x] Use ACLs only when required


Sudo Hardening

[x] No NOPASSWD rules

[x] No wildcard commands (*)

[x] Only admin group has sudo


Auditing

[x] Monitor /etc/passwd, /etc/shadow, /etc/sudoers

[x] Forward logs to remote system (optional)

[x] Archive logs monthly



---

10. Ongoing Security Policy

Quarterly review of user accounts and group memberships

Regular permission auditing using:

sudo find / -perm -0002 -type d 2>/dev/null

Routine review of sudo logs:

sudo ausearch -k sudoers_changes

Developer team receives least privilege; no escalation path



---

11. Repository Structure Example

/project-secure-lab
│
├── README.md
├── evidence/
│   ├── ls_before.png
│   ├── ls_after.png
│   ├── getfacl_output.txt
│   └── audit_logs.txt
├── policy/
│   ├── access_policy.md
│   ├── sudo_policy.md
│   └── remediation_checklist.md
└── scripts/
    ├── create_users.sh
    ├── configure_acls.sh
    └── audit_rules.sh
