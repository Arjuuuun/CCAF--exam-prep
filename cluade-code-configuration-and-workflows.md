# CCA-F Domain 2: Claude Code Configuration & Workflows
## Consolidated Study Notes (20% of exam)

---

## 1. CLAUDE.md — Three Levels of Memory/Scope

**Underlying concept:** `CLAUDE.md` is Claude Code's persistent instruction/context file, operating at three distinct scopes, each with a different audience, lifecycle, and version-control status.

| Level | Location | Committed? | Audience | Confirmed Precedence |
|---|---|---|---|---|
| **User level** | `~/.claude/CLAUDE.md` (home directory) | No — never committed | Personal to the individual, applies across **all** their projects | Lowest priority (inferred — "most specific wins") |
| **Project level** | `CLAUDE.md` at project root, inside `.claude/` | Yes — committed, shared | Whole team, applies to the entire project | Middle priority |
| **Directory level** | `CLAUDE.md` inside a specific subfolder (e.g., `api/CLAUDE.md`) | Yes — committed | Scoped to just that folder/subtree | **Highest priority** (confirmed live) |

**Confirmed fact (live demo):** Directory-level rules take priority over project-level rules when both apply to the same file. Not by default present: a fresh machine has **no** user-level `CLAUDE.md` unless manually created.

**Situation:** A rule ("all API responses must include a request ID") was written in a developer's *user-level* CLAUDE.md. A new teammate clones the repo and never sees it — because user-level never travels through git. **Fix:** move the rule to the project-level CLAUDE.md so it's committed and shared.

**Situation:** A project-level CLAUDE.md contains "I prefer Tailwind over CSS, my name is Vijay" — personal preference leaking into a shared team file. **Fix:** move to user-level.

---

## 2. Import — Don't Repeat Rules

**Underlying concept:** Avoid duplicating the same rule text across multiple CLAUDE.md files (e.g., "always return JSON" copy-pasted into `api/CLAUDE.md` and `db/CLAUDE.md`).

**Confirmed syntax:** `@import filename.md` — e.g., `@import standard-dates.md` pulls a shared rules file into the current CLAUDE.md, single source of truth, update once.

**Situation:** Date-format validation rules are needed in multiple places across a project. Define them once in `standard-dates.md`, then `@import standard-dates.md` from the project-level CLAUDE.md (and anywhere else needed) — rather than duplicating the rule text.

---

## 3. Split the Monolith

**Underlying concept:** A single CLAUDE.md covering testing, API conventions, deployment, and security all at once (1000+ lines) becomes hard to navigate/maintain.

**Fix:** Split into topic-specific files (`testing.md`, `api-conventions.md`, `deployment.md`) within `.claude/`, referenced/imported as needed. Rule of thumb: once CLAUDE.md grows past roughly one screen, split it.

**Distinct from → CLAUDE.md vs. Skills (Section 5):** Splitting a monolith keeps content *always-loaded* but organized. Moving occasional/task-specific content to **Skills** makes it *load-on-demand only* — two different fixes for two different symptoms. Don't conflate them.

---

## 4. Diagnosing Inconsistent Behavior — `/memory`

**Underlying concept:** When Claude seems to ignore configured rules, the fix is not guesswork — it's a direct diagnostic command.

**Confirmed command:** `/memory` — lists exactly which CLAUDE.md files (including imports) are currently loaded in the session.

**Situation:** "Claude is ignoring our testing rules" → don't guess/move files around → run `/memory` → immediately reveals whether the expected file is loaded, missing, or misplaced.

**Reliability note (confirmed live):** When asked about rules that don't exist in any loaded CLAUDE.md (e.g., "are testing rules active for this file?" when none are defined), Claude correctly reports it has **no visibility** into such rules rather than fabricating plausible-sounding ones. This is a positive reliability behavior — connects to Domain 5.

---

## 5. Path-Scoped / Conditional Rules (Glob-Based Loading)

**Underlying concept:** Beyond the 3-level scope hierarchy, rules can be conditionally loaded based on **file pattern matching**, so a rule only enters context when a relevant file type is actually being edited.

