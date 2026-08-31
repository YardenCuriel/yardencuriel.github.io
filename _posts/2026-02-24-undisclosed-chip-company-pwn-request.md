---
title: "Self-Hosted GPU Runners in a Leading Chip Company's ML Inference Engine"
date: 2026-02-24 15:00:00 +0200
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, pwn-request, github-actions, self-hosted-runner, ml-framework, gpu-computing, undisclosed-vendor]
pin: false
---

On February 24, 2026, I discovered a **PWN Request** vulnerability in the GPU ML inference engine repository of a leading chip company.

**Details:**
- **Vulnerability**: PWN Request — untrusted pull-request code executed on self-hosted GPU runners
- **Target**: Undisclosed — GPU ML inference engine (graph optimizer/compiler)
- **Status**: Fixed
- **Program**: Undisclosed vendor security team

The repository's CI/CD pipeline used `pull_request_target` and checked out the pull-request author's fork, then built and executed that code on the company's self-hosted GPU runners — with repository secrets, host networking, and direct GPU device access. A single pull request from a fork was enough to achieve code execution on internal GPU test infrastructure.

The issue was responsibly disclosed to the vendor's security team; no response has been received as of this writing, so the company and repository are withheld.

> **Note**: This is the classic GitHub Actions "PWN Request" pattern — a `pull_request_target` workflow that checks out and runs untrusted code while retaining access to base-repo secrets and self-hosted runners.