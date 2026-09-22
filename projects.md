---
layout: default
title: Projects
permalink: /projects/
---

# Projects

This page contains practical projects developed during my studies in Offensive Security, Linux, networking and cybersecurity.

The goal is to document not only the final result, but also the problem, technical decisions, implementation, testing and lessons learned from each project.

---

## Kali Linux — Fail-Closed VPN Environment

**Status:** Completed

A virtualized Kali Linux environment configured to prevent network traffic from leaving the machine outside an encrypted VPN tunnel.

### Technologies

- Kali Linux
- VirtualBox
- WireGuard
- Mullvad VPN
- nftables

### Concepts

- Default Deny
- Fail-Closed
- Network Isolation
- Egress Filtering
- Least Privilege

[View project documentation →](https://pedro18102601.github.io/projects/kali-vpn-killswitch/)

---

## Python Port Scanner

**Status:** Completed

A simple TCP port scanner built in Python to understand how TCP connection scanning works and to practice basic network connectivity concepts.

### Technologies

- Python
- socket
- argparse
- ipaddress

### What I practiced

- Building a command-line tool with `argparse`
- Validating IPv4 addresses with `ipaddress`
- Parsing and validating user-provided ports
- Creating TCP sockets with Python
- Understanding how a TCP connection attempt can indicate whether a port is open or closed

### Repository

[View repository](https://github.com/pedro18102601/port-scanner-python)

---

## 2FA Brute-Force Bypass (Broken Rate-Limiting)

**Status:** Completed

A Python script that automates bypassing broken brute-force protection on a 2FA verification flow, built while solving an expert-level lab from PortSwigger's Web Security Academy. The application allowed only 2 code attempts per session before invalidating it, but imposed no limit on how many times the login cycle itself could be repeated — the script exploits this by cycling through re-authentication automatically until the correct 4-digit code is found.

### Technologies

- Python
- `requests` (`requests.Session()`)
- `argparse`
- `logging`
- `re`

### What I practiced

- Identifying and exploiting a logic flaw in session-scoped brute-force protection
- Managing HTTP session state and CSRF token lifecycles across multi-step authentication flows
- Extracting values from HTML responses with regex
- Modeling brute-force success probabilistically (geometric distribution) to estimate expected attack duration
- Structuring error handling to distinguish transient network failures from unexpected application behavior during long-running, unsupervised execution
- Structured, timestamped logging to both console and file for auditing multi-hour runs

### Repository

[View repository](https://github.com/pedro18102601/2fa-bruteforce-bypass-python)

---

_More projects will be added as my studies progress._
