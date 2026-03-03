---
name: gitcommit-go
description: Use when generating git commit messages for Go projects. Triggers on "generate commit", "write commit message", "gitcommit-go", "commit文案", "生成commit", "commit message".
---

# Git Commit Message Generator (Go)

Generate a standards-compliant commit message by analyzing git diff. **Output only the commit message. Never execute `git commit`.**

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

### Step 2: Analyze Changes

From the diff output:
- Identify all modified files and their paths
- Count added/removed lines per file
- Identify the primary change pattern

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

### Step 3: Infer Scope

Determine scope from the file paths changed:

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

### Step 4: Generate Commit Message

Format:
```
<type>(<scope>): <description>

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
- Changes span multiple files or modules
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

## Examples

### Single core change
```
feat(auth): add JWT token refresh mechanism
```

### Core change with related secondary changes
```
feat(auth): add JWT token refresh mechanism

- Update authentication documentation
- Add unit tests for token refresh flow
- Fix code formatting issues
```

### Fix with context
```
fix(api): resolve user data validation error

- Update API documentation with validation rules
- Add error handling for edge cases
```

### Large refactor with breaking change
```
refactor(core): restructure approval module architecture

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
- Generating multiple commit messages → **STOP**. Always produce ONE unified message.
- Adding explanation before or after the commit message → **STOP**. Output only the commit message.
- Using past tense (`added`, `fixed`) → **STOP**. Use imperative (`add`, `fix`).
- Subject line over 72 characters → **STOP**. Shorten or omit scope.
- Vague descriptions (`fix bug`, `update code`) → **STOP**. Be specific about what changed.

## Quick Reference

| Element | Rule |
|---------|------|
| Type | Required. Use priority table above. |
| Scope | Optional but recommended. Infer from file paths. |
| Description | Required. Imperative, lowercase, ≤72 chars total. |
| Body | Optional. Use when multiple related changes exist. |
| Footer | Optional. For issue refs and breaking changes only. |