| Principle | Mechanism |
|---|---|
| **Rules with a filter** | A rule file (e.g., `sql.md`) loads only when a matching file (e.g., `*.sql`) is being edited — not for every file |
| **Load only when needed** | `python.md`, `css.md`, `docker.md` etc. each pattern-scoped to their relevant file type — editing `app.py` loads only `python.md` |
| **Match by type, not folder** | A rule should apply to all files of a type *anywhere* in the project (e.g., `**/*.md`), not just files sitting in one specific directory |
| **Globs beat sub-directory duplication** | One glob-scoped rule (`**/*.md`) replaces copy-pasting the same CLAUDE.md into every subdirectory that needs it |

**Situation:** A markdown-formatting rule lives in `source/pages/CLAUDE.md` — markdown files in `source/blog/` and `source/docs/` never receive it. **Fix:** replace directory-scoped placement with a glob pattern (`**/*.md`) applied project-wide.

**Connects to Domain 5:** This section's own framing — "smaller context is a sharper Claude" — makes explicit that conditional rule-loading is fundamentally a **context-budget management technique**, not just organizational tidiness.

---

## 6. Custom Slash Commands — Team vs. Personal

**Underlying concept:** Same personal/team scoping pattern as CLAUDE.md, applied to custom commands.

| Scope | Location | Shared? |
|---|---|---|
| **Team command** | `.claude/commands/` at project root | Yes — committed |
| **Personal command** | `.claude/commands/` in user-level `~/.claude/` | No — private |

**Situation:** A developer creates `deploy.md` in the **team-level** commands folder, pointing to their own personal test server. A teammate runs `/deploy` and accidentally deploys to the wrong target. **Fix:** move to user-level as `my-deploy.md`, invoked via `/my-deploy`.

---

## 7. Skills (`SKILL.md`) — Front Matter, Scoping, Personal Variants

**Underlying concept:** Skills are on-demand, invokable capabilities (distinct from always-loaded CLAUDE.md), defined via a `SKILL.md` file whose **front matter is a contract** Claude reads before execution.

| Principle | Mechanism |
|---|---|
| **Front matter is the contract** | Specifies allowed tools, argument hints, context — read *before* the skill's instructions execute |
| **Keep noisy tasks separate (context fork)** | A verbose skill (e.g., "scan project") runs in an isolated context; only a summary returns to the main chat |
| **Restrict tools (`allowed-tools`)** | E.g., `allowed-tools: Read` on a summarize skill — prevents accidental edit/delete capability it doesn't need |
| **Argument hints** | Specify required input (e.g., name + role for `create_user`) so Claude doesn't silently fail or guess |
| **Personal variant** | Copy a team skill to user-level `.claude/skills/` under a new name to customize without affecting the team version |

**CLAUDE.md vs. Skills — the key distinction:** CLAUDE.md = **always loaded**, universal/common rules. Skills = **loaded only when invoked**, for specific/occasional tasks. A bloated CLAUDE.md covering occasional workflows (deployment steps, one-off review checklists) should have that content **moved to Skills**, not just split into more always-loaded files.

**Situation:** A `summarize_file` skill has no `allowed-tools` restriction — technically retains edit/delete capability it doesn't need. **Fix:** `allowed-tools: Read`.

---

## 8. Plan Mode vs. Direct Execution

**Underlying concept:** Match process weight to task risk/ambiguity — a Claude Code–specific instantiation of Domain 1's "match strategy to workflow" (chain vs. adapt).

| Principle | When |
|---|---|
| **Plan the complex** | Multiple valid approaches, architectural decisions, many files affected (e.g., "convert project to TypeScript") |
| **Execute the simple** | Small, single-file, obvious-path change (e.g., "change button text Login→Sign In") |
| **Plan prevents rework** | Surfaces intended changes/risks *before* code is written, avoiding mid-implementation breakage |
| **Send an Explorer** | Broad/verbose discovery tasks (e.g., "find all TODO comments") run in isolated context; only a summary returns to main chat |
| **Plan then execute (combined)** | Bounded planning phase → approval → focused execution of the validated plan |

**Anti-pattern — analysis paralysis:** Repeatedly re-planning ("plan it more carefully," "plan again with examples") without ever executing wastes time without producing results. Plan Mode is not unconditionally good — over-planning is its own failure mode.

**Situation:** "Redesign the dashboard" issued in Plan Mode → Claude explains changes and risks *before* any code is written, avoiding a mid-redesign break to existing layout. Contrast: "Change button text from Login to Sign In" issued in Plan Mode → unnecessary overhead; should be direct execution.

