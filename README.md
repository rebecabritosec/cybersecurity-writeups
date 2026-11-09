# Cybersecurity | Penetration Testing Portfolio

**Hands-on security assessments, vulnerability analysis, and technical write-ups.**

This repository documents my practical work in cybersecurity, with a focus on **penetration testing, web application security, and privilege escalation**.

The projects are based primarily on [TryHackMe](https://tryhackme.com/) laboratories and Capture The Flag (CTF) challenges. Each write-up documents the assessment process, from initial reconnaissance and enumeration to vulnerability identification, exploitation, and security impact analysis.

The goal is to build and demonstrate practical security assessment skills through structured technical investigations, reproducible findings, and clear documentation.

## Featured Security Assessments

### Web Application Security

#### [Recruit — Web Application Penetration Testing](http://tryhackme/recruit/README.md)

Security assessment of a PHP-based recruitment application, focusing on information disclosure, database security, and access control weaknesses.

**Key findings and techniques:**

- Web content discovery and endpoint enumeration using Gobuster.

- Local File Disclosure and exposure of sensitive application configuration.

- PHP source code analysis and credential discovery.

- UNION-based SQL Injection and database enumeration through `information\\\\\\\_schema`.

- Credential extraction and administrative access.

**Outcome:** Identified and chained application vulnerabilities to obtain administrative access.

[Read the technical write-up](http://tryhackme/recruit/README.md)

#### [Support — Web Application Penetration Testing](http://tryhackme/support/README.md)

Assessment of a corporate support application, examining authentication mechanisms, authorization controls, information exposure, and server-side command execution.

**Key findings and techniques:**

- Directory and file enumeration using Feroxbuster.

- Directory Listing and Local File Disclosure.

- Authentication testing and password auditing with Hydra and John the Ripper.

- Client-controlled authorization bypass through cookie manipulation.

- Insecure Direct Object Reference (IDOR) in a user API.

- PHP source code analysis and OS Command Injection through `shell\\\\\\\_exec()`.

**Outcome:** Chained multiple weaknesses to obtain administrative access and demonstrate command execution.

[Read the technical write-up](http://tryhackme/support/README.md)

### Linux Privilege Escalation

#### [Linux — Recon Pipeline](http://tryhackme/privilege-escalation/Linux/README.md)

Multi-stage privilege escalation assessment involving insecure Linux automation, file permissions, service execution, and sudo configurations.

**Attack path:**

`Anonymous FTP → recon\\\\\\\_user → dev\\\\\\\_user → monitor\\\\\\\_user → ops\\\\\\\_user → root`

**Key findings and techniques:**

- Anonymous FTP uploads and automated script execution.

- Exploitation of writable backup scripts and excessive group permissions.

- PATH Hijacking through a misconfigured systemd monitoring service.

- Abuse of writable deployment dependencies executed through sudo.

- Root privilege escalation through a `less` shell escape documented in GTFOBins.

**Outcome:** Successfully escalated from unauthenticated FTP access to a root shell by chaining insecure automation and privilege boundaries.

[Read the technical write-up](http://tryhackme/privilege-escalation/Linux/README.md)

### Windows Privilege Escalation

#### [Windows — Service and Access Control Misconfigurations](http://tryhackme/privilege-escalation/Windows/README.md)

Windows privilege escalation assessment focused on exposed credentials, insecure service configurations, and insufficient file permission controls.

**Attack path:**

`Guest SMB → thmuser → notadmin → svcadmin → Administrative Access`

**Key findings and techniques:**

- SMB share enumeration and credential disclosure.

- Interactive access through RDP.

- Plaintext AutoLogon credential discovery using WinPEAS.

- Windows service permission and NTFS ACL analysis.

- Service Binary Hijacking to execute code under another account.

- Exploitation of a writable maintenance script to obtain administrative privileges.

**Outcome:** Progressively escalated through multiple Windows security contexts and obtained local administrative access.

[Read the technical write-up](http://tryhackme/privilege-escalation/Windows/README.md)

## Technical Competencies

The documented assessments demonstrate practical experience with the following security concepts.

| **Domain** | **Practical Experience** |
| :-: | :-: |
| **Reconnaissance & Enumeration** | Service discovery, port scanning, directory enumeration, endpoint analysis |
| **Web Application Security** | SQL Injection, IDOR, Local File Disclosure, authentication and authorization weaknesses, OS Command Injection |
| **Credential Security** | Credential exposure analysis, authentication testing, password auditing |
| **Linux Security** | File permissions, group membership analysis, automated tasks, PATH Hijacking, systemd, sudo misconfigurations |
| **Windows Security** | SMB enumeration, AutoLogon analysis, NTFS permissions, Windows services, Service Binary Hijacking |
| **Security Investigation** | Source code review, process inspection, exploit chain analysis, technical documentation |


## Tools & Technologies

Selected tools used throughout the documented laboratories and security exercises.

| **Category** | **Tools** |
| :-: | :-: |
| Reconnaissance & Discovery | Nmap, Gobuster, Feroxbuster |
| Web Application Testing | Burp Suite |
| Authentication & Password Auditing | Hydra, John the Ripper |
| Exploitation & Connectivity | Metasploit Framework, Netcat |
| Linux Enumeration | Linux CLI, pspy, LinPEAS |
| Windows Enumeration | WinPEAS, PowerShell, Windows CLI |
| Operating Systems & Scripting | Linux, Windows, Bash, PowerShell |


Tools are used according to the assessment objective, with emphasis on understanding the underlying vulnerability and validating the results rather than relying exclusively on automated findings.

## Assessment & Documentation Approach

The write-ups follow a structured approach to documenting security investigations:

1. **Reconnaissance:** Identify exposed services, application components, and potential entry points.

2. **Enumeration:** Investigate configurations, permissions, authentication mechanisms, and accessible resources.

3. **Vulnerability Analysis:** Determine the root cause of identified weaknesses and their potential security impact.

4. **Exploitation & Validation:** Validate findings within the authorized laboratory scope and document the resulting access or behavior.

5. **Technical Documentation:** Record relevant evidence, commands, exploitation chains, and security considerations.

The emphasis is on understanding **why a vulnerability exists, how it can be exploited, and what security boundary is affected**.

## Repository Structure

```
\\\`cybersecurity/\\\`    
    
\\\`│\\\`    
    
\\\`├── README.md\\\`    
    
\\\`│\\\`    
    
\\\`└── tryhackme/\\\`    
    
\\\`    │\\\`    
    
\\\`    ├── recruit/\\\`    
    
\\\`    │   └── README.md\\\`    
    
\\\`    │\\\`    
    
\\\`    ├── support/\\\`    
    
\\\`    │   └── README.md\\\`    
    
\\\`    │\\\`    
    
\\\`    └── privilege-escalation/\\\`    
    
\\\`        │\\\`    
    
\\\`        ├── Linux/\\\`    
    
\\\`        │   └── README.md\\\`    
    
\\\`        │\\\`    
    
\\\`        └── Windows/\\\`    
    
\\\`            └── README.md\\\`
```

Each challenge has its own technical write-up, allowing the repository to expand while keeping the main portfolio concise and easy to navigate.

## Current Learning Focus

My ongoing technical development focuses on strengthening practical penetration testing methodology, particularly in:

- Web application vulnerability discovery and manual validation.

- Authentication, authorization, and access control testing.

- Linux and Windows privilege escalation techniques.

- Network reconnaissance and service enumeration.

- Exploit chain analysis, evidence collection, and security reporting.

The objective is to continue developing a methodical approach to security assessments, connecting practical exploitation techniques with their underlying technical causes and defensive implications.

## Ethical & Authorized Testing

All exploitation activities documented in this repository were performed in **authorized educational environments**, primarily TryHackMe laboratories and CTF challenges.

The content is intended for educational purposes, technical skill development, and security research.

No unauthorized systems or third-party infrastructure are targeted as part of these documented exercises.

*This repository is maintained as a technical portfolio of hands-on cybersecurity learning, practical investigations, and documented security assessments.*

