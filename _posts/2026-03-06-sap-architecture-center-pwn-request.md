---
title: "PWN Request & Website Poisoning in SAP Architecture Center"
date: 2026-03-06 16:00:00 +0200
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, sap, pwn-request, github-actions, pull-request-target, rce, ci-cd-security, supply-chain, npm, hall-of-fame, responsible-disclosure]
pin: false
---

On March 6, 2026, I discovered a **PWN Request** vulnerability in **SAP/architecture-center**, the repository behind SAP's public Architecture Center website.

**Details:**
- **Vulnerability**: PWN Request, leading to RCE on GitHub-hosted CI runners and Website Poisoning
- **Target**: [SAP/architecture-center](https://github.com/SAP/architecture-center) (SAP Architecture Center)
- **Status**: Fixed
- **Program**: SAP Product Security Response Team
- **Credit**: Listed on SAP's [Credits for Security Researchers](https://support.sap.com/en/my-support/knowledge-base/security-notes-news/credits-for-security-researchers.html?isu_page=1) page (September 2026)

The vulnerability enabled external attackers to execute arbitrary code on GitHub-hosted CI runners by opening a malicious pull request from a fork, and to push attacker-controlled content to the public SAP Architecture Center website.

**Timeline:**
- 2026-03-06: Reported to SAP Product Security.
- 2026-03-08: SAP removed the vulnerable workflow ([`600cd8a5`](https://github.com/SAP/architecture-center/commit/600cd8a59ac13a0eaae6c4a7c964fc10547d9068)). The preview pipeline was later reworked to drop the untrusted push path.
- 2026-09: SAP confirmed the finding and credited me on its Security Researchers Hall of Fame.

The issue was responsibly disclosed to SAP Product Security, who removed the vulnerable workflow within two days of the report.

> **Note**: This is the classic GitHub Actions "PWN Request" pattern: a `pull_request_target` workflow that checks out and runs untrusted code from a fork while retaining write access to production content.

Thanks to the SAP Product Security Response Team for the fast remediation and for the Hall of Fame acknowledgement.