---

## 9. Claude Code in CI/CD Pipelines

**Underlying concept:** Running Claude Code non-interactively and reliably inside automated pipelines, where no human is present to respond to prompts.

| Principle | Mechanism |
|---|---|
| **Print, don't prompt** | `-p` flag — runs non-interactively, prints output, exits automatically. Without it, CI jobs can hang waiting for input |
| **Schema, not speech** | `--output-format json` — structured, machine-parseable output for automated downstream processing (e.g., auto-posting review comments) |
| **See the standards in CLAUDE.md** | CI-run Claude needs the same project CLAUDE.md context as interactive sessions, or generated output (tests, code) won't follow team conventions |
| **Fresh eyes review best** | The same session that wrote code is more likely to miss its own mistakes; use a **separate, isolated session** dedicated to review |
| **Don't repeat the review** | Pass forward what was already fixed ("8 issues fixed yesterday, review only new ones") — avoid re-flagging resolved issues every run |
| **Don't repeat the tests** | Provide existing test file as context so Claude generates only new tests for uncovered cases, not duplicates |

**Situation:** A CI step running `claude "review this PR"` hangs and times out. **Root cause:** missing `-p` flag. **Fix:** `claude -p "review this PR" --output-format json`.

**Situation:** The same Claude session both writes and reviews login code in one pass — misses its own bug. **Fix:** dedicate a fresh, separate session purely to review.

---

## 10. Claude Code Setup & Codebase Q&A (Operational)

**Facts to know:**
- Verify install: `claude --version`
- Launch: `claude` (no args) triggers theme selection, login method, folder trust confirmation
- **Three login/billing paths:** (1) Claude subscription account, (2) Anthropic Console (API-based billing), (3) third-party platforms (Bedrock, Foundry, Vertex AI)
- **Folder trust** — required security confirmation before Claude Code acts within a directory
- `/` reveals available slash commands in an active session
- Alternative access: **VS Code extension** (installable via Extensions panel) — CLI remains primary for automation/CI use cases

**Codebase Q&A capability:** Claude Code can answer natural-language questions about an unfamiliar codebase ("what does this project do," "which file saves data to disk") by autonomously using its built-in Grep/Glob/Read tools (Domain 3) — no need to explicitly invoke these tools; Claude's own reasoning decides when to search/read.

---

# 🎯 EXAM-DAY CHEATSHEET — Domain 2

### CLAUDE.md Precedence (confirmed)
```
Directory-level  >  Project-level  >  User-level (inferred, lowest priority)
(highest priority)                    (never committed, broadest scope)
```

### The Personal-vs-Team Scoping Pattern (applies identically across 3 mechanisms)
| Mechanism | Team/Shared Location | Personal Location |
|---|---|---|
| CLAUDE.md | `./CLAUDE.md` (project root, committed) | `~/.claude/CLAUDE.md` |
| Slash Commands | `.claude/commands/` (project) | `.claude/commands/` (user home) |
| Skills | `.claude/skills/` (project) | `.claude/skills/` (user home) |

**Rule of thumb:** `./.claude/` = shared/committed. `~/.claude/` = personal/never committed.

### Where Does This Content Belong? (Decision Tree)
```
Is it a fact/rule that should ALWAYS be active for every session?
├── Applies to just YOU across all projects?        → User-level CLAUDE.md
├── Applies to the WHOLE TEAM on this project?       → Project-level CLAUDE.md
├── Applies only within ONE folder/subtree?          → Directory-level CLAUDE.md
├── Applies only when editing a SPECIFIC FILE TYPE?  → Path/glob-scoped rule file
└── Is it a SPECIFIC, OCCASIONAL TASK (not a rule)?  → Skill (SKILL.md, on-demand only)
```

### Command Quick Reference
| Command | Purpose |
|---|---|
| `claude --version` | Verify installation |
| `claude` | Launch interactive session (theme, login, trust prompts) |
| `/memory` | List currently loaded CLAUDE.md files (diagnose "ignoring my rules") |
| `/` | Reveal available slash commands |
| `@import file.md` | Import a shared rules file into CLAUDE.md |
| `claude -p "..."` | Non-interactive mode — required for CI/CD, prevents hanging |
| `claude -p "..." --output-format json` | Structured, machine-parseable output for automation |

