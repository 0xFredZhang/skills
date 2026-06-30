---
name: gitcommit-go
description: Use when generating git commit messages for Go projects. Triggers on "generate commit", "write commit message", "gitcommit-go", "commit文案", "生成commit", "commit message".
---

# Git Commit Message Generator (Go)

Generate a standards-compliant commit message by analyzing git diff. **Output only the commit message(s). Never execute `git commit`.**

## Execution Steps

### Step 1: Gather Repository State

Run these commands to collect all required information:

```bash
git status --porcelain
git branch --show-current
git log --oneline -5
git diff --cached
git diff
```

Also check if a root-level CLAUDE.md exists and read it to detect project architecture:

```bash
cat CLAUDE.md 2>/dev/null || cat claude.md 2>/dev/null
```

### Step 2: Detect Project Architecture

**Is this a multi-module project?**

Check CLAUDE.md for signals like:
- Listed sub-projects or modules (e.g., `wallet`, `tokentool`, `server`)
- Monorepo structure descriptions
- Module-specific directories at the root level

**Decision:**

```
CLAUDE.md describes multiple modules?
├── YES → multi-module mode: use <type>(<module>) format
│         split commits by module if changes span multiple modules
└── NO  → single-module mode: infer scope from file path patterns (Step 3b)
```

### Step 3: Analyze Changes

From the diff output:
- Identify all modified files and their paths
- Group files by their top-level directory / module
- Count added/removed lines per file
- Identify the primary change pattern per module

**Core change priority (highest wins for subject line):**

| Priority | Type | When |
|----------|------|------|
| 1 | `feat` | New files, new interfaces, new functionality |
| 2 | `fix` | Bug fixes, error handling corrections |
| 3 | `refactor` | Code restructure without behavior change |
| 4 | `perf` | Performance optimizations |
| 5 | `style` | Formatting, import ordering (body only) |
| 6 | `docs` | Documentation updates (body only) |
| 7 | `test` | Test additions/changes (body only if feat/fix present) |
| 8 | `chore` | Config, build, dependencies (body only) |
| 9 | `ci` | CI/CD configuration changes |
| 10 | `revert` | Reverting a previous commit |
| 11 | `build` | Build system or external dependency changes |

**Rule:** `feat/fix/refactor/perf` become the subject type. `style/docs/test/chore` go in the body unless they are the only change.

### Step 3a: Multi-Module Mode — Infer Module

Use the module name directly from CLAUDE.md architecture description, matched against file paths:

| Changed files under... | Module scope |
|------------------------|--------------|
| `wallet/` | `wallet` |
| `tokentool/` | `tokentool` |
| `server/` or `api/` | `server` |
| Root config only | omit module scope |

**If changes span multiple modules → plan separate commits, one per module.**

### Step 3b: Single-Module Mode — Infer Scope from Paths

| Scope | File path patterns |
|-------|-------------------|
| `api` | `**/api/**`, `**/handler/**`, `**/route/**` |
| `auth` | `**/auth/**`, `**/middleware/**` |
| `core` | `**/core/**`, `**/service/**`, `**/domain/**` |
| `ui` | `**/ui/**`, `**/web/**`, `**/template/**` |
| `db` | `**/db/**`, `**/repository/**`, `**/migration/**` |
| `config` | `**/config/**`, `*.yaml`, `*.toml`, `*.env` |
| `deps` | `go.mod`, `go.sum` |

Use the scope of the **primary changed package**. Omit scope if changes span too many areas.

### Step 4: Generate Commit Message(s)

**Format:**
```
<type>(<module-or-scope>): <description>

[optional body]

[optional footer]
```

**Subject line rules:**
- Max 72 characters
- Imperative mood: `add`, `fix`, `update`, `remove` (not `added`, `fixes`)
- Lowercase first letter of description
- No period at end
- Describe WHAT was done from a business perspective, not HOW

**Add body when:**
- Multiple related changes exist beyond the core change
- Changes span multiple files within the same module
- Important technical decisions need explanation
- There are side effects or breaking changes

**Body format:**
```
- Update related documentation
- Fix code formatting issues
- Add missing error handling
```

**Add footer when:**
- Closing an issue: `Closes #123`
- Breaking change: `BREAKING CHANGE: describe the change`

**Multi-module output format:**

When changes span multiple modules, output ALL commit messages together with clear separators and staging instructions:

