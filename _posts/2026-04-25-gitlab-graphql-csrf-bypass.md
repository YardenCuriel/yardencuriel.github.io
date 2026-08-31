---
title: "GraphQL Channel CSRF Bypass in GitLab"
date: 2026-04-25 17:09:00 +0000
categories: [Security Research, Vulnerability Disclosure]
tags: [vulnerability, gitlab, csrf, graphql, websocket, hackerone]
pin: false
---

On April 25, 2026, I reported a **CSRF bypass** in **GitLab** to their HackerOne program.

**Details:**
- **Vulnerability**: GraphQL Channel CSRF Bypass
- **Target**: GitLab (`GraphqlChannel` WebSocket at `/-/cable`)
- **Status**: Closed as Informative
- **Program**: GitLab Security Team

GitLab exposes GraphQL through two entry points: HTTP `/api/graphql` (`GraphqlController`) and the ActionCable WebSocket `/-/cable` (`GraphqlChannel`). The HTTP path enforces CSRF token validation, a 10,000-character query size cap, and search rate limiting — the channel enforces none of these, and passes any query or mutation straight to `GitlabSchema.execute`. An authenticated user could execute state-changing mutations (`commitCreate`, `createBranch`, `createIssue`, `updateIssue`) with only a session cookie, and send oversized and rate-limited queries that the HTTP path would reject.

Thanks to the GitLab Security Team for their review.