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

_More projects will be added as my studies progress._
