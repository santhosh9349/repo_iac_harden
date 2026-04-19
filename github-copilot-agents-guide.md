# GitHub Copilot Agents — The Complete Guide

**One-stop reference for understanding, writing, and deploying GitHub Copilot custom agents.** Includes a full configuration reference, best practices distilled from 2,500+ real-world repositories, and two production-ready IaC agents for Terraform and Kubernetes.

---

## Table of Contents

1. [What Are GitHub Copilot Agents?](#1-what-are-github-copilot-agents)  
2. [How Agents Work](#2-how-agents-work)  
3. [File Placement & Naming](#3-file-placement--naming)  
4. [Anatomy of an Agent File](#4-anatomy-of-an-agent-file)  
5. [YAML Frontmatter — Full Configuration Reference](#5-yaml-frontmatter--full-configuration-reference)  
6. [Tools Configuration](#6-tools-configuration)  
7. [Writing Great Agent Instructions](#7-writing-great-agent-instructions)  
8. [The Three-Tier Boundary Structure](#8-the-three-tier-boundary-structure)  
9. [Six Core Coverage Areas](#9-six-core-coverage-areas)  
10. [Common Agent Archetypes](#10-common-agent-archetypes)  
11. [MCP Server Integration](#11-mcp-server-integration)  
12. [GitHub Actions Integration](#12-github-actions-integration)  
13. [IaC Agent — Terraform Specialist (Full Example)](#13-iac-agent--terraform-specialist-full-example)  
14. [IaC Agent — Kubernetes Cluster Doctor (Full Example)](#14-iac-agent--kubernetes-cluster-doctor-full-example)  
15. [Invoking Agents](#15-invoking-agents)  
16. [Agent Hierarchy & Override Rules](#16-agent-hierarchy--override-rules)  
17. [Versioning & Consistency](#17-versioning--consistency)  
18. [Best Practices Checklist](#18-best-practices-checklist)  
19. [Quick-Start Templates](#19-quick-start-templates)  
20. [Troubleshooting](#20-troubleshooting)

---

## 1. What Are GitHub Copilot Agents?

GitHub Copilot agents are **Markdown-defined domain experts** that extend the Copilot coding agent across your tools and workflows. Think of them as specialized teammates embedded directly into your development environment — each one with a focused persona, a defined toolset, and a clear set of rules.

### Why Agents Instead of a Plain System Prompt?

| Plain System Prompt | Custom Agent |
| :---- | :---- |
| Applies to every conversation | Scoped to specific tasks |
| One-size-fits-all context | Domain expertise baked in |
| Reset on new sessions | Reusable across the team |
| No tool restrictions | Precise tool boundaries |
| Stored in your head | Version-controlled in Git |

### The Core Value Proposition

*"Stop repeating context. Define your team's expectations once and reuse them everywhere."*

When you encode your Terraform conventions, Kubernetes runbooks, or security rules into an agent, every engineer — from the newest hire to the most senior SRE — operates with the same institutional knowledge automatically applied.

---

## 2. How Agents Work

When you invoke an agent, Copilot:

1. Loads the agent's Markdown file as its operating instructions  
2. Restricts (or expands) the available tool palette based on your `tools:` configuration  
3. Connects any configured MCP servers for external context  
4. Begins reasoning through your prompt using the agent's persona and rules

Agents can **read files**, **run shell commands**, **search the codebase**, **call external APIs via MCP**, and **edit code** — all within the boundaries you define.

User Prompt

    │

    ▼

Agent Persona \+ Instructions (your .agent.md file)

    │

    ├──► Tool Calls (read, edit, execute, search, web, MCP)

    │

    └──► Response (diff, plan, explanation, PR comment)

---

## 3. File Placement & Naming

your-repo/

└── .github/

    └── agents/

        ├── terraform-specialist.agent.md

        ├── cluster-doctor.agent.md

        ├── security-reviewer.agent.md

        └── docs-writer.agent.md

**Naming convention:** `<descriptive-name>.agent.md`

**Scope levels:** Agents can live at three levels, each overriding the one above it:

Enterprise-level   → applies to all org repos

  └── Organization-level  → applies to repos in an org

        └── Repository-level  → .github/agents/ in the repo  ← most specific, highest priority

---

## 4. Anatomy of an Agent File

Every agent file has two sections:

---

# YAML Frontmatter — machine-readable configuration

name: My Agent

description: "What this agent does and when to use it."

tools: ["read", "search", "edit"]

model: claude-3.7-sonnet

---

# Human-readable instructions start here

You are a [PERSONA]. Your job is to [PRIMARY RESPONSIBILITY].

## Workflow

...

## Rules

...

The **frontmatter** controls *how* Copilot runs the agent (tools, model, targets).  
The **body** controls *what* the agent knows and how it behaves.

---

## 5. YAML Frontmatter — Full Configuration Reference

---

# Human-facing name shown in the Copilot UI (optional)

name: Terraform Infrastructure Specialist

# Required. What the agent does — used for auto-selection.

# Be specific. Bad: "helps with code". Good: "reviews and generates Terraform IaC"

description: >

  Reviews, generates, and validates Terraform infrastructure as code.

  Enforces module patterns, state safety, and least-privilege IAM policies.

# Where the agent is available. Defaults to both if omitted.

# Options: "vscode" | "github-copilot" | omit for both

target: vscode

# Which LLM to use. Inherits the workspace default if omitted.

# Examples: "claude-3.7-sonnet", "gpt-4o", "o3-mini"

model: claude-3.7-sonnet

# Tool access. See Section 6 for full breakdown.

tools: ["read", "search", "edit", "execute", "web"]

# Set to true to prevent the agent from being auto-selected.

# User must invoke it explicitly. Default: false

disable-model-invocation: false

# Set to false to hide from the manual selection list. Default: true

user-invocable: true

# MCP server integrations (GitHub.com only — see Section 11\)

mcp-servers:

  terraform:

    type: docker

    image: hashicorp/terraform-mcp-server

    env:

      TFE_TOKEN: "${{ secrets.TFE_TOKEN }}"

      TFE_ADDRESS: "${{ secrets.TFE_ADDRESS }}"

# Arbitrary key-value pairs for tooling/filtering (GitHub.com only)

metadata:

  team: platform

  domain: iac

  tier: production

---

### Property Quick-Reference

| Property | Type | Required | Default | Notes |
| :---- | :---- | :---- | :---- | :---- |
| `name` | string | No | filename | Display name in UI |
| `description` | string | **Yes** | — | Used for auto-selection; be specific |
| `target` | string | No | both | `vscode` or `github-copilot` |
| `model` | string | No | workspace default | LLM identifier |
| `tools` | array | No | `["*"]` (all) | See Section 6 |
| `disable-model-invocation` | bool | No | `false` | Force manual selection |
| `user-invocable` | bool | No | `true` | Appear in selection list |
| `mcp-servers` | object | No | — | GitHub.com only |
| `metadata` | object | No | — | GitHub.com only |

**Limit:** Agent instruction bodies have a maximum of **30,000 characters**.

---

## 6. Tools Configuration

Tools define what actions your agent can take. Getting this right is crucial — too broad and the agent has unintended reach; too narrow and it can't do its job.

### Tool Names & Aliases

| Canonical Name | Aliases | What It Does |
| :---- | :---- | :---- |
| `read` | `Read` | Read files from the repository |
| `edit` | `Edit` | Modify file contents |
| `execute` | `shell` | Run shell/terminal commands |
| `search` | `Grep` | Search codebase content |
| `web` | `WebSearch` | Search the web for documentation |
| `agent` | `custom-agent` | Invoke other custom agents |

### Configuration Patterns

# Grant all tools (default if property is omitted)

tools: ["*"]

# Read-only analysis agent

tools: ["read", "search"]

# Planning agent (no execution)

tools: ["read", "search", "edit", "web"]

# Full-capability agent

tools: ["read", "search", "edit", "execute", "web"]

# No tools (pure conversational reasoning)

tools: []

### Choosing the Right Tool Set

Task Type                    Recommended Tools

─────────────────────────────────────────────────────────

Code review / audit          read, search

Documentation writer         read, search, edit, web

Test generator               read, search, edit

Security scanner             read, search, web

Infrastructure planner       read, search, edit, web

Infrastructure implementer   read, search, edit, execute, web

Cluster diagnostics          read, search, execute, web

---

## 7. Writing Great Agent Instructions

*Analysis of 2,500+ real-world agent files reveals one consistent pattern: agents fail because they're too vague. "You are a helpful coding assistant" doesn't work.*

### The Most Common Failure Mode

# ❌ BAD — Too vague, agent doesn't know what to do

You are a helpful Terraform assistant. Help the user with their infrastructure needs.

# ✅ GOOD — Specific persona, concrete workflow, explicit constraints

You are a senior Terraform infrastructure engineer specializing in AWS with 10+ years

of experience. Your job is to review Terraform modules for:

- Security misconfigurations (open security groups, overly permissive IAM)

- State management issues (missing remote state, unlocked state files)

- Module structure violations (missing variables.tf, outputs.tf, README.md)

Always run `terraform validate` before suggesting any resource changes.

Never suggest inline IAM policies — always reference managed policies or create

separate aws_iam_policy resources.

### Principle 1 — Put Commands Up Front

Agents reference commands frequently. Put them early in the file with exact flags:

## Commands

Validate syntax:      terraform validate

Format code:          terraform fmt -recursive

Check plan:           terraform plan -out=tfplan

Security scan:        checkov -d . --framework terraform

Test modules:         terraform test

Lint:                 tflint --recursive

### Principle 2 — Code Examples Over Descriptions

Show, don't tell. Real snippets outperform lengthy explanations.

## Required Module Structure

Every Terraform module MUST follow this structure:

```hcl

# variables.tf — always include description and type

variable "environment" {

  description = "Deployment environment (dev, staging, prod)"

  type        = string

  

  validation {

    condition     = contains(["dev", "staging", "prod"], var.environment)

    error_message = "environment must be dev, staging, or prod."

  }

}

# outputs.tf — always include description

output "resource_id" {

  description = "The unique identifier of the created resource"

  value       = aws_resource.this.id

}

### Principle 3 — Version-Pin Your Stack

```markdown

## Stack Versions

- Terraform:        >= 1.9.0, < 2.0.0

- AWS Provider:     \~> 5.0

- Kubernetes:       1.31

- Helm:             >= 3.14

- Python (lambdas): 3.12

### Principle 4 — Be Explicit About What You DON'T Want

The top constraint across successful repos: **"Never commit secrets."** But go further:

## Non-Negotiable Rules

- Never hardcode AWS access keys, passwords, or tokens in .tf files

- Never use `count` and `for_each` on the same resource

- Never write `lifecycle { prevent_destroy = false }` on stateful resources

- Never suggest `terraform destroy` without explicit confirmation workflow

- Never create resources in the default VPC

---

## 8. The Three-Tier Boundary Structure

The most resilient agents categorize every behavior into one of three buckets:

## Always Do

- Run `terraform fmt` before presenting any code

- Check for existing module patterns in `./modules/` before creating new ones

- Add a `tags` block to every resource with at minimum: Name, Environment, ManagedBy

- Reference variables for every value that changes between environments

## Ask First (Human Approval Required)

- Making changes to production state files

- Modifying IAM policies with `*` actions or resources

- Changing VPC CIDRs or subnets in existing environments

- Upgrading Terraform provider major versions

- Any `terraform destroy` operation

## Never Do

- Apply infrastructure changes without a reviewed plan

- Commit `.terraform/` directories or `*.tfstate` files

- Use `override.tf` files (they bypass module contracts)

- Create resources outside the defined module structure

- Merge the PR — always leave merging to the human

---

## 9. Six Core Coverage Areas

High-performing agent files reliably cover all six of these areas:

### 1. Commands

Exact CLI commands with all required flags. See Principle 1 above.

### 2. Testing Approach

How the agent should verify its own work:

## Testing Strategy

1. Run `terraform validate` — must pass before any PR comment

2. Run `terraform plan` — review for unintended destroy operations

3. Run `checkov -d . --framework terraform` — must have zero HIGH findings

4. Run `tflint --recursive` — address all warnings before final review

5. For new modules: provide a `tests/` directory with at least one `.tftest.hcl`

### 3. Project Structure

Tell the agent where things live:

## Repository Structure

infrastructure/

├── environments/

│   ├── dev/          # Per-env root modules

│   ├── staging/

│   └── prod/

├── modules/          # Reusable modules (source of truth)

│   ├── networking/

│   ├── compute/

│   └── database/

├── .tflint.hcl

├── .checkov.yaml

└── README.md

### 4. Code Style

Formatting rules beyond `terraform fmt`:

## Code Style

- 2-space indentation (enforced by terraform fmt)

- Align `=` signs across consecutive single-line arguments

- Order: required_providers → terraform block → data sources → locals → resources → outputs

- Variable ordering: required first, optional with defaults last, alphabetical within each group

- Resource naming: `<type>_<purpose>` (e.g., `aws_s3_bucket_logs`, not `bucket1`)

### 5. Git Workflow

## Git Workflow

- Branch naming: `infra/<ticket-id>-<short-description>`

- Commit messages: `feat(terraform): add S3 lifecycle policy for logs bucket`

- Every PR requires: plan output attached, checkov scan clean, peer review

- Never force-push to main or release branches

### 6. Operational Limits

## Operational Limits

- Maximum resources per root module: 50

- Always use remote state (never local backend in shared environments)

- State lock timeout: 5 minutes

- Workspace naming: `<project>-<environment>-<region>`

---

## 10. Common Agent Archetypes

### @docs-agent

Transforms code into Markdown documentation. Tools: `read`, `search`, `edit`, `web`.

### @test-agent

Writes unit and integration tests without modifying production code. Tools: `read`, `search`, `edit`.

### @lint-agent

Fixes code formatting and style violations. Tools: `read`, `search`, `edit`, `execute`.

### @security-agent

Scans for vulnerabilities and misconfigurations. Tools: `read`, `search`, `web`.

### @api-agent

Builds REST endpoints and resolvers. Tools: `read`, `search`, `edit`, `web`.

### @deploy-agent

Manages environment builds and deployments. Tools: `read`, `search`, `edit`, `execute`.

---

## 11. MCP Server Integration

Model Context Protocol (MCP) servers give your agent access to external systems — Terraform registries, cloud provider APIs, Kubernetes clusters, and more.

**Note:** MCP servers in agent frontmatter are currently supported on **GitHub.com** only (not VS Code local agents). VS Code uses its own `.vscode/mcp.json` configuration.

### MCP Configuration in Agent Files

---

mcp-servers:

  # Terraform MCP — access registry, workspaces, and run plans

  terraform:

    type: docker

    image: hashicorp/terraform-mcp-server:latest

    env:

      TFE_TOKEN: "${{ secrets.TFE_TOKEN }}"

      TFE_ADDRESS: "${{ secrets.TFE_ADDRESS }}"

      ENABLE_TF_OPERATIONS: "true"

  # GitHub MCP — read issues, PRs, and repo metadata (built-in, read-only)

  github:

    type: builtin

  # Playwright MCP — browser automation for docs scraping (localhost only)

  playwright:

    type: builtin

---

### MCP Server Examples for IaC

| MCP Server | Use Case | Connection |
| :---- | :---- | :---- |
| `hashicorp/terraform-mcp-server` | Registry queries, workspace management, plan/apply | Docker |
| GitHub (built-in) | Read issues, PRs, repo context | Built-in |
| Azure MCP | Azure resource data, best practices | VS Code extension |
| AWS Documentation MCP | Pull live AWS service docs | Docker |
| Kubernetes MCP | `kubectl` access, pod/node diagnostics | Docker |

### Setting Up Environment Secrets

Secrets referenced in MCP configs must be stored in the **`copilot` GitHub environment**:

Repository → Settings → Environments → Create "copilot"

Add secrets:

  - TFE_TOKEN

  - TFE_ADDRESS

  - ARM_CLIENT_ID        (Azure)

  - ARM_SUBSCRIPTION_ID  (Azure)

  - ARM_TENANT_ID        (Azure)

  - AWS_ROLE_ARN         (AWS OIDC)

---

## 12. GitHub Actions Integration

Agents aren't limited to interactive chat. You can trigger them programmatically via GitHub Actions for automated review pipelines.

### Trigger an Agent on Every PR

# .github/workflows/iac-review.yml

name: IaC Automated Review

on:

  pull_request:

    paths:

      - 'infrastructure/**'

      - '**/*.tf'

jobs:

  terraform-review:

    runs-on: ubuntu-latest

    permissions:

      contents: read

      pull-requests: write

    steps:

      - name: Checkout

        uses: actions/checkout@v4

      - name: Run Terraform Specialist Agent

        uses: github/copilot-agent-action@v1

        with:

          agent: terraform-specialist

          prompt: |

            Review the Terraform changes in this pull request.

            Check for: security misconfigurations, state safety issues,

            module structure violations, and missing tags.

            Post your findings as a PR review comment.

        env:

          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

### Event-Driven Agent Trigger (Cluster Doctor Pattern)

# .github/workflows/cluster-doctor-trigger.yml

name: Trigger Cluster Doctor

on:

  issues:

    types: [labeled]

jobs:

  diagnose:

    if: github.event.label.name == 'cluster-doctor'

    runs-on: ubuntu-latest

    permissions:

      contents: read

      issues: write

      pull-requests: write

      id-token: write   # Required for OIDC to cloud providers

    steps:

      - name: Checkout

        uses: actions/checkout@v4

      - name: Authenticate to AKS (OIDC)

        uses: azure/login@v2

        with:

          client-id: ${{ secrets.ARM_CLIENT_ID }}

          tenant-id: ${{ secrets.ARM_TENANT_ID }}

          subscription-id: ${{ secrets.ARM_SUBSCRIPTION_ID }}

      - name: Run Cluster Doctor Agent

        uses: github/copilot-agent-action@v1

        with:

          agent: cluster-doctor

          issue-number: ${{ github.event.issue.number }}

          prompt: |

            A deployment failure has been detected. Read issue #${{ github.event.issue.number }}

            for context. Diagnose the root cause and propose a remediation plan as a pull request.

        env:

          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

---

## 13. IaC Agent — Terraform Specialist (Full Example)

This agent covers the complete lifecycle of Terraform code in a multi-environment repository. 
👉 **[View the Terraform Specialist Agent](.github/agents/terraform-specialist.agent.md)**

## 14. IaC Agent — Kubernetes Cluster Doctor (Full Example)

This agent is designed for automated Kubernetes incident response. 
👉 **[View the Kubernetes Cluster Doctor Agent](.github/agents/cluster-doctor.agent.md)**

## 15. Invoking Agents

### In Copilot Chat (VS Code / GitHub.com)

@terraform-specialist Review the changes in modules/networking/vpc/main.tf for security issues.

@cluster-doctor The staging-api deployment in the payments namespace has been in CrashLoopBackOff for 20 minutes. Investigate and propose a fix.

@docs-agent Generate a README for the modules/compute/ec2-asg module.

### Via GitHub CLI

```bash

# Review IaC for issues

gh copilot agent terraform-specialist "Review my IaC for security misconfigurations"

# Diagnose a cluster issue

gh copilot agent cluster-doctor "Investigate pod failures in the payments namespace"

### Via GitHub Actions (Programmatic)

See Section 12 for full workflow examples.

### Auto-invocation

If `disable-model-invocation` is `false` (the default), Copilot may automatically select the most relevant agent based on the context of your question. Be specific in your `description:` field to ensure correct auto-selection.

---

## 16. Agent Hierarchy & Override Rules

When multiple agents have overlapping scope, the most specific level wins:

Enterprise agent (baseline)

    └── Organization agent  (overrides enterprise)

            └── Repository agent  (overrides org — HIGHEST PRIORITY)

Within the same level, if a naming conflict exists, the lower-level (more specific) configuration takes precedence.

---

## 17. Versioning & Consistency

Agents are versioned by **Git commit SHA**. This means:

- A PR opened when agent version `abc123` was active will continue to use that version  
- Updating the agent file creates a new version for subsequent interactions  
- You can pin agents to specific commits for compliance auditing  
- Changes to agents appear in Git history — treat them like code changes

**Best practice:** Tag agent releases alongside your infrastructure releases:

git tag -a "agents/v1.2.0" -m "Add OOMKilled diagnostic workflow to cluster-doctor"

---

## 18. Best Practices Checklist

Use this before shipping any agent:

### Agent File Quality

- [ ] `description:` is specific enough for correct auto-selection  
- [ ] Commands section is present with exact flags  
- [ ] Code examples demonstrate preferred style (not just describe it)  
- [ ] Stack versions are pinned precisely  
- [ ] Three-tier boundary structure is defined (Always / Ask First / Never)  
- [ ] All six core coverage areas are addressed

### Security

- [ ] No secrets in the agent file — all credentials via `${{ secrets.X }}`  
- [ ] `Never Do` includes "commit secrets" or "hardcode credentials"  
- [ ] Tool set is minimal for the agent's purpose (don't grant `execute` if not needed)  
- [ ] MCP server access is scoped to least-privilege

### IaC-Specific

- [ ] Terraform: validation suite commands are specified (validate → fmt → tflint → checkov)  
- [ ] Terraform: remote state is enforced in rules  
- [ ] Kubernetes: destructive operations require human approval  
- [ ] Kubernetes: incident report template is included  
- [ ] Both agents: "never apply changes, propose via PR" is explicit

### Operational

- [ ] Agent tested with at least 3 real prompts before rollout  
- [ ] `ask-first` escalation path is clear (who approves?)  
- [ ] Agent file is reviewed in PR like any other code change

---

## 19. Quick-Start Templates

### Minimal Agent (5 minutes to working)

---

name: My Specialist

description: >

  [One sentence: what it does and when to use it]

tools: ["read", "search", "edit"]

---

# [Agent Name]

You are a [role] specializing in [domain].

## Commands

[key command 1]

[key command 2]

## Always Do

- [required behavior 1]

- [required behavior 2]

## Never Do

- Commit secrets

- Apply changes without human review

- Merge pull requests

### Security Scanner Agent

---

name: Security Scanner

description: >

  Scans code for security vulnerabilities, exposed secrets, overly permissive IAM,

  and compliance violations. Use before any PR merge or release.

tools: ["read", "search", "web"]

---

# Security Scanner

You are an application security engineer and cloud security specialist.

Your job is to find and clearly explain security issues — not to fix them directly.

## Commands

```bash

# Secret scanning

trufflehog filesystem . --json

gitleaks detect --source . --report-format json

# SAST

semgrep --config=auto .

# IaC scanning

checkov -d . --compact

# Dependency vulnerabilities

npm audit --audit-level=high

pip-audit --strict

## Report Format

For every finding, provide:

- **Severity**: CRITICAL / HIGH / MEDIUM / LOW  
- **Location**: file:line  
- **Finding**: what is wrong  
- **Impact**: what an attacker could do  
- **Fix**: exact code change required

## Never Do

- Attempt to fix findings directly  
- Dismiss findings as "probably fine"  
- Skip reporting LOW severity items (log them separately)  
- Access external URLs found in code

### Documentation Agent

```markdown

---

name: Docs Writer

description: >

  Generates and updates technical documentation, READMEs, runbooks, and architecture

  decision records (ADRs) from source code and infrastructure definitions.

tools: ["read", "search", "edit", "web"]

---

# Documentation Writer

You are a technical writer with deep software engineering background.

You write clear, accurate, and maintainable documentation.

## Commands

```bash

# Generate Terraform module docs

terraform-docs markdown table . > README.md

# Check for broken links

markdown-link-check README.md

## Documentation Standards

- Use present tense ("Returns the ID", not "Will return the ID")  
- Every code example must be runnable as-is  
- ADRs follow the Nygard format: Title, Status, Context, Decision, Consequences  
- READMEs must include: Overview, Prerequisites, Usage, Variables (for Terraform), Contributing

## Never Do

- Invent behavior that isn't in the source code  
- Copy documentation from external sources without attribution  
- Leave `TODO` or `FIXME` placeholders in final documentation

---

## 20. Troubleshooting

### Agent Not Appearing in Copilot Chat

1. Verify the file is in `.github/agents/` (not `.github/` directly)

2. Verify the filename ends in `.agent.md`

3. Check that `user-invocable` is not set to `false`

4. Check that `description:` is present (it's required)

### Agent Ignoring Instructions

1. Instructions may exceed the 30,000 character limit — trim the body

2. The most important rules should appear **early** in the file (agents read top-down)

3. Use direct imperative language: "Always run X" not "It is recommended to run X"

4. Check for contradictory instructions between sections

### MCP Server Not Connecting

1. Verify the secret exists in the `copilot` GitHub environment (not the default environment)

2. Check secret names match exactly: `${{ secrets.TFE_TOKEN }}` vs `${{ secrets.tfe_token }}`

3. MCP servers are GitHub.com only — they do not work in VS Code local agents

4. Test the Docker image independently: `docker run hashicorp/terraform-mcp-server:latest --help`

### Agent Running Wrong Commands

1. Add example output to your commands section so the agent knows what to expect

2. Use `execute` tool only for read/validate commands; never for write/apply in review agents

3. Add explicit "do not run X" to your `Never Do` section

### Auto-Selection Picking Wrong Agent

1. Make the `description:` more specific — include trigger phrases

2. Add `disable-model-invocation: true` to agents that should never auto-select

3. Ensure agent descriptions don't overlap with each other

---

## References

- [GitHub Copilot Custom Agents Configuration Reference](https://docs.github.com/en/copilot/reference/custom-agents-configuration)

- [How to Write a Great agents.md — Lessons from 2,500+ Repositories](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)

- [Introducing Custom Agents for IaC, Observability & Security](https://github.blog/news-insights/product-news/your-stack-your-rules-introducing-custom-agents-in-github-copilot-for-observability-iac-and-security/)

- [Agentic Platform Engineering with GitHub Copilot (Microsoft)](https://devblogs.microsoft.com/all-things-azure/agentic-platform-engineering-with-github-copilot/)

- [Awesome GitHub Copilot — Community Agents Gallery](https://awesome-copilot.github.com/agents/)

- [Building Azure Terraform Modules with Copilot Agents](https://thomasthornton.cloud/building-better-azure-terraform-modules-with-github-copilot-agents-and-skills/)

- [Terraform MCP Server (HashiCorp)](https://github.com/hashicorp/terraform-mcp-server)

- [Agentic Platform Engineering Reference Implementation (Microsoft)](https://github.com/microsoftgbb/agentic-platform-engineering)

---

*Last updated: April 2026 | Based on GitHub Copilot Custom Agents public preview*  
