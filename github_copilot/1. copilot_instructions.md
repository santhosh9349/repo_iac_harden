# GitHub Copilot Instruction Files: A Complete Guide for IaC Repositories

GitHub Copilot instruction files let you embed persistent, project-specific context directly into your repository so that every Copilot suggestion — completions, chat answers, and code reviews — reflects your team's standards automatically. No more repeating "always tag resources" or "use snake_case" in every prompt.

This guide focuses on an **Infrastructure-as-Code (IaC) repository** containing Terraform, PowerShell, Bash, and Python, and covers the two types of instruction files in depth.

---

## Table of Contents

1. [How Instruction Files Work](#how-instruction-files-work)
2. [Repo-Wide Instructions](#repo-wide-instructions)
3. [Path-Specific Instructions](#path-specific-instructions)
4. [Repo-Wide vs. Path-Specific: Side-by-Side](#repo-wide-vs-path-specific-side-by-side)
5. [Example 1 — Repo-Wide Instructions for an IaC Repository](#example-1--repo-wide-instructions-for-an-iac-repository)
6. [Example 2 — Path-Specific Instructions for Terraform Files](#example-2--path-specific-instructions-for-terraform-files)
7. [Example 3 — Path-Specific Instructions for Scripting Languages](#example-3--path-specific-instructions-for-scripting-languages)
8. [Best Practices](#best-practices)
9. [Quick-Start Checklist](#quick-start-checklist)

---

## How Instruction Files Work

Copilot reads instruction files automatically before responding. You write them once; they apply to every team member using Copilot in the repo, with no IDE configuration required.

```
your-iac-repo/
├── .github/
│   ├── copilot-instructions.md          ← repo-wide (always loaded)
│   └── instructions/
│       ├── terraform.instructions.md    ← loaded only for *.tf files
│       └── scripts.instructions.md     ← loaded only for *.ps1 / *.sh / *.py files
├── modules/
├── scripts/
└── ...
```

There are **two scopes**:

| Scope | File | When loaded |
|---|---|---|
| Repo-wide | `.github/copilot-instructions.md` | Every Copilot interaction in the repo |
| Path-specific | `.github/instructions/*.instructions.md` | Only when a matched file is open/in scope |

Path-specific instructions **stack on top of** repo-wide instructions — they do not replace them.

---

## Repo-Wide Instructions

### File location

```
.github/copilot-instructions.md
```

### What goes here

The repo-wide file is the **foundation layer**. Put anything that applies regardless of which file a developer is editing:

- Project purpose and architecture overview
- Universal security policies (no hardcoded secrets, use managed identities)
- Cross-language naming conventions
- Version control and review standards

### When to use it

- Standards that span all languages (Terraform, PowerShell, Bash, Python)
- Security or compliance policies that must never be skipped
- Context that helps Copilot understand the overall system

---

## Path-Specific Instructions

### File location

```
.github/instructions/<name>.instructions.md
```

### Required front matter

Every path-specific file **must** start with a YAML front matter block containing an `applyTo` field:

```markdown
---
applyTo: "**/*.tf"
---
```

The `applyTo` value is a [glob pattern](https://docs.github.com/en/get-started/using-git/gitignore#pattern-format). Common patterns:

| Pattern | Matches |
|---|---|
| `**/*.tf` | All Terraform files anywhere in the repo |
| `**/*.ps1` | All PowerShell scripts |
| `scripts/**/*.sh` | Bash scripts under the `scripts/` directory |
| `**/*.{ps1,sh,py}` | PowerShell, Bash, and Python files |
| `modules/network/**` | Everything under the network module |

### What goes here

Rules that are **too specific** for the repo-wide file:

- Language-specific idioms, syntax requirements, and linting rules
- Directory-specific conventions (e.g. naming rules for a `modules/` folder)
- Framework-specific patterns (e.g. Terraform module structure)

### When to use it

- You want different Copilot behavior depending on the file type
- Some rules would be noisy or confusing if applied globally
- A subdirectory has its own conventions distinct from the rest of the codebase

---

## Repo-Wide vs. Path-Specific: Side-by-Side

| Feature | Repo-Wide | Path-Specific |
|---|---|---|
| **File path** | `.github/copilot-instructions.md` | `.github/instructions/*.instructions.md` |
| **`applyTo` front matter** | Not used | Required — glob pattern |
| **Scope** | All files in the repository | Files matching the glob pattern |
| **Relationship** | Base layer, always active | Additive — layers on top of repo-wide |
| **Typical content** | Security policies, cross-language standards | Language rules, directory conventions |
| **Number of files** | One per repo | Many — one per language or area |
| **Supported in** | Copilot Chat, code completion, code review | Copilot Chat (VS Code, Visual Studio), code review, cloud agent |

---

## Example 1 — Repo-Wide Instructions for an IaC Repository

**File:** `.github/copilot-instructions.md`

````markdown
# IaC Repository — Copilot Instructions

## Project Overview
This repository provisions and hardens cloud infrastructure using Terraform (Azure/AWS),
PowerShell (Windows automation), Bash (Linux automation), and Python (helper tooling).
All code must meet CIS benchmark and internal security hardening standards.

## Security (applies to all languages)
- Never hardcode credentials, connection strings, passwords, or API keys.
- Always retrieve secrets from Azure Key Vault, AWS Secrets Manager, or environment variables.
- Prefer managed identities and service accounts over username/password authentication.
- Flag any code that disables TLS verification, logging, or audit trails.

## Naming Conventions
- Resources and variables: `snake_case` in all languages.
- Booleans: prefix with `is_`, `has_`, or `enable_` (e.g., `is_encrypted`, `enable_logging`).
- Constants: `UPPER_SNAKE_CASE`.

## Version Control Standards
- Every change must be traceable to an issue or ticket number.
- Do not suggest inline `TODO` comments — open a tracking issue instead.

## Terraform (global rules)
- Always use `snake_case` for resource names, variable names, and output names.
- Every resource block must include the mandatory tags: `environment`, `owner`, `cost_center`.
- Prefer reusable modules over inline resource blocks.
- Do not use `count` when `for_each` is more appropriate.

## PowerShell (global rules)
- Use `Verb-Noun` naming for all functions (follow approved PowerShell verbs).
- Every script must begin with a `#Requires -Version 5.1` statement.

## Bash (global rules)
- Every script must begin with `#!/usr/bin/env bash`.
- Always enable `set -euo pipefail` immediately after the shebang.

## Python (global rules)
- Follow PEP 8 style guidelines.
- Use type hints on all function signatures.
- Use the `logging` module — never use bare `print()` for operational output.
````

---

## Example 2 — Path-Specific Instructions for Terraform Files

**File:** `.github/instructions/terraform.instructions.md`

This file loads **only when a `.tf` file is open**, layering Terraform-specific rules on top of the repo-wide instructions above.

````markdown
---
applyTo: "**/*.tf"
---

# Terraform-Specific Copilot Instructions

## Module Structure
- Every module must have `main.tf`, `variables.tf`, `outputs.tf`, and `versions.tf`.
- `versions.tf` must declare a `required_providers` block with pinned provider versions.

## State Management
- Always configure a remote backend (Azure Storage or S3). Never use local state.
- Include a `backend.tf` file; do not embed the backend block inside `main.tf`.

## Resource Tagging
Every resource that supports tags must include all of the following:

```hcl
tags = {
  environment = var.environment
  owner       = var.owner
  cost_center = var.cost_center
  managed_by  = "terraform"
}
```

Do not suggest hardcoded tag values — always use input variables.

## Variable Declarations
- Every variable must have a `description` and a `type`.
- Sensitive variables (passwords, keys) must include `sensitive = true`.

```hcl
# Correct
variable "db_password" {
  description = "Database administrator password"
  type        = string
  sensitive   = true
}

# Incorrect — missing description, type, and sensitive flag
variable "db_password" {}
```

## Formatting and Validation
- All suggested code must be compatible with `terraform fmt` (2-space indentation, aligned `=`).
- Suggest `terraform validate` after any structural change.

## Security Rules
- Never set `publicly_accessible = true` on databases.
- Storage accounts must have `allow_blob_public_access = false`.
- Security group rules must not use `0.0.0.0/0` as the source for inbound SSH or RDP.
````

---

## Example 3 — Path-Specific Instructions for Scripting Languages

**File:** `.github/instructions/scripts.instructions.md`

This file loads for PowerShell, Bash, and Python scripts, adding language-specific depth beyond what the repo-wide file covers.

````markdown
---
applyTo: "**/*.{ps1,sh,py}"
---

# Scripting Languages — Copilot Instructions

## PowerShell

### Error Handling
- Set `$ErrorActionPreference = 'Stop'` at the top of every script.
- Use `try/catch/finally` blocks around all external calls (REST APIs, file I/O, registry).
- Never suppress errors with `-ErrorAction SilentlyContinue` unless explicitly justified by a comment.

### Style
- Do not use aliases (`ls`, `gci`, `%`, `?`) — use full cmdlet names for readability.
- Use `[CmdletBinding()]` and `param()` blocks for all scripts that accept parameters.

```powershell
# Correct
[CmdletBinding()]
param (
    [Parameter(Mandatory)]
    [string]$ResourceGroupName
)

# Incorrect — positional args, no param block
param($rg)
```

### Security
- Validate and sanitize all inputs before using them in commands or file paths.
- Use `ConvertTo-SecureString` for any password handling; never store as plain text.

---

## Bash

### Robustness
- Always start scripts with `set -euo pipefail` to catch errors early.
- Quote all variable expansions: use `"${VAR}"` not `$VAR`.
- Check that required tools are available before using them:

```bash
command -v terraform >/dev/null 2>&1 || { echo "terraform not found"; exit 1; }
```

### Style
- Use `local` for all variables inside functions.
- Prefer `[[ ]]` over `[ ]` for conditionals.
- Add a usage function and call it when `--help` is passed.

### Security
- Never `eval` user-supplied input.
- Avoid world-writable temporary files — use `mktemp` to create secure temp files.

---

## Python

### Script Entry Points
- All scripts must use `argparse` for argument parsing — never use `sys.argv` directly.
- Wrap execution logic in `if __name__ == "__main__":`.

```python
# Correct
def main() -> None:
    parser = argparse.ArgumentParser(description="Provision storage accounts")
    parser.add_argument("--env", required=True)
    args = parser.parse_args()
    ...

if __name__ == "__main__":
    main()
```

### Logging
- Configure the `logging` module at the top level; set level from an argument or environment variable.
- Use `logger.info()`, `logger.warning()`, `logger.error()` — never bare `print()`.

### Type Safety
- All function signatures must include type hints.
- Use `Optional[T]` (or `T | None` in Python 3.10+) for nullable parameters.

### Security
- Use `subprocess.run()` with a list of arguments — never pass shell strings to avoid injection.
- Never log sensitive values (tokens, passwords, keys) at any log level.
````

---

## Best Practices

### Write instructions, not descriptions
Use short, imperative directives. Copilot acts on commands, not prose.

| Instead of... | Write... |
|---|---|
| "It would be nice if resources had tags" | "Every resource must include the mandatory tags block" |
| "We use snake_case generally" | "Use `snake_case` for all resource names, variables, and outputs" |
| "Credentials should probably not be hardcoded" | "Never hardcode credentials. Always retrieve secrets from Key Vault" |

### Include the "why" for non-obvious rules
When a rule has a non-obvious reason (compliance, a past incident, a subtle bug), a one-line rationale helps Copilot make better decisions in edge cases.

```markdown
- Do not use `count` to create multiple similar resources — use `for_each` instead.
  (count causes destructive replacements when items are reordered; for_each is stable by key)
```

### Provide code examples
Show a correct and an incorrect pattern side by side. Examples outperform abstract rules.

### Keep files focused and under ~1,000 lines
Beyond that, response quality can degrade. Split large files by language or domain.

### Iterate incrementally
Start with 10–20 specific instructions targeting your most frequent review findings. Add more after testing Copilot's behavior on real code.

### Test your instructions
Open a relevant file, ask Copilot to "review this file against our coding standards", and verify it cites your rules correctly.

---

## Quick-Start Checklist

```
[ ] Create .github/copilot-instructions.md
      [ ] Add project overview (2-3 sentences)
      [ ] Add universal security policies
      [ ] Add cross-language naming conventions

[ ] Create .github/instructions/terraform.instructions.md
      [ ] Add applyTo: "**/*.tf" front matter
      [ ] Add module structure requirements
      [ ] Add mandatory tagging block with example
      [ ] Add state management rules

[ ] Create .github/instructions/scripts.instructions.md
      [ ] Add applyTo: "**/*.{ps1,sh,py}" front matter
      [ ] Add PowerShell error handling and style rules
      [ ] Add Bash robustness (set -euo pipefail, quoting)
      [ ] Add Python argparse and logging requirements

[ ] Test: open a .tf file → ask Copilot "review against our standards"
[ ] Test: open a .ps1 file → ask Copilot "review against our standards"
[ ] Iterate: add rules for common review findings over time
```

---

## Further Reading

- [Adding repository custom instructions for GitHub Copilot](https://docs.github.com/en/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [GitHub Copilot custom instructions tutorials](https://docs.github.com/en/copilot/tutorials/customization-library/custom-instructions)
- [Unlocking the full power of Copilot code review: master your instruction files](https://github.blog/ai-and-ml/github-copilot/unlocking-the-full-power-of-copilot-code-review-master-your-instructions-files/)
- [5 tips for writing better custom instructions for Copilot](https://github.blog/ai-and-ml/github-copilot/5-tips-for-writing-better-custom-instructions-for-copilot/)
