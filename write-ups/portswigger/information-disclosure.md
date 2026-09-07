---
layout: default
title: "PortSwigger — Information Disclosure"
permalink: /write-ups/portswigger/information-disclosure/
---

# PortSwigger — Information Disclosure

## Overview

This page documents my notes and write-ups from the **Information Disclosure** module of PortSwigger Web Security Academy.

Information disclosure vulnerabilities occur when an application unintentionally exposes sensitive information, technical details, internal paths, credentials, source code, debugging data, or implementation details.

The goal of this module was to practice identifying useful information leaks and understanding how small disclosures can support further attacks.

---

## Labs Covered and Main Technique

| 01 | Information disclosure in error messages | Triggering verbose errors
| 02 | Information disclosure on debug page | Finding exposed debug data
| 03 | Source code disclosure via backup files | Discovering backup source files
| 04 | Authentication bypass via information disclosure | Abusing leaked internal header
| 05 | Information disclosure in version control history | Recovering secrets from Git history

---

# Lab 01 — Information disclosure in error messages

## Objective

Identify sensitive information leaked through verbose application error messages.

## Methodology

I started by browsing the application normally and observing the requests in Burp Proxy. One product page contained a parameter that controlled which product was loaded.

I sent the request to Burp Repeater and changed the expected numeric value to an unexpected string value ('a'). This caused the application to return a verbose error response.

## Finding

The application returned a detailed exception instead of a generic error message. The response exposed internal technology information, including the framework and version used by the application.

## Impact

Verbose error messages can reveal details such as frameworks, versions, internal paths, database behavior, and stack traces. Even when this information is not directly exploitable by itself, it can help an attacker refine future tests or search for known vulnerabilities affecting the disclosed technology.

## Lesson Learned

Unexpected input can force applications to reveal internal behavior. During web testing, error messages should be reviewed carefully because they may disclose useful technical details even when the page itself appears normal.

---

# Lab 02 — Information disclosure on debug page

## Objective

Find sensitive information exposed through a debugging page.

## Methodology

I inspected the application and looked for information that was not visible directly on the rendered page. By reviewing the page source and the requests captured in Burp, I identified a developer comment that referenced a debug endpoint.

The hidden debug path was:

```text
/cgi-bin/phpinfo.php
```

I accessed this endpoint and reviewed the debugging output. The page exposed application and environment information that should not have been available in production.

## Finding

The debug page disclosed sensitive configuration data, including a `SECRET_KEY` environment variable.

## Impact

Debug pages can leak environment variables, server configuration, internal paths, loaded modules, software versions, and application secrets. This information may help an attacker understand the application and potentially chain the disclosure with other vulnerabilities.

## Lesson Learned

Developer comments and debug endpoints are common sources of information disclosure. Content that is hidden from the visual interface can still be present in the HTML source or accessible through direct paths.

---

# Lab 03 — Source code disclosure via backup files

## Objective

Recover sensitive information from a source code backup file exposed by the application.

## Methodology

I started by checking common discovery locations that may reveal hidden paths. The `robots.txt` file disclosed the existence of a backup directory.

Relevant paths:

```text
/robots.txt
/backup/
/backup/ProductTemplate.java.bak
```

After accessing the backup directory, I found a `.bak` file containing a backup copy of a Java source code file. I opened the file directly and reviewed the source code for sensitive values.

## Finding

The backup file exposed Java source code containing a hardcoded PostgreSQL database password.

## Impact

Source code disclosure can be serious because it may reveal credentials, API keys, database connection strings, internal logic, hidden functionality, or implementation details that would otherwise be unavailable to an attacker.

In this lab, the sensitive information was not exposed through the normal application interface, but it was still accessible because a backup file had been left inside the web root.

## Lesson Learned

`robots.txt` can reveal useful hidden paths, but it is not an access control mechanism. Backup files should not be stored in publicly accessible directories, and secrets should not be hardcoded in source code.

---

# Lab 04 — Authentication bypass via information disclosure

## Objective

Use disclosed information to bypass access control and access the admin interface.

## Methodology

I first accessed the admin endpoint and observed that the application restricted access. The response indicated that the admin area was only available to administrators or to requests coming from a local IP address.

I then tested the HTTP `TRACE` method against the `/admin` endpoint. The server reflected the request it received and disclosed a custom authorization header that had been added to the request.

## Key Request

```http
TRACE /admin HTTP/1.1
Host: LAB-ID.web-security-academy.net
X-Test: trace-check
```

