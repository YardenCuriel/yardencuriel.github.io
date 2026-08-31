---
title: "PWN Request & Website Poisoning in a Leading Enterprise Software Company's Architecture Learning Platform"
date: 2026-03-06 16:00:00 +0200
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, pwn-request, github-actions, pull-request-target, rce, ci-cd-security, supply-chain, npm, undisclosed-vendor, bug-bounty, responsible-disclosure]
pin: false
---

On March 6, 2026, I discovered a **PWN Request** vulnerability in the architecture learning platform of a leading enterprise software company.

**Details:**
- **Vulnerability**: PWN Request → RCE on GitHub-hosted CI runners + Website Poisoning
- **Target**: Undisclosed — enterprise software company's architecture learning platform
- **Status**: Fixed
- **Program**: Undisclosed vendor security team

The vulnerability enabled external attackers to execute arbitrary code on GitHub-hosted CI runners by opening a malicious pull request from a fork, and to push attacker-controlled content to the company's public architecture learning website.

The issue was responsibly disclosed to the vendor's security team; no response has been received as of this writing, so the company and repository are withheld.

> **Note**: This is the classic GitHub Actions "PWN Request" pattern — a `pull_request_target` workflow that checks out and runs untrusted code from a fork while retaining write access to production content.