---
title: "PWN Request in Rancher Elemental Toolkit"
date: 2026-03-04 11:00:00 +0200
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, rancher, pwn-request, github-actions, composite-action, kubernetes, suse]
pin: false
---

On March 4, 2026, I discovered a **PWN Request** vulnerability in **rancher/elemental-toolkit**.

**Details:**
- **Vulnerability**: Pull Request Code Execution (untrusted `pull_request_target` checkout)
- **Target**: rancher/elemental-toolkit (Rancher's OS management toolkit)
- **Status**: Fixed
- **Program**: SUSE/Rancher Security Team

The `cache-cleanup` workflow triggered on `pull_request_target` and checked out the pull-request head SHA, then ran a composite action resolved from that attacker-controlled checkout. A pull request from a fork was enough to execute arbitrary code in the CI pipeline that builds the operating system images used to provision Rancher Kubernetes nodes.

> **Note**: The classic GitHub Actions "PWN Request" pattern — `pull_request_target` combined with an untrusted checkout. Successfully patched by the Rancher security team.

Thanks to the SUSE Rancher Security Team for their swift response and remediation.
