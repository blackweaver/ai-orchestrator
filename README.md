# AI-CLI — AI-Powered Development Workflow

> A command-line orchestrator that supercharges Claude Code, GitHub Copilot CLI, Cursor Agent CLI, and Codex CLI with structured workflows for debugging, code reviews, feature planning, refactoring, and more.

---

## What is AI-CLI?

AI-CLI is a bash script that acts as an intelligent middleware between you and AI coding assistants (Claude Code CLI, GitHub Copilot CLI, Cursor Agent CLI, and Codex CLI). Instead of manually crafting prompts and gathering context every time, AI-CLI automatically collects relevant repository information, combines it with your project's documentation, and sends structured, context-rich prompts to your preferred AI provider.

Think of it as a **context orchestrator** — it knows what information each type of task needs and assembles the perfect prompt automatically.

---

## Why Use AI-CLI?

| Benefit                         | Description                                                                                         |
| ------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Structured Workflows**        | 16 pre-built commands across the full development lifecycle, plus free-form chat                    |
| **Automatic Context Injection** | Gathers git status, recent commits, changed files, and project structure                            |
| **Project Conventions**         | Resolves `CLAUDE.md` / `docs/` / `.claude/rules/` imports in bash so **every** provider gets them   |
| **Token Efficiency**            | Sends only relevant context per task type, avoiding unnecessary token consumption                   |
| **Provider Agnostic**           | Switch between Claude, Copilot, Cursor, and Codex with a single word                                |
| **Persistent Documentation**    | Generates structured markdown files and optional interactive HTML task plans in `.ai-private/`      |
| **Dry-Run Mode**                | Preview the exact prompt and provider command before execution                                      |
| **Model Override**              | Override the default model per run with `--model`                                                   |
| **Base Branch Detection**       | Diffs against `master`, or `main` when `master` doesn't exist                                       |

---

## Getting Started

### Prerequisites

