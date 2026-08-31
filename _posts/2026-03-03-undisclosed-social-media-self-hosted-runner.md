---
title: "Self-Hosted Runner RCE in a Leading Social Media Company's Open-Source Networking Tool"
date: 2026-03-03 14:00:00 +0200
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, rce, self-hosted-runners, github-actions, networking, undisclosed-vendor, bug-bounty, responsible-disclosure]
pin: false
---

On March 3, 2026, I discovered a **Self-Hosted Runner RCE** vulnerability in an open-source systems framework of a leading social media company.

**Details:**
- **Vulnerability**: Remote Code Execution on Self-Hosted GitHub Actions Runners
- **Target**: Undisclosed — open-source systems project
- **Status**: Vulnerable (unpatched at time of writing)
- **Program**: Undisclosed vendor bug bounty program

The repository's CI pipeline ran a performance job on a persistent self-hosted runner for `pull_request` events from forks, checking out and executing the pull-request author's benchmark script with no fork-protection guard (`head.repo.full_name == github.repository`). With the approval policy auto-triggering runs for returning contributors, a single merged innocuous PR made a fork a "known contributor" — after which a second PR executed arbitrary code on the company's benchmark infrastructure with full runner privileges and a readable GitHub token.

The issue was responsibly disclosed to the vendor's security team; as of this writing the vulnerable workflow still triggers unguarded `pull_request` events on the self-hosted runner, so the company and repository are withheld.

> **Note**: A self-hosted runner "PWN Request" variant — fork pull requests executing untrusted code on a persistent benchmark runner, with no `head.repo.full_name` guard and an approval policy that trusted returning contributors.
