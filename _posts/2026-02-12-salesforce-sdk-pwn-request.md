---
title: "Vulnerability Found in Salesforce Mobile SDK - PWN Request"
date: 2026-02-12 14:00:00 +0200
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, salesforce, pwn-request, github-actions]
pin: false
---

On February 12, 2026, I discovered a **PWN Request** vulnerability in **Salesforce Mobile SDK for iOS**.

**Details:**
- **Vulnerability**: PWN Request
- **Target**: forcedotcom/SalesforceMobileSDK-iOS
- **Status**: Patched
- **Program**: Salesforce Security Team

The issue was responsibly disclosed to the Salesforce Security Team.

> **Note**: This is the classic GitHub Actions "PWN Request" pattern — checking out and running untrusted pull-request code while retaining access to base-repository secrets.

Thanks to the Salesforce Security Team for their swift response and remediation.