The reflected response revealed the following header:

```http
X-Custom-IP-Authorization: <client-ip>
```

This indicated that the application or front-end infrastructure was using this header to make IP-based authorization decisions.

After identifying the header, I sent a normal request to `/admin` while manually setting the header value to the localhost address:

```http
GET /admin HTTP/1.1
Host: LAB-ID.web-security-academy.net
X-Custom-IP-Authorization: 127.0.0.1
```

Equivalent `curl` example:

```bash
curl -i https://LAB-ID.web-security-academy.net/admin \
-H "X-Custom-IP-Authorization: 127.0.0.1"
```

Once the admin interface became accessible, I completed the required action in the lab.

## Finding

The application trusted a client-controllable HTTP header to determine whether a request came from a local IP address.

## Impact

Authorization decisions should not be based on headers that the user can control. If an application trusts such headers without ensuring they were set by a trusted reverse proxy, an attacker may be able to spoof the header and bypass access control.

## Lesson Learned

The `TRACE` method can disclose request headers that are added by intermediate systems. When these headers are used for authorization, a simple information leak can become an authentication or authorization bypass.

---

# Lab 05 — Information disclosure in version control history

## Objective

Recover an administrator password from an exposed Git version control history.

## Methodology

I discovered that the application exposed its Git metadata through the web server. Accessing the following path showed that the `.git` directory was publicly available:

```text
/.git/
```

The exposed directory contained typical Git files and folders, such as:

```text
.git/HEAD
.git/config
.git/objects/
.git/refs/
.git/logs/
.git/index
.git/COMMIT_EDITMSG
```

I downloaded the exposed Git metadata and reconstructed the repository locally.

## Key Commands

```bash
wget -r -np -nH --reject "index.html*" https://LAB-ID.web-security-academy.net/.git/
```

```bash
git --git-dir=.git --work-tree=. checkout -f
```

```bash
git log --oneline --all
```

The commit history contained a message indicating that the administrator password had been removed from a configuration file:

```text
Remove admin password from config
```

I inspected the relevant commit to see what had changed:

```bash
git show COMMIT_HASH
```

The diff showed that the hardcoded administrator password had been replaced with an environment variable. However, the removed password was still visible in the Git history.

## Finding

The current version of the application no longer contained the administrator password, but the password was still recoverable from a previous commit in the exposed Git history.

## Impact

Exposing a `.git` directory can leak source code, commit history, configuration files, internal comments, file paths, and secrets that were committed in the past. Removing a secret from the latest version of a file does not remove it from the repository history.

## Lesson Learned

Version control history must be treated as sensitive. Secrets should never be committed to Git. If a secret is accidentally committed, it must be rotated and removed properly from the repository history. Web servers should also block access to `.git` directories in production.

---

# General Takeaways

## Common Sources of Information Disclosure

During this module, I practiced identifying information leaks from:

- Verbose error messages
- Debug pages
- Developer comments
- Backup files
- Source code exposure
- Insecure HTTP methods
- Internal HTTP headers
- Exposed Git repositories
- Version control history

## Testing Methodology

My general workflow for information disclosure testing is:

1. Browse the application normally.
2. Review HTTP history in Burp Proxy.
3. Inspect response headers and HTML source.
4. Check common files such as `robots.txt` and `sitemap.xml`.
5. Look for comments, hidden paths, and unusual endpoints.
6. Send interesting requests to Burp Repeater.
7. Modify parameters to trigger errors.
8. Check for exposed backup files and directories.
9. Test for unnecessary HTTP methods such as `TRACE`.
10. Investigate exposed version control directories such as `.git`.

## Remediation Notes

To reduce information disclosure risks, applications should:

- Disable verbose error messages in production.
- Remove debug endpoints before deployment.
- Strip unnecessary developer comments from production responses.
- Prevent public access to backup files.
- Block access to version control directories.
- Avoid hardcoding secrets in source code.
- Rotate secrets that were accidentally exposed.
- Never trust client-controllable headers for authorization.
- Disable unnecessary HTTP methods such as `TRACE`.
- Review public responses for sensitive data before release.

---

# Final Reflection

This module helped me understand that information disclosure is not always a vulnerability with immediate impact by itself. However, leaked technical details can become extremely useful when combined with other weaknesses.

The most important lesson was that small pieces of information, such as a framework version, a hidden path, a debug variable, an internal header, or an old Git commit, can significantly improve an attacker's ability to understand and exploit an application.
