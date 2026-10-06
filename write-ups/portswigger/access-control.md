---
layout: default
title: "PortSwigger — Access Control"
permalink: /write-ups/portswigger/access-control/
---

# PortSwigger — Access Control

## Overview

This page documents my notes and write-ups from the **Access Control** module of PortSwigger Web Security Academy.

Access control is the application of constraints on who or what is authorized to perform actions or access resources. Broken access controls are among the most prevalent and critical vulnerabilities in web applications, consistently ranking at the top of the OWASP Top 10.

This module covers three main categories of access control failures:

- **Vertical privilege escalation:** a user accesses functionality reserved for higher-privilege roles (e.g., a regular user reaching an admin panel).
- **Horizontal privilege escalation:** a user accesses resources belonging to another user of the same privilege level (e.g., viewing another user's account data).
- **Horizontal-to-vertical escalation:** horizontal access is used as a stepping stone to gain higher privileges (e.g., accessing an admin's account page to steal their password).

---

## Labs Covered and Main Technique

| # | Lab | Difficulty | Main Technique |
| --- | --- | --- | --- |
| 01 | Unprotected admin functionality | Apprentice | robots.txt recon |
| 02 | Unprotected admin functionality with unpredictable URL | Apprentice | JS source leak |
| 03 | User role controlled by request parameter | Apprentice | Cookie forgery |
| 04 | User role can be modified in user profile | Apprentice | JSON mass assignment |
| 05 | User ID controlled by request parameter | Apprentice | IDOR — predictable ID |
| 06 | User ID controlled by request parameter, with unpredictable user IDs | Apprentice | IDOR — GUID leak via blog post |
| 07 | User ID controlled by request parameter with data leakage in redirect | Apprentice | IDOR — data in 302 body |
| 08 | User ID controlled by request parameter with password disclosure | Apprentice | IDOR → horizontal-to-vertical |
| 09 | Insecure direct object references | Apprentice | IDOR — static file reference |
| 10 | URL-based access control can be circumvented | Practitioner | X-Original-URL header |
| 11 | Method-based access control can be circumvented | Practitioner | HTTP method switch (POST → GET) |
| 12 | Multi-step process with no access control on one step | Practitioner | Skipping to unprotected step |
| 13 | Referer-based access control | Practitioner | Forged Referer header |

---

# Lab 01 — Unprotected admin functionality

## Objective

Access the admin panel and delete the user `carlos`. No credentials are provided, the goal is to find and access an unprotected admin panel.

## Methodology

I opened the application in the browser and navigated directly to `/robots.txt` to check which paths the site was attempting to exclude from search engine indexing. The file contained a `Disallow` entry pointing to the admin panel.

I accessed the path directly, without any authentication. The panel loaded without restriction.

From the admin interface I deleted the user `carlos`, completing the lab.

## Finding

The admin panel was located at `/administrator-panel`. It was reachable by any user without authentication or authorization checks. The path was disclosed in `robots.txt`.

## Impact

`robots.txt` is a public file used to instruct web crawlers, not to restrict access. Listing a sensitive path in a `Disallow` entry effectively announces it to anyone who looks. When combined with a complete absence of access control on the endpoint itself, any unauthenticated user can reach and operate the admin interface.

## Lesson Learned

Obscurity is not access control. Hiding a URL does not protect it, the server must enforce authentication and authorization on the resource itself. Any sensitive endpoint must deny access by default and require proof of identity and privilege, regardless of how the URL was discovered.

---

# Lab 02 — Unprotected admin functionality with unpredictable URL

## Objective

Access the admin panel, whose URL is not predictable and is not listed in `robots.txt`, and delete the user `carlos`.

## Methodology

I opened the page source with `Ctrl+U` and searched for references to administrative functionality. The page contained a JavaScript block that conditionally inserted a link to the admin panel based on the value of an `isAdmin` variable.

The URL was embedded as a string literal inside the script:

/admin-1ovgyx

Although the `isAdmin` flag was set to `false` for my session, the script, including the URL, was delivered in full to every user, authenticated or not.

I navigated to the path directly, accessed the panel without any authentication prompt, and deleted `carlos`.

## Finding

The admin panel was accessible at `/admin-1ovgyx`. The URL was disclosed in client-side JavaScript that was served to all users, regardless of their role. No server-side access control protected the endpoint.

## Impact

The use of an unpredictable URL is a form of security by obscurity. It raises the bar for discovery slightly compared to a guessable path, but it provides no real protection because the application itself reveals the URL in its own source code. The browser receives the full script, and any user who reads the page source can find the path.

The fundamental problem is the same as in Lab 01: the server did not verify privilege when the endpoint was accessed.

## Lesson Learned

Client-side logic that conditionally displays a link based on role does not constitute access control. The decision of whether to serve protected content must be enforced on the server side. Code delivered to the browser is readable by anyone, regardless of the conditional logic around it.

---

# Lab 03 — User role controlled by request parameter

## Objective

Access the admin panel at `/admin`, which identifies administrators using a client-controlled cookie, and delete `carlos`. Credentials provided: `wiener:peter`.

## Methodology

I logged in with the provided credentials, enabling Burp Proxy interception. The POST containing the login credentials looked normal. On the subsequent GET to `/my-account`, I observed that the server set a cookie:

Admin=false

I modified all outgoing requests to send `Admin=true` instead. After this change, the admin panel link appeared in the UI. I applied the same modification to the GET request for the admin panel, accessed the interface, and deleted `carlos`.

## Finding

The application determined admin status entirely based on the value of the `Admin` cookie. The cookie was plain text with no signature or server-side session binding. Any user who changed its value to `true` gained full administrative access.

## Impact

Storing an authorization decision in a client-controlled cookie with no integrity protection means any user can escalate their own privileges without needing to compromise any other account. The `Admin=true` value is trivially forgeable.

## Lesson Learned

Authorization decisions must be derived from server-side session state, not from values the client submits. Cookies, hidden fields, and query parameters are all user-controllable. Role information must be stored and checked server-side, tied to the authenticated session.

> **Tip:** In scenarios where the cookie must be modified across many requests, Burp's **Proxy → Match and Replace** feature automates the substitution without requiring manual interception on each request.

---

# Lab 04 — User role can be modified in user profile

## Objective

Access the admin panel at `/admin`, which is only accessible to users with `roleid` of `2`, and delete `carlos`. Credentials provided: `wiener:peter`.

## Methodology

I logged in and used the email update feature on my account page. I observed that the JSON response from the server included a `roleid` field reflecting my current role.

I sent the email update request to Burp Repeater and added `"roleid":2` to the JSON body before resending:

```json
{
  "email": "test@test.com",
  "roleid": 2
}
```

The server accepted the modified request and returned a response confirming that my `roleid` had changed to `2`. I then navigated to `/admin` and deleted `carlos`.

## Finding

The email update endpoint accepted and processed a `roleid` field submitted by the client. The server applied the value without validating whether the user had permission to change their own role.

## Impact

This is a classic **mass assignment** vulnerability: the server binds user-submitted JSON fields directly to a data model, including fields that should be read-only from the client's perspective. An attacker who discovers that the response reveals internal fields can attempt to inject those same fields in the next request.

The role escalation path is: observe `roleid` in a response → include it in the next request with a privileged value → server applies it.

## Lesson Learned

Endpoints that update user data must implement a whitelist of fields the client is allowed to modify. Fields such as `roleid`, `isAdmin`, or `permissions` must never be accepted from user input. The server must derive authorization state from its own data, not from values the client sends.

---

# Lab 05 — User ID controlled by request parameter

## Objective

Retrieve the API key of the user `carlos` by exploiting an IDOR vulnerability. Credentials provided: `wiener:peter`.

## Methodology

I logged in and observed that my account page was loaded at:

/my-account?id=wiener

I modified the `id` parameter to `carlos` and reloaded the page. The application served the account page for `carlos` without any authorization check, exposing his API key directly.

## Finding

The `id` parameter in the URL maps directly to a user account in the backend. The server did not verify that the requesting user was entitled to access the account identified by the parameter.

## Impact

This is a textbook **horizontal privilege escalation** via **IDOR** (Insecure Direct Object Reference). Any authenticated user can access any other user's account data by substituting a known identifier. In this case the identifier was a predictable username.

The impact includes exposure of account data, API keys, and any sensitive information stored on the account page.

## Lesson Learned

The server must verify that the user identified by the active session matches the resource being requested. The value of a URL parameter like `id` must never be trusted as authorization. The backend must check: does the authenticated session belong to `wiener`? If so, only serve `wiener`'s data.

---

# Lab 06 — User ID controlled by request parameter, with unpredictable user IDs

## Objective

Find the GUID of the user `carlos`, use it to access his account, and retrieve his API key. Credentials provided: `wiener:peter`.

## Methodology

My first hypothesis was that a GUID belonging to `carlos` might appear in a blog comment. I opened the first post and inspected both the visible content and the page source. There were no comments from `carlos` and nothing sensitive in the HTML.

Before navigating away, I noticed that the post author's name was rendered as a hyperlink. I clicked it and was redirected to a filtered view of posts by that author. The URL exposed the author's internal GUID as a query parameter:

/blogs?userId=<GUID>

I browsed the blog posts and found one authored by `carlos`. Clicking his name provided his GUID. I then navigated to:

/my-account?id=<carlos-GUID>

His account page loaded and the API key was exposed.

## Finding

The application used GUIDs as user identifiers, which are not guessable. However, the blog post listing feature exposed each author's GUID in a public URL. This transformed a private identifier into publicly accessible data, which could then be reused to exploit the IDOR in the account page endpoint.

## Impact

Using unpredictable identifiers such as GUIDs raises the bar for IDOR attacks by eliminating trivial guessing. However, if the same identifier is leaked elsewhere in the application, through a public-facing feature, the protection is negated. The underlying flaw is unchanged: the server does not verify that the requester owns the resource.

## Lesson Learned

Unpredictable identifiers reduce the risk of IDOR but do not eliminate it. Applications must not use the same identifier for both internal authorization and public-facing references. If users need to be referenced in public contexts (blog posts, comments, reviews), a separate public-facing slug or display name should be used instead of the internal account identifier. Verification on the server side remains mandatory regardless of identifier format.

---

# Lab 07 — User ID controlled by request parameter with data leakage in redirect

## Objective

Retrieve the API key of the user `carlos` by exploiting an IDOR where access control is enforced through a redirect. Credentials provided: `wiener:peter`.

## Methodology

I logged in and observed the account page URL pattern:

/my-account?id=wiener

I modified the `id` parameter to `carlos`. The server responded with an HTTP `302` redirect pointing to the login page, indicating that unauthorized access had been detected. However, I inspected the raw response in Burp before the browser followed the redirect.

The body of the `302` response contained the fully rendered HTML of `carlos`'s account page, including his API key. I extracted the key from the response body and submitted it to complete the lab.

## Finding

The server generated the full account page content for `carlos` before evaluating whether the requesting user was authorized. The authorization check only determined which status code and `Location` header to send, it did not prevent the rendered content from being included in the response body.

## Impact

A redirect response does not guarantee that sensitive data was not transmitted. Browsers follow the `Location` header and discard the response body, which creates the illusion of protection. However, any tool that intercepts raw HTTP traffic, such as Burp Suite or `curl`, reads the full response before following the redirect, exposing whatever data the server included.

## Lesson Learned

Authorization must be evaluated before any sensitive data is rendered or included in a response. When the server decides to redirect, it must return an empty body. Never assume that a redirect prevents the client from reading response content.

---

# Lab 08 — User ID controlled by request parameter with password disclosure

## Objective

Exploit an IDOR vulnerability to obtain the administrator's password, then log in as administrator and delete `carlos`. Credentials provided: `wiener:peter`.

## Methodology

I logged in and observed the account page URL pattern. I changed the `id` parameter from `wiener` to `administrator`. The server returned the administrator's account page.

The page contained a password change form. The current password field was pre-populated with the administrator's actual password, visible in the HTML source even though the browser displayed it as masked asterisks.

I extracted the password from the source, logged in as `administrator`, navigated to the admin panel, and deleted `carlos`.

## Finding

The account page endpoint served any user's account data based on the `id` parameter with no authorization check. The password change form pre-filled the current password value in the HTML, making it recoverable from the page source.

## Impact

This lab demonstrates a **horizontal-to-vertical privilege escalation** chain:

1. **Horizontal IDOR:** access the administrator's account page as a non-admin user.
2. **Credential extraction:** recover the administrator's password from the pre-filled form field.
3. **Vertical escalation:** authenticate as administrator and gain full admin privileges.

The `type="password"` attribute on an input field masks the value visually in the browser but does not protect it. The value is present in the HTML and is accessible through page source, browser DevTools, or any intercepting proxy.

## Lesson Learned

Password change forms should never pre-fill the current password. The field should be empty or use a placeholder. Any sensitive value present in the HTML is readable by anyone who can view the source, regardless of the input type used to display it on screen.

---

# Lab 09 — Insecure direct object references

## Objective

Access the live chat transcript of the user `carlos`, retrieve his password, and use it to log in to his account.

## Methodology

I opened the live chat feature and sent a few messages. The chat used WebSocket connections for real-time communication. I then clicked the "View transcript" button and intercepted the resulting request in Burp. The application sent a request to `/download-transcript`, which responded with a `302` redirect to:

/download-transcript/2.txt

This revealed that transcripts were stored as sequentially numbered `.txt` files. I reasoned that:

1. A file `1.txt` likely existed before mine.
2. Since the live chat did not require authentication, transcripts were probably not segregated by user, all files were stored in the same flat directory.

I sent `GET /download-transcript/1.txt` in Burp Repeater. The response contained the transcript of a previous chat session belonging to `carlos`, in which he disclosed his password. I logged in as `carlos` and completed the lab.

## Finding

Transcripts were stored as statically served files with predictable sequential names. The endpoint `/download-transcript/N.txt` did not require authentication and did not verify ownership of the requested file. Anyone who knew the naming pattern could enumerate and access any transcript.

## Impact

This is an **IDOR with a direct reference to a static file**. The server stored sensitive user data as files accessible directly over HTTP, with no access control and no namespace separation per user. Sequential numeric filenames made enumeration trivial.

> **Note on WebSockets:** WebSocket is a protocol for full-duplex communication over a persistent TCP connection. Unlike HTTP's request/response model, a WebSocket connection stays open after the initial handshake, allowing both sides to send messages at any time. It is commonly used for live chats, real-time notifications, and online games. In Burp Suite, WebSocket traffic appears in the **WebSockets history** tab. The vulnerability in this lab was not in the WebSocket protocol itself, but in the HTTP endpoint that served the stored transcript files.

## Lesson Learned

Sensitive user data must not be served as unauthenticated static files. The download endpoint must verify that the requesting session belongs to the owner of the requested transcript. Sequential identifiers must not be used for sensitive resources. Transcripts should be served through an authenticated API that queries records belonging to the current user.

---

# Lab 10 — URL-based access control can be circumvented

## Objective

Access the admin panel at `/admin` and delete `carlos`. The front-end enforces access control based on the URL in the request line. The back-end supports the `X-Original-URL` header.

## Methodology

Accessing `/admin` directly returned an access denied response from the front-end layer.

The `X-Original-URL` header is a legacy header originally used in Microsoft IIS environments to preserve the original request path after URL rewriting. Some frameworks — including Symfony (PHP) prior to CVE-2018-14773 and several Zend libraries, honoured this header even without verifying that the server was actually running IIS.

The vulnerability arises from a discrepancy: the front-end evaluates access control based on the path in the **request line**, while the back-end resolves the resource based on the path in the **`X-Original-URL` header**. By sending a permitted path in the request line and the restricted path in the header, the front-end check is bypassed.

**First request — accessing the admin panel:**

GET / HTTP/2
Host: <lab-host>
X-Original-Url: /admin

The front-end saw `GET /` (permitted) and passed the request. The back-end read `/admin` from the header and served the panel.

**Second request — deleting carlos:**

Attempting to pass `/admin/delete?username=carlos` entirely in the `X-Original-URL` header returned a bad request error. The back-end reads query parameters from the **request line**, not from the header. The correct approach was to split the two components:

GET /?username=carlos HTTP/2
Host: <lab-host>
X-Original-Url: /admin/delete

The back-end composed the effective request from both sources: the path from the header (`/admin/delete`) and the query string from the request line (`?username=carlos`). The operation succeeded and `carlos` was deleted.

## Finding

The front-end and back-end resolved the target URL from different sources. The front-end applied access control based on the request line path, the back-end honoured the `X-Original-URL` header for routing. This discrepancy allowed bypassing front-end restrictions by placing an innocuous path in the request line and the sensitive path in the header.

## Impact

Access control applied exclusively at the front-end or proxy layer is ineffective when the back-end can be influenced through a separate channel. The attacker does not break the front-end check, they route around it entirely.

## Lesson Learned

Access control must be enforced as close to the resource as possible, on the server that actually serves it. Non-standard headers such as `X-Original-URL` and `X-Rewrite-URL` should be stripped or ignored at the perimeter unless explicitly required. Any header that can influence routing must be treated as a potential attack vector.

---

# Lab 11 — Method-based access control can be circumvented

## Objective

Elevate the privileges of the user `wiener` to administrator by bypassing an access control rule that only applies to a specific HTTP method. Credentials: `wiener:peter` and `administrator:admin`.

## Methodology

I logged in as `administrator` and observed the user management interface. Granting a user administrator privileges sent the following request:

POST /admin-roles
Content-Type: application/x-www-form-urlencoded

username=carlos&action=upgrade

I sent this request to Burp Repeater and replaced the session cookie with one from a `wiener:peter` session. Sending it as `POST` returned an access denied response, the method-level control was in place.

I changed the method from `POST` to `GET`. The server responded with an error indicating that the `username` parameter was missing, not an access denied response. This confirmed that `GET /admin-roles` was not subject to the same access control as `POST /admin-roles`, and that the request had reached the handler.

Since `GET` reads parameters from the query string rather than the request body, I moved the parameters to the URL:

GET /admin-roles?username=wiener&action=upgrade HTTP/2
Cookie: session=<wiener-session>

The server applied the role change and `wiener` was granted administrator privileges.

## Finding

The access control rule was defined for `POST /admin-roles` only. The equivalent `GET` request to the same endpoint was not protected. The server treated `GET /admin-roles` and `POST /admin-roles` as distinct authorization targets.

## Impact

When access control is applied per method+path combination rather than per resource, switching to an alternative method bypasses the restriction entirely. The error message "missing username" on the `GET` request was also informative: it confirmed that the request had passed the authorization layer and reached the application handler, making it a useful signal during testing.

## Lesson Learned

Access control must be applied to the resource (the endpoint), not to a specific combination of resource and method. Any HTTP method that reaches a handler capable of performing a privileged action must be subject to the same authorization check.

---

# Lab 12 — Multi-step process with no access control on one step

## Objective

Elevate the privileges of the user `wiener` to administrator by exploiting a missing access control check on the final step of a multi-step admin workflow. Credentials: `wiener:peter` and `administrator:admin`.

## Methodology

I logged in as `administrator` and used the user management panel to upgrade `carlos` to administrator. The workflow consisted of two steps:

1. Select the user and the action from the admin panel.
2. Confirm the change on a confirmation page.

The confirmation POST contained:

POST /admin-roles

username=carlos&action=upgrade&confirmed=true

I confirmed that the first steps were protected: attempting to access `/admin` or `/admin-roles` directly with a `wiener` session returned access denied.

I sent the confirmation POST to Burp Repeater, replaced the session cookie with a `wiener` session, changed `username` to `wiener`, and submitted the request directly, bypassing steps 1 and 2 entirely:

POST /admin-roles
Cookie: session=<wiener-session>

username=wiener&action=upgrade&confirmed=true

The server applied the role change without checking whether the request came from an authorized user.

## Finding

Steps 1 and 2 of the workflow enforced access control correctly. Step 3, the confirmation POST that actually applied the change, did not verify that the submitting user held administrator privileges. The `confirmed=true` field was accepted as sufficient to proceed.

## Impact

The application assumed that any request reaching the confirmation step had already been authorized by the earlier steps. This assumption is invalid: an attacker can construct and submit the final request directly, without passing through the protected earlier steps. Multi-step workflows do not guarantee sequential execution on the server side.

## Lesson Learned

Every step of a multi-step process that performs or authorizes a privileged action must independently verify the session's authorization. The server must never infer that a client is authorized based on which step they appear to have reached. Fields like `confirmed=true` submitted by the client are not authorization signals, they are client-controlled data.

---

# Lab 13 — Referer-based access control

## Objective

Elevate the privileges of `wiener` to administrator by forging the `Referer` header to bypass access control on a sub-page of the admin interface. Credentials: `wiener:peter` and `administrator:admin`.

## Methodology

I logged in as `administrator` and observed that user privilege changes were performed via:

GET /admin-roles?username=<user>&action=upgrade

Attempting to access this endpoint directly with a `wiener` session returned `401 Unauthorized`, confirming that access control was in place.

Based on the lab description, I formed the hypothesis that the `/admin-roles` sub-page only checked the `Referer` header rather than the session's actual role. If the `Referer` pointed to `/admin`, the server would assume the request originated from the admin panel and permit it.

The `Referer` header is set by the browser to indicate the page the user navigated from. It is fully controllable by the client.

I modified a request with a `wiener` session to include:

GET /admin-roles?username=wiener&action=upgrade HTTP/2
Cookie: session=<wiener-session>
Referer: https://<lab-host>/admin

The server accepted the request and granted `wiener` administrator privileges.

## Finding

The `/admin` page enforced access control correctly. The `/admin-roles` endpoint checked only whether the `Referer` header contained `/admin`, inferring that the request had originated from the admin panel. Since the `Referer` is a client-controlled header, any user can forge it to match the expected value.

## Impact

Using the `Referer` header for access control creates a false sense of security. The header is supplied by the client and can be set to any value using a proxy or scripted HTTP client. It carries no integrity guarantee and cannot be trusted as evidence that a request came from a particular page or user context.

## Lesson Learned

Access control must be based on verifiable server-side state, specifically, the authenticated session and its associated role. Client-controllable HTTP headers such as `Referer`, `X-Forwarded-For`, `X-Original-URL`, and similar values must never be used as authorization signals.

---

# General Takeaways

## Common Patterns Across Labs

All thirteen labs share a single underlying root cause: **the server makes an authorization decision based on data the client can control**, or **applies the check in the wrong layer or at the wrong step**. The specific vector differs, a URL parameter, a cookie, a JSON field, an HTTP header, a file path, or a process step, but the failure is always the same.

A useful framing throughout this module:

> **Where is the control applied vs. what does the back-end actually process?**

When these two things differ, a bypass exists. Labs 10 and 11 (platform misconfiguration) make this explicit, but the same logic applies to parameter-based controls (Labs 3 and 4), redirect-based controls (Lab 7), and multi-step controls (Lab 12).

## Testing Methodology

My general workflow for access control testing:

1. Map the application as an authenticated low-privilege user. Note all endpoints, parameters, and identifiers in requests and responses.
2. Identify any endpoint that refers to a resource by a user-controlled identifier (ID, username, GUID, filename, sequence number).
3. Check whether substituting another user's identifier returns their data.
4. Identify endpoints that perform privileged actions. Check whether access control is applied consistently across all HTTP methods.
5. For multi-step workflows, attempt to submit the final step directly without completing the earlier ones.
6. Review response bodies of redirect responses (3xx). The body may contain rendered content even when the browser does not display it.
7. Check whether non-standard headers (`X-Original-URL`, `X-Rewrite-URL`) are honoured by the back-end.
8. If access to a sub-page is controlled by the `Referer` header, test whether forging the header bypasses the check.
9. Inspect JavaScript and HTML source for URLs, role checks, or internal identifiers not visible in the rendered interface.
10. For any identifier that appears sequential or predictable, test adjacent values.

## Remediation Notes

- Apply access control at the server side, as close to the resource as possible. Never rely on front-end checks, client-supplied headers, or URL obscurity.
- Deny access by default. Every endpoint must require explicit authorization, not just those that appear sensitive.
- Verify, on every request, that the authenticated session is entitled to the resource being requested, not just that the user is logged in.
- Never store authorization state in locations the client can modify: cookies without integrity protection, hidden form fields, query parameters, or JSON fields in update requests.
- Apply access control at every step of a multi-step workflow independently.
- Never use client-controllable headers (`Referer`, `X-Original-URL`, etc.) as authorization signals.
- Do not pre-fill password fields with the current password value.
- Do not serve sensitive files as unauthenticated static resources. Use authenticated API endpoints that verify ownership before returning data.
- Treat redirect responses as non-confidential: do not include sensitive data in the body of a 3xx response.

---

# Final Reflection

This module reinforced a principle that appeared in every lab: **the server must never trust the client to tell it what the client is allowed to do**.

The attacks ranged from trivial (changing a URL parameter) to more subtle (splitting a path across a header and a query string, or submitting the final step of a workflow directly). In every case, the vulnerability existed because the application made an authorization decision based on something the attacker controlled.

The most transferable insight is the distinction between **where a control is applied** and **what the back-end actually processes**. Access controls placed at a proxy layer, in client-side code, or in only some steps of a workflow leave a gap that can be exploited by bypassing or manipulating the input the control evaluates.

Access control bugs are consistently ranked as the most common and critical class of web vulnerability. The reason is that they require human design decisions at every endpoint, and every decision is an opportunity for an assumption to be wrong.