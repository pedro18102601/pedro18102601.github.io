---
layout: default
title: "PortSwigger — Path Traversal"
permalink: /write-ups/portswigger/path-traversal/
---

# PortSwigger — Path Traversal

## Overview

This page documents my notes and write-ups from the **Path Traversal** module of PortSwigger Web Security Academy.

Path traversal, also known as directory traversal, occurs when an application uses user-controllable input to build a filesystem path without properly validating where the final path resolves. When this happens, an attacker may be able to read files outside the intended directory, including application files, configuration files, credentials, and operating system files.

The goal of this module was to understand how path traversal works, how common defenses can fail, and how to reason about filesystem paths, URL encoding, path normalization, and validation logic.

---

## Labs Covered and Main Technique

| # | Lab | Main Technique |
|---|-----|----------------|
| 01 | File path traversal, simple case | Basic traversal using `../` sequences |
| 02 | File path traversal, traversal sequences blocked with absolute path bypass | Using an absolute filesystem path |
| 03 | File path traversal, traversal sequences stripped non-recursively | Using nested traversal sequences |
| 04 | File path traversal, traversal sequences stripped with superfluous URL-decode | Using encoded traversal sequences |
| 05 | File path traversal, validation of start of path | Preserving the expected base path before traversal |
| 06 | File path traversal, validation of file extension with null byte bypass | Bypassing extension validation with a null byte |

---

# Lab 01 — File path traversal, simple case

## Objective

Read the contents of a sensitive operating system file by exploiting a file path traversal vulnerability in the product image loading functionality.

## Methodology

I started by browsing the application normally and observing how product images were loaded. In Burp Proxy, I identified an image request that used a `filename` parameter to tell the server which file should be returned.

Because the parameter value was being used to reference a file on the server, I sent the request to Burp Repeater and modified the filename to include directory traversal sequences.

## Key Request Pattern

```http
GET /image?filename=../../../etc/passwd HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

## Finding

The application returned the contents of `/etc/passwd`, confirming that the user-controlled `filename` parameter was being appended to a server-side directory and used in an unsafe filesystem operation.

## Why It Worked

The application appeared to read files from a fixed image directory, but it did not properly validate the final resolved path. The `../` sequence moves one directory up in the filesystem, so chaining multiple traversal sequences allowed the request to escape the intended images directory and reach a file outside it.

## Impact

A vulnerable implementation may allow an attacker to read application source code, configuration files, credentials, logs, or sensitive operating system files.

## Lesson Learned

When a parameter controls a filename, it is important to test whether the application validates the final canonical path, not just whether the requested file appears to be an expected image.

---

# Lab 02 — File path traversal, traversal sequences blocked with absolute path bypass

## Objective

Bypass a defense that blocks traversal sequences and retrieve the contents of `/etc/passwd`.

## Methodology

I first understood that the application blocked payloads containing traversal sequences such as `../`. Instead of trying to move up directories using relative traversal, I tested whether the application would accept an absolute path directly from the filesystem root.

## Key Request Pattern

```http
GET /image?filename=/etc/passwd HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

## Finding

The application returned the contents of `/etc/passwd`, even though traversal sequences were blocked.

## Why It Worked

The defense focused on blocking traversal sequences, but it did not prevent the user from supplying a complete absolute path. If the backend passes the value directly to a filesystem API, `/etc/passwd` can be interpreted as an absolute path rather than a filename inside the intended image directory.

## Impact

Blocking only `../` is not enough. If absolute paths are accepted, an attacker may still be able to reference sensitive files directly.

## Lesson Learned

A filter that only searches for traversal sequences is incomplete. The application must verify that the final resolved path remains inside the expected base directory.

---

# Lab 03 — File path traversal, traversal sequences stripped non-recursively

## Objective

Bypass a sanitization mechanism that strips path traversal sequences from the supplied filename.

## Methodology

The direct traversal payload was not effective because the application removed traversal sequences before using the input. I then tested nested traversal sequences, which are designed to survive weak sanitization logic.

## Key Request Pattern

```http
GET /image?filename=....//....//....//etc/passwd HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

## Finding

The application returned the contents of `/etc/passwd` after receiving nested traversal sequences.

## Why It Worked

The application appeared to remove `../` only once, without recursively re-checking the sanitized result. The sequence `....//` contains an inner traversal pattern. When the inner `../` is stripped, the remaining characters can collapse back into a valid `../` sequence.

Conceptually:

```text
....//  →  ../
```

Because the sanitization was non-recursive, the dangerous sequence reappeared after the first stripping pass.

## Impact

Non-recursive sanitization can create a false sense of security. An attacker may be able to craft input that becomes dangerous only after the first sanitization step.

## Lesson Learned

Sanitization should not rely on simple string replacement. The safer approach is to canonicalize the final path and verify that it starts with the expected base directory.

---

# Lab 04 — File path traversal, traversal sequences stripped with superfluous URL-decode

## Objective

Bypass a filter that blocks path traversal sequences before the application performs an additional URL decode operation.

## Methodology

The application blocked direct traversal input. I tested encoded traversal sequences to understand whether the filter and the final filesystem access were interpreting the input at different stages.

The successful payload used URL encoding so that the traversal sequence would not appear as a literal `../` during the initial filtering step, but would later decode into a traversal sequence before being used by the application.

## Key Request Pattern

