---
title: "Self-Hosted Runner RCE in a Leading Tech Company's Cross-Platform Mobile Framework"
date: 2026-04-20 10:00:00 +0200
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, rce, self-hosted-runners, github-actions, mobile-framework, undisclosed-vendor]
pin: false
---

On April 20, 2026, I reported a **Self-Hosted Runner RCE** vulnerability in the cross-platform mobile framework of a leading tech company.

**Details:**
- **Vulnerability**: Remote Code Execution on Self-Hosted GitHub Actions Runners
- **Target**: Undisclosed — cross-platform mobile development framework
- **Status**: Fixed
- **Program**: Undisclosed vendor security team

The company's workflow ran on `[self-hosted, linux, shared]` runners for `pull_request` events from forks, and the approval policy appeared to auto-trigger builds for returning contributors without manual review. A malicious pull request that modified `build.gradle` or Android source files could therefore execute arbitrary code on the vendor's self-hosted CI infrastructure.

The issue was responsibly disclosed to the vendor's security team; no response has been received as of this writing, so the company and repository are withheld.

> **Note**: A self-hosted runner "PWN Request" variant — fork pull requests executing untrusted code on persistent `[self-hosted, linux, shared]` runners, with an approval policy that appeared to trust returning contributors.