### Plan Mode Decision Rule
```
Multiple valid approaches / many files / architectural decision?  → Plan Mode
Single file / obvious path / trivial change?                       → Direct execution
Verbose/broad discovery task (e.g. "find all TODOs")?               → Isolated Explorer, summary only to main chat
Repeatedly re-planning without ever executing?                     → STOP — analysis paralysis, move to execution
```

### CI/CD Checklist
- [ ] `-p` flag present (non-interactive, prevents hanging)
- [ ] `--output-format json` for anything requiring automated parsing
- [ ] CLAUDE.md context loaded/available so generated output follows team conventions
- [ ] Review runs in a **separate session** from the one that wrote the code
- [ ] Prior findings passed forward — don't re-review already-fixed issues
- [ ] Existing test files passed as context — don't regenerate duplicate tests

### Common Anti-Patterns = Exam Wrong-Answer Signatures
- Team rule accidentally placed in a personal/user-level file → teammates never see it
- Personal preference leaking into a shared/committed CLAUDE.md
- Same rule duplicated across multiple files instead of using `@import`
- One 1000+ line CLAUDE.md instead of split topic files or Skills
- A rule loading for every file type when it should be path/glob-scoped
- Skill with no `allowed-tools` restriction (unnecessary capability)
- Skill with no argument hints (silent failure / guessing)
- Personal customization written directly into a team-level Skill
- Plan Mode used for a trivial one-line change (unnecessary overhead)
- Direct execution used for a large multi-file architectural change (risk of costly rework)
- Repeated re-planning with no execution (analysis paralysis)
- CI job hanging due to missing `-p` flag
- Natural-language (non-JSON) output feeding an automated pipeline step
- Same session writing AND reviewing its own code (misses self-authored bugs)
- Full re-review or full test regeneration on every CI run instead of incremental

---
*Domain 2 complete. Pending: Domain 4 (Prompt Engineering & Structured Output), Domain 5 (Context Management & Reliability).*

## Cross-Domain Meta-Principles (running, updated through Domain 2)
1. **Code/parameter beats prompt** — hooks (D1), `tool_choice` forcing (D3), tool-level validation (D1).
2. **Narrow scope = more reliable** — sub-agent task splitting (D1), per-file/cross-file review passes (D1), tool-set scoping (D3), `allowed-tools` in Skills (D2).
3. **Adapt as you discover / incremental exploration** — task decomposition (D1), dynamic planning (D1), Grep-then-Read exploration (D3).
4. **Good vs. anti-pattern paired teaching** — near-universal across the course; exam distractors are very likely modeled directly on demonstrated anti-patterns.
5. **Personal (local/uncommitted) vs. team (shared/committed) scoping** — now a *generalized Claude Code pattern* across CLAUDE.md, commands, and Skills — all follow `~/.claude/` = personal, `./.claude/` = team.
6. **Isolated execution + summary-only return to main context** — sub-agent context isolation (D1), Skill context-forking (D2), Explorer pattern in Plan Mode (D2).
7. **Match process weight to task risk/ambiguity** — task decomposition chain-vs-adapt (D1), Plan Mode vs. direct execution (D2).
8. **Independent review / fresh-eyes principle** — coordinator gap-checking (D1), per-file/cross-file passes (D1), reviewer-session isolation in CI/CD (D2). One of the most consistently recurring principles in the entire course.
9. **Don't guess on ambiguity — surface it explicitly** — dynamic planning (D1), Plan Mode's investigative phase (D2), "interview before building" prompting technique (Prompt Engineering, pending consolidation).
10. **Conditional/on-demand context loading** — always-loaded CLAUDE.md → path-scoped rules (loaded when file matches) → Skills (loaded when invoked) — a continuum, all serving Domain 5's context-budget goals.

## Gaps Tracker (as of end of Domain 2)
- `/compact` command — referenced in course catalog, never covered in transcripts.
- MCP **prompts** primitive — still uncovered (tools ✅, resources ✅, prompts ❌).
- Exit-code handling conventions for CI/CD failure signaling — natural extension, not explicitly covered.
- Exact three-way CLAUDE.md precedence confirmation for user-level's position (directory > project confirmed; user-level assumed lowest by "most specific wins" logic, not explicitly stated).
- Exact `Edit` tool parameter schema (`old_string`/`new_string` or equivalent) — conceptually covered in Domain 3, not confirmed at schema level.