```
========== Commit 1: wallet ==========
Stage: git add wallet/

feat(wallet): add balance query endpoint

- Add HTTP handler for balance lookup
- Add unit tests for balance service

========== Commit 2: tokentool ==========
Stage: git add tokentool/

fix(tokentool): resolve token expiry validation bug

- Correct timestamp comparison logic
- Add edge case handling for zero expiry
```

### Step 5: Enforce Shell-Safe Output

The message is meant to be used directly in `git commit -m "<message>"`. Before outputting, scan EVERY line — subject, body, and footer — and remove or replace any character the shell interprets inside double quotes. One stray character breaks or corrupts the command.

**Forbidden characters (each one breaks `git commit -m "..."`):**

- `"` double quote — ends the message string early. Fix: omit it, or rephrase without quotes.
- `` ` `` backtick — triggers shell command substitution. Fix: write the identifier as plain text.
- `$` dollar sign — triggers variable / `$(...)` expansion. Fix: spell the word out.
- `\` backslash — shell escape character. Fix: omit it; use `/` for paths.
- `!` exclamation mark — triggers bash history expansion. Fix: omit it; rephrase.

**Allowed:** letters, numbers, spaces, and `- _ . , : ; ( ) / @ #`. Apostrophes (`'`) are safe inside double quotes — avoid them only if the message may instead be wrapped in single quotes.

**Most common mistake — backticked identifiers.** Write code identifiers, types, and function names as plain text, never wrapped in backticks:

```
Bad:  fix(api): handle `nil` return from `UserRepo.Find`
Good: fix(api): handle nil return from UserRepo.Find
```

## Examples

### Single module change
```
feat(wallet): add balance query endpoint
```

### Module change with related secondary changes
```
feat(wallet): add balance query endpoint

- Add unit tests for balance service
- Update API documentation
- Fix import ordering
```

### Fix with context
```
fix(tokentool): resolve token expiry validation bug

- Correct timestamp comparison logic
- Add edge case handling for zero expiry
```

### Multi-module changes (output TWO separate messages)
```
========== Commit 1: wallet ==========
Stage: git add wallet/

feat(wallet): add multi-currency support

- Add currency conversion service
- Update wallet model with currency field

========== Commit 2: tokentool ==========
Stage: git add tokentool/

refactor(tokentool): simplify token generation logic

- Remove legacy token format support
- Consolidate token factory methods
```

### Large refactor with breaking change
```
refactor(wallet): restructure approval module architecture

- Remove generated mock files and outdated interfaces
- Introduce new approval repository pattern
- Add permission constants and provider configuration

BREAKING CHANGE: ApprovalService interface has been redesigned
```

### Performance + related changes
```
perf(db): optimize user query with proper indexing

- Add database migration for new indexes
- Update query documentation
- Remove unused query methods
```

## Red Flags - You Are Doing It Wrong

- Executing `git commit` → **STOP**. Output only, never commit.
- Changes span multiple modules but you generated ONE message → **STOP**. Split by module, output one message per module with staging instructions.
- Generating multiple messages for changes within the SAME module → **STOP**. Consolidate into one message.
- Adding explanation before or after the commit message(s) → **STOP**. Output only the message(s) with separators if multi-module.
- Using past tense (`added`, `fixed`) → **STOP**. Use imperative (`add`, `fix`).
- Subject line over 72 characters → **STOP**. Shorten or omit scope.
- Vague descriptions (`fix bug`, `update code`) → **STOP**. Be specific about what changed.
- Ignoring CLAUDE.md architecture info → **STOP**. Always check for multi-module project structure.
- Message contains `"`, a backtick, `$`, `\`, or `!` → **STOP**. These break `git commit -m "..."`. Rewrite with plain shell-safe text (Step 5).
- Wrapping code identifiers in backticks → **STOP**. Write them as plain text (e.g. `UserRepo`, not a backticked token).

## Quick Reference

| Element | Rule |
|---------|------|
| Type | Required. Use priority table above. |
| Module/Scope | Optional but recommended. Use module name (multi-module) or infer from file paths (single-module). |
| Description | Required. Imperative, lowercase, ≤72 chars total. |
| Body | Optional. Use when multiple related changes exist within same module. |
| Footer | Optional. For issue refs and breaking changes only. |
| Multiple modules | Output one commit message per module with `git add <module>/` staging instructions. |
| Shell-safe | Required. No `"`, backtick, `$`, `\`, or `!` anywhere — the message must survive `git commit -m "..."`. |
