---
title: "PWN Request Vulnerability in Canonical Chisel"
date: 2026-03-04 11:00:00 +0200
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, canonical, pwn-request, github-actions]
pin: false
---

On March 3, 2026, I discovered a **PWN Request** vulnerability in **canonical/chisel**.

**Details:**
- **Vulnerability**: Pull Request Code Execution
- **Target**: canonical/chisel (Ubuntu's file system slicing tool)
- **Status**: Patched
- **Program**: Canonical Security Team

The issue was responsibly disclosed to the Canonical Security Team.

> **Note**: This is the classic GitHub Actions "PWN Request" pattern — checking out and running untrusted pull-request code while retaining access to base-repository secrets.

Thanks to the Canonical Security Team for their swift response and remediation.