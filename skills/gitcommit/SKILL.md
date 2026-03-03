---
name: gitcommit
description: Use when creating git commits with conventional commit format and emoji. Triggers on "commit my changes", "git commit", "create a commit", "commit this", "stage and commit", "gitcommit".
---

# Smart Git Commit

Analyze repository changes, generate an emoji conventional commit message, and execute `git commit`. Supports `--no-verify` to skip pre-commit hooks and `--amend` to amend the last commit.

## Execution Steps

### Step 1: Gather Repository State

```bash
git status --porcelain
git branch --show-current
git log --oneline -5
git diff --cached
git diff
```

### Step 2: Stage Files (If Nothing Staged)

Check `git status --porcelain` output:
- If staged files exist → proceed with those only
- If **no staged files** → run `git add -A` to stage all modified and new files, then re-run `git diff --cached`

### Step 3: Analyze for Commit Splitting

Examine the staged diff for **distinct logical changes**. Consider splitting when:
- Changes touch **unrelated concerns** (e.g., new feature + dependency update)
- Changes mix **different types** (e.g., feat + refactor + docs in unrelated areas)
- Changes affect **different file groups** (e.g., source code vs CI config)
- The diff is large enough that separate commits would be easier to review

If split is warranted: propose a split plan, stage the first subset, commit it, then continue with remaining changes. Ask user before proceeding with a multi-commit split.

If no split needed: proceed with a single commit.

### Step 4: Generate Commit Message

Format: `<emoji> <type>(<scope>): <description>`

**Type + emoji mapping:**

| Emoji | Type | When |
|-------|------|------|
| ✨ | `feat` | New feature or functionality |
| 🐛 | `fix` | Bug fix |
| 📝 | `docs` | Documentation changes |
| 💄 | `style` | Formatting, code style (no logic change) |
| ♻️ | `refactor` | Code restructure without behavior change |
| ⚡️ | `perf` | Performance improvements |
| ✅ | `test` | Add or fix tests |
| 🔧 | `chore` | Tooling, config, build process |
| 🚀 | `ci` | CI/CD improvements |
| ⏪️ | `revert` | Revert a previous commit |
| 🚑️ | `fix` | Critical hotfix |
| 🔒️ | `fix` | Fix security issues |
| 🏷️ | `feat` | Add or update types |
| 🚚 | `refactor` | Move or rename resources |
| 🏗️ | `refactor` | Architectural changes |
| 🔥 | `fix` | Remove code or files |
| 🎨 | `style` | Improve code structure/format |
| ➕ | `chore` | Add a dependency |
| ➖ | `chore` | Remove a dependency |
| 💥 | `feat` | Introduce breaking changes |
| 🩹 | `fix` | Simple fix for a non-critical issue |
| 🦺 | `feat` | Add or update validation |
| ♿️ | `feat` | Improve accessibility |
| 💡 | `docs` | Add or update source code comments |
| 🌐 | `feat` | Internationalization and localization |
| 🧑‍💻 | `chore` | Improve developer experience |
| 🚨 | `fix` | Fix linter warnings |
| 💚 | `fix` | Fix CI build |
| 📌 | `chore` | Pin dependencies to specific versions |
| 🔖 | `chore` | Release/version tags |
| 🚧 | `wip` | Work in progress |

**Subject line rules:**
- Max 72 characters total
- Imperative mood: `add`, `fix`, `update` (not `added`, `fixes`)
- Lowercase after emoji and type
- No period at end
- Scope is optional: infer from affected module/package

**Choose emoji precisely:** Match the most specific emoji for the change. `🚑️` for critical fixes, not just `🐛`.

### Step 5: Execute Commit

```bash
git commit -m "<emoji> <type>(<scope>): <description>"
```

For multi-line body:
```bash
git commit -m "<emoji> <type>(<scope>): <description>" -m "<body line 1>" -m "<body line 2>"
```

With `--no-verify` flag: append `--no-verify` to skip pre-commit hooks.

With `--amend` flag: append `--amend` to modify the last commit.

## Examples

### Single change
```
✨ feat(auth): add JWT token refresh mechanism
```

### Bug fix
```
🐛 fix(api): resolve nil pointer in user query handler
```

### Multiple related changes (with body)
```
♻️ refactor(core): simplify payment processing logic

- Remove deprecated payment methods
- Update related test cases
```

### Split commit example

Diff contains: new feature in `handler/` + dependency update in `go.mod`:
- Commit 1: `✨ feat(api): add user profile endpoint` (stage handler/ files)
- Commit 2: `➕ chore(deps): add uuid library` (stage go.mod, go.sum)

### Critical fix
```
🚑️ fix(auth): patch token validation bypass vulnerability
```

### Breaking change
```
💥 feat(api): redesign user authentication interface

BREAKING CHANGE: AuthService.Login signature changed
```

## Red Flags - You Are Doing It Wrong

- Generating message but not committing → **STOP**. This skill executes `git commit`. Use `gitcommit-go` if you only want to generate.
- Skipping `git add` check when nothing is staged → **STOP**. Always check and auto-stage if empty.
- Vague descriptions (`fix bug`, `update code`) → **STOP**. Be specific.
- Skipping split analysis for large mixed-concern diffs → **STOP**. Always check for logical separation.
- Using past tense (`added`, `fixed`) → **STOP**. Imperative mood only.
- Subject line over 72 characters → **STOP**. Shorten.
- Omitting emoji → **STOP**. Emoji is required in this format.

## Quick Reference

| Flag | Effect |
|------|--------|
| *(none)* | Stage all if empty → commit |
| `--no-verify` | Skip pre-commit hooks |
| `--amend` | Amend last commit |