1. **Git** — The script works exclusively within git repositories
2. **At least one provider CLI** (install only the ones you plan to use):

   | Provider  | Install                                                                                   |
   | --------- | ----------------------------------------------------------------------------------------- |
   | `claude`  | `npm install -g @anthropic-ai/claude-code` — [guide](https://docs.anthropic.com/en/docs/claude-code) |
   | `copilot` | `npm install -g @github/copilot` — [guide](https://docs.github.com/en/copilot/concepts/agents/about-copilot-cli) |
   | `cursor`  | `curl https://cursor.com/install -fsS \| bash` (installs `cursor-agent`)                  |
   | `codex`   | `npm install -g @openai/codex` — [guide](https://github.com/openai/codex)                 |

3. **jq** (required for the `cursor` provider — used to parse its streamed JSON output)
   ```bash
   brew install jq
   ```
4. **gh CLI** (optional, used by the `suggestions` command to read the current PR)
   ```bash
   brew install gh
   ```

### Installation

```bash
mkdir -p ~/bin
cp ai-cli ~/bin/ai-cli
chmod +x ~/bin/ai-cli
```

Add `~/bin` to your PATH if not already present:

```bash
# Add to ~/.zshrc or ~/.bashrc
export PATH="$HOME/bin:$PATH"
source ~/.zshrc
```

### Environment Variables

AI-CLI does not source any env file itself — each provider CLI (`claude`, `copilot`, `cursor-agent`, `codex`) must already be authenticated and have any API keys it needs available in your shell environment before you run `ai-cli`. Run each provider's own login command (e.g. `codex login`) or export the keys in your `~/.zshrc`/`~/.bashrc`.

### Repository Setup

In your project's root directory:

```bash
# Base context file (used by suggestions and test, and as the entry point for project conventions)
touch CLAUDE.md AGENTS.md

# Optional: private context files (add .ai-private/ to .gitignore)
mkdir -p .ai-private
touch .ai-private/PLANNING.md     # feature specs and implementation plans
touch .ai-private/DEBUG.md        # bug descriptions and reproduction steps
touch .ai-private/REVIEW.md       # review output and fix suggestions
touch .ai-private/REFACTOR.md     # refactoring summaries
touch .ai-private/BUG-HUNTING.md  # criteria/checklist used by the audit command
touch .ai-private/LEARNED.md      # lessons from past reviews (read by review)

echo ".ai-private/" >> .gitignore
```

A missing context file is **not** fatal: `ai-cli` prints `Warning: Missing file ... (skipped)` and continues without it.

**File purposes:**

| File                                   | Purpose                                                                                              |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `CLAUDE.md`                            | Project-wide instructions. Its `@path` lines are expanded into the conventions context               |
| `AGENTS.md`                            | Agent instructions, included in the conventions context                                              |
| `docs/`                                | Convention docs referenced from `CLAUDE.md` (e.g. `docs/workflow/git.md`, `docs/testing/playwright.md`) |
| `.claude/rules/*.md`                   | Optional path-scoped rules (`paths:` globs + an `@doc` reference)                                    |
| `.github/PULL_REQUEST_TEMPLATE.md`     | Template the `pr` command follows                                                                    |
| `.ai-private/PLANNING.md`              | Feature specs and plans (read by `feature`, `planning`; written by `planning`)                       |
| `.ai-private/DEBUG.md`                 | Bug descriptions and reproduction steps (read by `debug`)                                            |
| `.ai-private/REVIEW.md`                | Review output and fix suggestions (written by `review`, `debug`, `audit`, `suggestions`, `playwright-debug`; read by `fix`, `refactor`, `review`) |
| `.ai-private/REFACTOR.md`              | Refactoring summaries (written by `refactor`)                                                        |
| `.ai-private/BUG-HUNTING.md`           | Bug-hunting criteria for the branch's changes (read by `audit`)                                      |
| `.ai-private/LEARNED.md`               | Notes from old reviews to keep in mind (read by `review`)                                            |
| `.ai-private/tasks/`                   | Interactive HTML plans (written by `debug`, `review`, `planning` when planning HTML is enabled)      |
| `.ai-private/.generated-docs-context.md` | Auto-generated copy of the last resolved conventions context, for manual inspection                |

---

## Usage

### Basic Syntax

```bash
ai-cli [flags] <provider> [flags] <command> "your request"
```

**Providers:** `claude` | `copilot` | `cursor` | `codex`

### Flags

| Flag                    | Short | Description                                                                                              |
| ----------------------- | ----- | -------------------------------------------------------------------------------------------------------- |
| `--dry-run`             | `-d`  | Show the generated prompt and provider command without executing                                        |
| `--model <name>`        | `-m`  | Override the default model for this run                                                                 |
| `--planning`            | `-p`  | Plain switch — generate the interactive HTML task visualization for `debug`, `review`, `planning` (off by default) |

Flags can appear **before or after the provider, but not after the command**:

```bash
ai-cli --dry-run claude review "..."
ai-cli claude --dry-run review "..."

ai-cli --model haiku claude lint "..."
ai-cli claude --model gpt-5-codex lint "..."   # explicit provider model

ai-cli claude --planning planning "design a caching layer"
```


### How each provider is launched

| Provider  | Mode                       | Command (simplified)                                                                                  |
| --------- | -------------------------- | ----------------------------------------------------------------------------------------------------- |
| `claude`  | Interactive session        | `claude --dangerously-skip-permissions [--model X] "<prompt>"`                                        |
| `copilot` | Interactive session        | `copilot [--model X] --allow-all-tools --allow-all-paths --add-dir $REPO_ROOT -i "<prompt>"`          |
| `cursor`  | Non-interactive, streamed  | `cursor-agent --print --force --trust --output-format stream-json --stream-partial-output [--model X] --add-dir $REPO_ROOT "<prompt>" \| jq …` |
| `codex`   | Interactive session        | `codex [--model X] --dangerously-bypass-approvals-and-sandbox --add-dir $REPO_ROOT "<prompt>"`        |

> ⚠️ `--dangerously-skip-permissions` (claude) and `--dangerously-bypass-approvals-and-sandbox` (codex) disable the provider's safety prompts and sandbox. Run `ai-cli` only in repositories you trust, and review changes with `git diff` afterwards.

---

## Commands

### `audit` — Bug-hunt the branch's changes

Reviews the current source-control changes (staged, unstaged, and against the base branch) against the criteria in `BUG-HUNTING.md` and the project conventions, looking for bugs, inefficiencies, and unclear code introduced by this branch.

```bash
ai-cli claude audit "focus on the payment retry logic"
```

- **Context:** branch, status, commits, staged files, unstaged files, changed files vs the base branch
- **Model:** sonnet
- **Reads:** `.ai-private/BUG-HUNTING.md`, project conventions (smart)
- **Output:** `.ai-private/REVIEW.md` (overwritten), grouped by severity

---

### `commit` — Generate a commit message

Generates a commit message for the staged changes following `docs/workflow/git.md`: `[TICKET-ID] type: description`, imperative mood, under 80 characters, types `feat/fix/refactor/docs/test/chore`, no `Co-Authored-By` trailer. The ticket ID is extracted from the branch name.

```bash
ai-cli claude commit "focus on the auth changes"
```

- **Context:** branch name and the staged diff (`git diff --cached`)
- **Model:** sonnet
- **Reads:** `docs/workflow/git.md`
- **Output:** Commit message printed to the terminal (the model returns only the message)

---

### `debug` — Root cause analysis

Analyzes the bug described in `DEBUG.md`: root cause, minimal reproduction steps, smallest safe fix, and regression tests. Does **not** modify source files.

```bash
# Uses opus by default (most capable — see note on cost below)
ai-cli claude debug "login throws 500 on special characters"

# Override to a cheaper model
ai-cli claude --model sonnet debug "..."
```

- **Context:** branch, status, commits, changed files, diff stats, `package.json` scripts
- **Model:** **opus** (override with `--model sonnet` to reduce cost)
- **Reads:** `.ai-private/DEBUG.md`, project conventions (smart)
- **Output:** `.ai-private/REVIEW.md`, `.ai-private/tasks/debug-{timestamp}.html` (with `--planning`)

---

### `explain` — Explain code or architecture

Explains a piece of code, a design pattern, or how a feature works.

```bash
ai-cli claude explain "how does the auth middleware work?"
ai-cli codex explain "what is the data flow in the checkout process?"
```

- **Context:** branch, status, commits, changed files, relevant file tree (depth 3, max 200)
- **Model:** **opus**
- **Reads:** full project conventions
- **Output:** Explanation printed to the terminal

---

### `feature` — Implement a feature

Implements the feature described in `.ai-private/PLANNING.md`, following project conventions and creating/updating unit tests.

```bash
# First, describe what to build in .ai-private/PLANNING.md (or generate it with `planning`)
ai-cli claude feature "implement the user auth flow"
```

- **Context:** branch, status, commits, changed files, relevant file tree (depth 3, max 200)
- **Model:** provider default (no model forced; use `--model` to choose)
- **Reads:** `.ai-private/PLANNING.md`, full project conventions
- **Output:** Source file changes

---

### `fix` — Apply fixes from a review

Applies the bugs and improvements listed in `.ai-private/REVIEW.md`. Avoids touching code outside this branch's changes.

```bash
ai-cli claude fix "apply the security and performance suggestions"
```

- **Context:** branch, status, commits, changed files (vs the base branch)
- **Model:** sonnet
- **Reads:** `.ai-private/REVIEW.md`, project conventions (smart)
- **Output:** Source file changes

---

### `lint` — Fix linting issues

Runs `npm run check` live, analyzes the output, and fixes ESLint and Prettier issues in the changed files.

```bash
ai-cli claude lint "fix all lint errors in the changed files"
```

- **Context:** branch, status, commits, changed files, **live output** of `npm run check`
- **Model:** sonnet
- **Reads:** project conventions (smart)
- **Output:** Source file changes

---

### `planning` — Generate an implementation plan

Creates an implementation plan (architecture overview, files to create/modify, risks and mitigations, implementation order, test strategy), split into small, tested PRs. Does **not** modify source files.

```bash
ai-cli claude planning "design a caching layer for the API endpoints"
ai-cli claude --planning planning "design a caching layer for the API endpoints"
```

- **Context:** branch, status, commits, changed files, branch commits, base branch recent commits, source files (.ts/.tsx/.js, max 150), full file tree (depth 3, max 100)
- **Model:** **opus**
- **Reads:** `.ai-private/PLANNING.md`, full project conventions
- **Output:** `.ai-private/PLANNING.md`, `.ai-private/tasks/planning-{timestamp}.html` (with `--planning`)

---

### `playwright-debug` — Analyze Playwright test failures

Runs `npm run playwright` live and analyzes the E2E output to find failures introduced by this branch: root cause, affected component, suggested fix, and which HAR mocks are stale. It also reviews changed spec files for patterns that could corrupt HAR recordings on a future `UPDATE_MOCKS` run, reported in its own **HAR Recording Risks** section.

```bash
ai-cli claude playwright-debug "investigate the failing checkout e2e"
```

- **Context:** branch, status, commits, changed files, changed `.playwright.ts` specs, changed `.har` mocks, `playwright.config.ts`, Playwright test files, live `npm run playwright` output
- **Model:** sonnet
- **Reads:** project conventions (smart)
- **Output:** `.ai-private/REVIEW.md`, grouped by severity

---

### `playwright-test` — Write a new Playwright E2E test

Writes a new Playwright test for the flow described in your request, following `docs/testing/playwright.md` (fixtures, hoisted `shouldUpdate`, HAR mocks, `storageState`). It reuses existing page objects when available, and never hand-writes HAR content — new mocks must be recorded with `UPDATE_MOCKS`.

```bash
ai-cli claude playwright-test "cover the checkout flow with a saved card"
```

- **Context:** branch, status, commits, existing page objects, existing tests, and links to the team's external Playwright docs (Mocking System, Pitfalls)
- **Model:** sonnet
- **Reads:** `docs/testing/playwright.md`, project conventions (smart)
- **Output:** New test under `app/playwright/tests/<authenticated|unauthenticated>/<domain>/<Name>.playwright.ts`

---

### `pr` — Generate a PR description

Generates a pull request description for the current branch against the base branch, following `.github/PULL_REQUEST_TEMPLATE.md` (Jira Link, What is changing?, How is this tested?, OWASP Top 10 Checklist). The ticket ID is extracted from the branch name.

```bash
ai-cli claude pr "highlight the security improvements"
```

- **Context:** branch, status, commits, changed files
- **Model:** sonnet
- **Reads:** `.github/PULL_REQUEST_TEMPLATE.md`
- **Output:** PR description (markdown) printed to the terminal

---

### `refactor` — Apply a refactoring

Applies the refactoring described in `.ai-private/REVIEW.md`, preserving behavior while improving code quality, and ensures tests pass.

```bash
ai-cli claude refactor "apply the suggested performance optimizations"
```

- **Context:** branch, status, staged files, unstaged files, changed files
- **Model:** provider default (no model forced; use `--model` to choose)
- **Reads:** `.ai-private/REVIEW.md`, project conventions (smart)
- **Output:** Source file changes, `.ai-private/REFACTOR.md`

---

### `review` — Code review

Reviews all changed files against the base branch: bugs, style violations, missing tests, performance issues, and security concerns, per the project conventions and `LEARNED.md`. Does **not** modify source files.

```bash
ai-cli claude review "focus on security and error handling"
ai-cli codex review "check for performance regressions"
```

- **Context:** branch, status, commits, changed files
- **Model:** provider default (no model forced; use `--model` to choose)
- **Reads:** `.ai-private/REVIEW.md`, `.ai-private/LEARNED.md`, project conventions (smart)
- **Output:** `.ai-private/REVIEW.md`, `.ai-private/tasks/review-{timestamp}.html` (with `--planning`)

---

### `suggestions` — Fetch PR review comments

Uses the GitHub MCP to get the fix-suggestion comments from the branch's PR, builds a fix plan, writes it to `REVIEW.md`, and ensures tests pass.

```bash
ai-cli claude suggestions "prioritize the blocking comments"
```

- **Context:** branch, status, commits, changed files, current PR info (`gh pr view`: number, URL, title)
- **Model:** sonnet
- **Reads:** `CLAUDE.md`
- **Output:** `.ai-private/REVIEW.md`
- **Requires:** `gh` CLI, an open PR for the current branch, and a GitHub MCP server configured in your provider

---

### `test` — Analyze test failures

Runs `npm run test`, captures the output, and asks the AI to diagnose root causes (not just make tests pass) and suggest fixes.

```bash
ai-cli claude test "the auth tests are failing after the middleware refactor"
```

- **Context:** branch, status, commits, changed files, **live test output** (`npm run test`)
- **Model:** sonnet
- **Reads:** `CLAUDE.md`
- **Output:** Fix suggestions printed to the terminal

---

### `types` — Fix TypeScript errors

Fixes TypeScript type errors in the changed files. Avoids `any` — uses `$TSFixMe` with a comment as a last resort.

```bash
ai-cli claude types "fix the type errors introduced by the new API response shape"
```

- **Context:** branch, status, commits, changed files
- **Model:** sonnet
- **Reads:** project conventions (smart)
- **Output:** Source file changes

---

### Free prompt (chat)

Any unrecognized command is treated as a free-form prompt — direct AI interaction with no structured workflow, no repo context, and no default model.

```bash
ai-cli claude "explain how the authentication middleware works"
ai-cli copilot "generate unit tests for the UserService class"
ai-cli cursor "summarize the open TODOs in this file"
ai-cli codex "write a regex that validates semantic versions"
```

---

## Providers & Models

### Default model per command

| Command                                                                         | Default Model | Notes                                                               |
| ------------------------------------------------------------------------------- | ------------- | ------------------------------------------------------------------- |
| `debug`                                                                         | opus          | Most expensive; override with `--model sonnet` if cost is a concern |
| `planning`                                                                      | opus          | Complex reasoning task                                              |
| `explain`                                                                       | opus          | Best for architectural explanations                                 |
| `audit`, `commit`, `fix`, `lint`, `playwright-debug`, `playwright-test`, `pr`, `suggestions`, `test`, `types` | sonnet |                                          |
| `feature`, `refactor`, `review`                                                 | *(none)*      | Uses the provider's own default unless `--model` is given           |
| free chat                                                                       | *(none)*      | Uses the provider's own default unless `--model` is given           |

### Model names per provider

| Provider  | Accepted `--model` values                                                                                                                                 |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `claude`  | `opus`, `sonnet`, `haiku` (or full Claude model IDs) — passed through as-is                                                                               |
| `copilot` | Explicit Copilot models (e.g. `gpt-4o`, `claude-sonnet-4`). `opus`/`sonnet`/`haiku` are silently mapped to the Copilot default; full `claude-*` IDs print a warning and fall back to it |
| `cursor`  | Explicit Cursor Agent models (e.g. `gpt-5`, `sonnet-4-thinking`). `opus`/`sonnet`/`haiku` map to the Cursor default; full `claude-*` IDs warn and fall back |
| `codex`   | Explicit OpenAI models (e.g. `gpt-5-codex`, `gpt-5`, `o3`). `opus`/`sonnet`/`haiku` map to the Codex default; full `claude-*` IDs warn and fall back       |

This means the per-command defaults above only take effect on `claude`; on the other providers they resolve to the provider's own default model. The header printed before every run shows the translation (e.g. `Model : sonnet → (default)`).

---

## How It Works

```
ai-cli claude review "check for security issues"
         │
         ▼
1. Parse flags (--dry-run, --model, --planning) and provider
2. Validate: inside a git repo
3. Detect base branch, auto-create .ai-private/ if missing
         │
         ▼
4. Load static context
   • Project conventions: CLAUDE.md → AGENTS.md + docs/*.md, with `@path`
     imports resolved in bash (+ .claude/rules/*.md), saved to
     .ai-private/.generated-docs-context.md
       - "smart": only rules whose `paths:` globs match the changed files
       - "full":  every doc referenced by CLAUDE.md and every rule
   • .ai-private/*.md (command-specific)
         │
         ▼
5. Generate dynamic context
   • git branch, status, log
   • git diff --name-only $BASE_BRANCH...HEAD
   • File tree / diff stats (command-specific)
   • Live command output (lint, test, playwright-debug)
   • PR info from gh pr view (suggestions only)
         │
         ▼
6. Assemble structured prompt
   ### TASK            — command instructions
   ### STATIC CONTEXT  — your docs
   ### DYNAMIC CONTEXT — git info + command output
   ### USER REQUEST    — your prompt
         │
         ▼
7. Execute AI provider
   • claude  --dangerously-skip-permissions [--model X] "<prompt>"
   • copilot [--model X] --allow-all-tools --allow-all-paths --add-dir $REPO_ROOT -i "<prompt>"
   • cursor-agent --print --force --trust --output-format stream-json \
       --stream-partial-output --add-dir $REPO_ROOT "<prompt>" | jq -r -j '<stream formatter>'
   • codex   [--model X] --dangerously-bypass-approvals-and-sandbox --add-dir $REPO_ROOT "<prompt>"
         │
         ▼
8. Output
   • AI response / interactive session in terminal
   • Generated .md files in .ai-private/ (written by the model itself)
   • Interactive HTML plans in .ai-private/tasks/ (with --planning)
```

### Context Functions

| Function                      | Used by                                                  | What it includes                                                                 |
| ----------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `base_repo_context()`         | all (via the others)                                     | Branch name, `git status`, last 8 commits                                        |
| `changed_files_context()`     | most                                                     | Changed files vs the base branch                                                 |
| `review_context()`            | fix, pr, review, suggestions, test, types, lint          | `base_repo_context` + changed files                                              |
| `feature_context()`           | explain, feature                                         | `review_context` + file tree (depth 3, max 200)                                  |
| `refactor_context()`          | audit, refactor                                          | `base_repo_context` + staged, unstaged, and changed files                        |
| `debug_context()`             | debug                                                    | `review_context` + diff stats + `package.json` scripts                           |
| `planning_context()`          | planning                                                 | `review_context` + branch commits + base recent commits + source files + full tree |
| `playwright_debug_context()`  | playwright-debug                                         | `review_context` + changed specs/HARs + Playwright config and test files         |
| `playwright_test_context()`   | playwright-test                                          | `base_repo_context` + existing page objects/tests + external docs links          |

---

## Dry-Run Mode

Preview the exact prompt and provider command without executing anything (live commands such as `npm run test` are **not** run in dry-run):

```bash
ai-cli --dry-run claude review "check for code smells"
ai-cli -d codex planning "add WebSocket support"
```

The dry-run output shows:

- Provider, command, branch, model, dry-run and planning flags
- The exact CLI command that would be run
- The prompt (without static context, to keep it readable)
- Prompt size in characters and lines

---

## Base Branch Detection

Commands that diff "against the branch" use the first of these that exists: `master`, `origin/master`, `main`, `origin/main`. If none exist it falls back to `master`. The result is shown in the prompts (e.g. "against $BASE_BRANCH") and used by `review`, `audit`, `fix`, `pr`, `playwright-debug`, and the context builders.

---

## Token Efficiency

Each command gathers only the context relevant to its task:

| Command                                              | Why scoped this way                                                      |
| ---------------------------------------------------- | ------------------------------------------------------------------------ |
| `commit`                                             | Only needs the staged diff and the git convention doc                    |
| `review`, `fix`, `lint`, `types`, `pr`               | Only needs the changed files list                                        |
| `audit`, `refactor`                                  | Needs staged vs unstaged distinction                                     |
| `debug`                                              | Adds diff stats and package scripts for runtime context                  |
| `feature`, `explain`                                 | Adds a file tree so the AI knows where to write/look                     |
| `planning`                                           | Broadest context — needs the full project picture                        |
| `test`, `lint`, `playwright-debug`                   | Run the project's tooling live and inject the real output                |
| `smart` conventions (most commands)                  | Only the docs whose `.claude/rules` globs match the changed files        |
| `full` conventions (`feature`, `planning`, `explain`)| No reliable diff to scope against, so every referenced doc is included   |

File trees are capped at 100–200 entries and limited to depth 3 to prevent context overflow.

---

## Contributing

### Reporting Issues

1. Check existing issues before creating a new one
2. Include your OS, shell version, and CLI versions
3. Provide the command you ran and the error output
4. Use `--dry-run` to share the generated prompt without exposing your code

### Submitting Changes

1. Fork the repository
2. Create a branch: `git checkout -b feature/my-feature`
3. Follow the existing patterns for new commands
4. Test with the `claude`, `copilot`, `cursor`, and `codex` providers using `--dry-run`
5. Submit a PR with a clear description of what and why

### Adding a New Command

1. Add the command name to `KNOWN_COMMANDS` (line ~210)
2. Add a `context_function()` if the command needs custom context gathering
3. Add a `case` block calling `run()` with the instruction, context, model, dynamic command, and files
4. Add the command to `usage()` for **each** provider (claude, copilot, cursor, codex)
5. Document it in this README

### Adding a New Provider

1. Add a branch for it in `translate_model()`
2. Add its dry-run command in the `case "$PROVIDER"` block of `run_ai()`
3. Add its real execution in the second `case "$PROVIDER"` block of `run_ai()`
4. Add its entries to `usage()` and document its prerequisites and model names here

---

## License

MIT License — Feel free to use, modify, and distribute.

---

**Made for developers who want AI to work smarter, not harder.**

If you ask for better question, will get a better answer - [Ricardo Castillo (dilettant poet)](https://www.ricardocastillo.com)