```http
GET /image?filename=..%252f..%252f..%252fetc/passwd HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

## Encoding Breakdown

The important part is the double-encoded slash:

```text
%252f → %2f → /
```

So after decoding, the payload becomes equivalent to:

```text
../../../etc/passwd
```

## Finding

The application returned the contents of `/etc/passwd`, showing that the input was filtered before all decoding had been completed.

## Why It Worked

The application blocked visible traversal sequences, but it later performed a URL-decode operation before using the filename. This means the dangerous path was not visible to the filter in its final form. After decoding, the traversal sequence reappeared and was used in the filesystem path.

## Impact

Different layers of a web application may decode and normalize input differently. If validation happens before the final decoding/canonicalization step, dangerous input can bypass the filter.

## Lesson Learned

Input should be decoded and normalized before validation, and the final canonical path should still be checked against the expected base directory.

---

# Lab 05 — File path traversal, validation of start of path

## Objective

Bypass a validation rule that requires the supplied filename to start with an expected base directory.

## Methodology

The application sent the full file path in the request parameter and validated that the supplied path started with the expected image directory. Instead of omitting the expected prefix, I preserved it and appended traversal sequences afterward.

## Key Request Pattern

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

## Finding

The application returned the contents of `/etc/passwd`, even though the supplied path started with the expected base directory.

## Why It Worked

The validation checked the beginning of the raw string, but it did not verify the final canonical path. Although the input started with `/var/www/images/`, the traversal sequences moved the resolved path outside that directory.

Conceptually:

```text
/var/www/images/../../../etc/passwd
```

resolves to:

```text
/etc/passwd
```

## Impact

Prefix validation alone is insufficient. An attacker can satisfy the prefix check while still escaping the intended directory using traversal sequences later in the path.

## Lesson Learned

The important question is not whether the raw input starts with the expected directory. The important question is whether the canonicalized final path remains inside that directory.

---

# Lab 06 — File path traversal, validation of file extension with null byte bypass

## Objective

Bypass a validation rule that requires the supplied filename to end with an expected image extension.

## Methodology

The application required the filename to end with a valid image extension, such as `.png`. To bypass this check, I used a null byte before the expected extension.

## Key Request Pattern

```http
GET /image?filename=../../../etc/passwd%00.png HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

## Finding

The application returned the contents of `/etc/passwd`, even though the supplied value appeared to end with `.png`.

## Why It Worked

The input satisfied the application's extension check because it ended with `.png`. However, the null byte sequence represented a string terminator in the vulnerable file handling context. As a result, the filesystem operation interpreted the effective path as ending before `.png`.

Conceptually:

```text
../../../etc/passwd%00.png
```

was treated as:

```text
../../../etc/passwd
```

## Impact

Relying only on extension validation can be bypassed in vulnerable contexts. If the backend does not safely handle the final resolved path, an attacker may still access unauthorized files.

## Lesson Learned

File extension checks are not enough to prevent path traversal. They should be combined with strict allowlists and canonical path validation.

---

# General Testing Methodology

During this module, I used the following methodology:

1. Identify request parameters that appear to reference files, especially image-loading endpoints.
2. Send the request to Burp Repeater.
3. Modify one parameter at a time and compare the responses.
4. Start with basic traversal sequences.
5. If basic traversal fails, identify the likely defense.
6. Test whether absolute paths are accepted.
7. Test whether traversal sequences are stripped non-recursively.
8. Test encoded or double-encoded traversal payloads when decoding behavior is suspected.
9. Check whether the application validates the start of the path or the file extension.
10. Confirm the vulnerability by retrieving a known file in the lab environment.

---

# Common Bypass Patterns

## Basic traversal

```text
../../../etc/passwd
```

Used when there is no effective defense against traversal sequences.

## Absolute path bypass

```text
/etc/passwd
```

Used when traversal sequences are blocked, but absolute filesystem paths are still accepted.

## Nested traversal sequence

```text
....//....//....//etc/passwd
```

Used when the application strips traversal sequences in a non-recursive way.

## Encoded traversal

```text
%2e%2e%2f
```

Represents:

```text
../
```

## Double-encoded traversal

```text
%252e%252e%252f
```

Decodes in two stages:

```text
%252e%252e%252f → %2e%2e%2f → ../
```

## Required base path bypass

```text
/var/www/images/../../../etc/passwd
```

Used when the application checks whether the input starts with the expected base directory.

## Null byte extension bypass

```text
../../../etc/passwd%00.png
```

Used when the application requires a specific file extension but the file handling context treats the null byte as a string terminator.

---

# Prevention Notes

The safest approach is to avoid passing user-controlled input directly to filesystem APIs. When that is not possible, defenses should include:

- Using allowlists of permitted file identifiers instead of raw filenames.
- Rejecting unexpected characters and path separators.
- Normalizing and canonicalizing the final path before use.
- Verifying that the canonical path starts with the expected base directory.
- Avoiding simple blacklist-based string replacement as the only defense.
- Preventing direct access to sensitive files through filesystem permissions and deployment architecture.

A secure design should not depend only on removing `../` from user input. The application should validate the final path that will actually be used by the filesystem.

---

# Final Reflection

This module helped me understand that path traversal is not only about memorizing payloads like `../../../etc/passwd`. The important skill is understanding how the application builds paths on the server side and where validation fails.

The main lesson was that weak defenses often validate the raw input instead of the final resolved path. Absolute paths, nested traversal sequences, encoded characters, base path prefixes, and null bytes all demonstrate the same underlying problem: the application trusts user-controlled input before confirming where the path actually resolves.

For future testing, I will focus first on identifying file-related parameters and then reasoning about the type of validation being applied. Instead of randomly trying payloads, I will compare responses and choose bypasses based on the application's behavior.

---

# References

- PortSwigger Web Security Academy — Path traversal: https://portswigger.net/web-security/file-path-traversal
- PortSwigger Web Security Academy — Path traversal learning path: https://portswigger.net/web-security/learning-paths/path-traversal
- PortSwigger Web Security Academy — File path traversal labs: https://portswigger.net/web-security/all-labs
