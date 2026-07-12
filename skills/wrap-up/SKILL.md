---
name: wrap-up
description: Close a session — review for new memories, get user approval, write approved memories to vault, and create a journal entry. Use at end of any working session.
---

# Session Wrap-Up

Close the current session by extracting durable knowledge and recording a journal entry.

## Configuration

This skill uses `$VAULT_DIR` throughout to mean the root of your Obsidian vault. Resolve it from the `autoMemoryDirectory` setting in `~/.claude/settings.json` (strip the `/memory` suffix) or the `CLAUDE_VAULT_DIR` environment variable. If neither is set, ask the user.

## Step 0: Triage inbox/

Drain the fast-capture `inbox/` folder into durable homes before reviewing the session. This runs every wrap-up. Background on the folder is in the `obsidian-retrieval-article-gap-analysis` project memory.

**List capture items** (everything except the `INBOX.md` index):

```bash
ls "$VAULT_DIR/inbox/" 2>/dev/null | grep -v '^INBOX.md$'
```

**If there are no capture items, skip silently and go to Step 1** — the same "silent unless there is work" stance as the Step 6.5 maintenance checks.

**For each capture file, read it and classify each distinct idea into one route.** A single dated capture note may hold several ideas bound for different homes, so route per idea, not per file:

| Route | Destination | Action |
|---|---|---|
| memory | `memory/<type>/<slug>.md` | Draft a memory (correct type, tags, `originSessionId`). Defer the draft into the Step 2 candidate list. Do NOT write it here. |
| todo | `TODO.md` | An entry under the matching project heading. |
| project | `projects/<name>/...` | An append or edit to the relevant project note. |
| discard | (deleted) | Drop the idea outright. |

**Build a routing table and carry it to Step 2.** Do not write files or prompt the user yet — Step 0 classifies silently; the table and any memory drafts are presented at the Step 2/3 approval gate. Example:

```
Inbox triage (3 items):
 1  r2-cdn-idea.md   -> TODO.md (pasqualo.to)
 2  nas-thought.md   -> memory/project/  (draft shown with candidates)
 3  random-link.md   -> DISCARD
```

Approved routes execute in Step 4 alongside memory writes. A source file is deleted once all its ideas are rehomed; a file with a deferred idea is rewritten with only the unprocessed remainder, so nothing is lost. `INBOX.md` is never read as a capture item and never deleted.

## Step 1: Review the session

Scan the current conversation for anything worth persisting across sessions:

- **Feedback** — corrections given ("don't do X"), approaches confirmed ("yes exactly, keep doing that"), surprising preferences
- **User** — revealed preferences about how to collaborate, background knowledge, working style
- **Project** — decisions made, constraints established, active initiatives, deadlines (convert relative dates to absolute)
- **Reference** — external systems mentioned (Linear projects, dashboards, Slack channels, APIs)

Do NOT save: code patterns derivable from reading the codebase, git history, anything already in CLAUDE.md, ephemeral task details.

**Feedback routing (Rule of Three).** Feedback observations split by origin:

- **Explicit directive.** The user said "remember this", "always X", "never Y", or otherwise directly instructed a standing behavior. Promote immediately: draft a feedback memory in Step 2.
- **Inferred observation.** A correction or pattern noticed without an explicit instruction. Route to the candidates ledger at `$VAULT_DIR/memory/feedback/CANDIDATES.md`. Match against existing rows by underlying meaning, not wording. Propose either incrementing a matching row (Seen +1, append the session id, update the last date) or adding a new row at Seen 1. A row reaching Seen 3 graduates: draft it as a feedback memory in Step 2 and propose removing the row in the same action.
- **Previously retired.** If an observation matches a rule in `memory/feedback/.archive/`, surface it as "previously retired, revive?" instead of adding a ledger row.

Ledger proposals are presented at the Step 3 gate, not written here. If the ledger file is missing, propose creating it with `type: ledger` frontmatter, the explainer header, and an empty `| Observation | Seen | Sessions | First / last | Draft rule |` table (see the Wave 1 spec in the vault: `projects/vault-system/specs/2026-07-12-vault-self-maintenance-wave1-design.md`).

User, project, and reference memories are unaffected by the ledger; they keep the existing Step 2 path.

## Step 1.5: Reflect on standing rules

A third-person pass over the session: which standing feedback rules fired, and did each help or misfire? A rule fired when the session presented a situation the rule governs, whether or not the rule was followed.

Identify fired rules from the feedback memories recalled into this session's context plus any read during the session. For each fired rule, propose ONE targeted edit:

- **Counter bump:** `helped: N+1` or `misfired: N+1`, plus `last_fired: <session start date>`. Add absent fields at the frontmatter root (flat YAML, never nested under `metadata:`). An absent counter reads as 0.
- **Refinement:** a one-line change to the rule text when the session showed the wording is off.
- **Retirement:** move the file to `$VAULT_DIR/memory/feedback/.archive/` when the rule misfires repeatedly or no longer applies.

Never rewrite a note wholesale. A proposal touches frontmatter counters or a single line, nothing more. If no standing rule fired, state "No standing rules fired this session" and move on.

These proposals are presented in the Step 2 message and share the Step 3 approval gate.

**Escalation path (documented, not built):** if wrap-ups start missing fired rules or making sloppy counter calls, replace this prose pass with a per-rule verifier workflow following the prose-hygiene pattern (one checker agent per fired rule, returning structured bump/refine/retire proposals).

## Step 2: Draft candidates

Format each candidate in compact B:

```
Type: feedback | user | project | reference
Proposed file: vault/memory/<type>/<slug>.md

**Rule/Fact:** [one or two sentences — the actionable rule or durable fact]
**Why:** [one sentence — the reason]
**Apply:** [one sentence — when this kicks in] ← omit for project/reference
```

**Zero `[[...]]` in memory files. None. Anywhere in the body.** No `**Links:**` line, no "Related:" bullet, no "see [[other-memory]]" inline, no link to the project hub or to spec/plan files. A single outbound wikilink pulls the memory onto the Obsidian graph and defeats the off-graph design (per `memory-and-journals-off-graph`).

Cross-references go in plain prose with backticks, e.g. "see the `shop-admin-migration-pending` memory" — never `[[wikilinks]]`. This applies even when two memories are obviously related (e.g. a follow-up issue and its parent) — backticked prose only.

The only exceptions are the `USER.md` and `REFERENCE.md` hubs, which DO link to their memories (graph clusters by design). The memory files themselves still have no outbound links.

Present ALL drafts in a single message, together with the inbox routing table from Step 0 (inbox memory items appear here as drafts; todo/project/discard routes appear as table rows), the Step 1 ledger proposals, and the Step 1.5 reflector proposals. Do not write any files yet.

**Codify gate.** This step must end in one of two states:

1. At least one action on the table: a memory draft, a ledger row add/increment/graduation, or a reflector proposal.
2. The explicit line "No durable lesson this session" plus a one-sentence justification.

Silent skipping is not an option; extraction is a required output of every wrap-up. In state 2, skip to Step 5 after stating the line.

## Step 3: Get approval

Wait for the user to respond. They may:
- Approve entries as-is
- Edit individual entries inline
- Reject entries (just skip them)

Only proceed with explicitly approved entries. This single approval also covers the Step 0 inbox routing table — memory, todo, project, and discard routes alike.

## Step 4: Write approved memories

For each approved entry, write to `$VAULT_DIR/memory/<type>/<slug>.md`:

**Every memory file MUST carry a root-level `originSessionId`.** Without it, the harness auto-memory adopts the file at the next turn boundary, re-nesting it under a `metadata:` block and stamping the session itself — which drops the file out of every tag/type-driven `.base` view (see the `memory-frontmatter-must-be-flat` rule). Read the current id from `~/.claude/session-start-state.json` (`.session_id`); if it is empty there, derive it from the active transcript directory name under `~/.claude/projects/`.

```markdown
---
type: <feedback | user | project | reference>
tags: [memory/<type>, <project-key>, <topic>]
originSessionId: <current session id>
---
# <Title>

**Rule:** <...>   ← use "Fact:" for project/reference types

**Why:** <...>

**Apply:** <...>   ← omit for project/reference
```

No `**Links:**` line — memory files are off-graph. Tag the file with the relevant project-key (hyphen form, e.g. `my-app`, `backend`, `infra`) so it surfaces in the right `memory.base`.

Subfolder routing and hub updates differ by type:

**feedback** → `memory/feedback/<slug>.md`
No hub update — `feedback.base` indexes by folder; per-project `memory.base` views surface tagged ones. Off-graph by design.

**user** → `memory/user/<slug>.md`
Then append under the matching section in `memory/user/USER.md`:
- Working-style memories (execution preferences, iteration style, biases, design workflow) → `## Working Style`
- Identity / environment memories (git config, machine quirks, OS / shell setup) → `## Identity & Environment`

```
- [[memory/user/<slug>]] — one-line hook
```

USER.md is a hub that intentionally wikilinks user memories — the resulting graph cluster is by design. **If appending pushes a section to ≥5 entries, surface that to the user** — that's the extraction trigger to promote the section into its own sub-hub at `memory/user/<theme>.md` (vault-system pattern), with USER.md keeping a single link line for the section.

**project** → `memory/project/<slug>.md`
No hub update. Project memories are surfaced via the project's `memory.base` (tag-driven Bases query); the project hub does NOT enumerate them with wikilinks. There is no global `PROJECT.md` hub — MEMORY.md's `## Projects` section points directly at `projects/projects.md`.

**reference** → `memory/reference/<slug>.md`
Then append under the matching kind-based sub-hub in `memory/reference/`:
- Credentials, test accounts, doc-access tricks → `memory/reference/CREDENTIALS.md`
- MCP servers → `memory/reference/MCP.md`
- CLI tools, sync scripts, local platform integrations → `memory/reference/TOOLS.md`

```
- [[memory/reference/<slug>]] — one-line hook
```

The three sub-hubs (CREDENTIALS, MCP, TOOLS) and the top-level REFERENCE.md hub intentionally wikilink reference memories — that's how they appear on the graph. The memory file itself still has no outbound links. If a reference doesn't fit any of the three buckets cleanly, default to TOOLS and mention the categorization decision in the commit message.

**Approved inbox routes (from Step 0) also execute here.** Write `TODO.md` entries and project-note edits to their targets. Delete discarded ideas and any source file whose ideas were all rehomed. If a multi-idea capture file has a deferred idea, rewrite the file with only the unprocessed remainder rather than deleting it. `INBOX.md` is never deleted.

## Vault structure reference

Follow these rules whenever creating or modifying vault files outside of memory writing above.

**Journal entries** (`journal/YYYY-MM-DD-<slug>.md`):
- Frontmatter: `date`, `project` (plain text slug — single-valued, lowercase, hyphen-form), `tags: [journal]`
- No outbound wikilinks anywhere in the entry — journals are off-graph (per `memory-and-journals-off-graph`). Cross-references in plain prose with backticks: "see the `my-project-cleanup-todos` project memory", never `[[wikilinks]]`. Sessions are surfaced via `journal/sessions.base` which filters by folder + groups by `project` frontmatter.
- Do NOT add `project_link:` frontmatter
- Do NOT create per-project journal hub files

**New project overview** (`projects/<name>/<name>.md`):
- Required sections in order: Key decisions, Gotchas, Changelog, Memory
- Include `aliases:` in frontmatter with 2-4 natural synonyms (conversation names, URLs, abbreviations)
- Memory section points at the project's `memory.base` (tag-driven Bases query) — do NOT enumerate individual memory files with wikilinks
- Do NOT add a Sessions section — sessions are navigated via MEMORY → JOURNALS → sessions.base
- Plans & specs are NOT enumerated in the hub at all — active ones live in `<project>/specs/` and `<project>/plans/` (folder browse), shipped ones move to `<project>/specs/archive/` and `<project>/plans/archive/` and surface via `archive.base`
- Spec/plan files are MOVED to `archive/` subdirs at branch close (NOT deleted) — see `vault-archive-implemented-specs-plans`. Add the archive folder to `.obsidian/app.json` `userIgnoreFilters` so it stays off the graph (folder paths only, no globs).
- After creating the hub, add it to `projects/projects.md` under `## Active` — or, if it belongs to a family, to the family's group hub and keep it OFF `projects.md` so the family hub stays the single entry point
- Do NOT create a `projects/<name>/journals.md` hub file

**Graph edge discipline** (see `memory-and-journals-off-graph`, `vault-graph-edge-principles`, `vault-system-keeps-graph-edges`):
- Memory files (feedback, project, reference, user) and journal entries carry NO outbound wikilinks anywhere — not in a Links field, not in "Related" or "See also" tails, not inline in prose. Cross-references go in backticked plain prose.
- Code-project plans and specs (under `projects/<name>/{plans,specs}/`) MUST NOT inline `[[memory/...]]` either. Even an inbound link from a plan to a memory pulls the memory onto the graph as an island. Backticked prose only, never `[[wikilinks]]`.
- The exceptions are the USER.md and REFERENCE.md hubs, which intentionally wikilink TO their memories (graph clusters by design). The memory files themselves still have no outbound links.
- Meta/infrastructure projects are a special case: their memories can be wikilinked from thematic sub-hubs within the project to stay visible. The memory file itself still has no outbound links.
- Hub section headers use plain text — no wikilinks in headers
- MEMORY.md excludes templates, bases, and canvases
- When registering a new project hub: add `[[projects/<name>/<name>]]` to `projects/projects.md` under `## Active`, OR (if it's part of a family) under the family hub's sub-projects/children list and keep it off the top-level `projects.md`. The hub will not show up in the graph cluster until one of those happens.

## Step 5: Write journal entry

Determine the **session start date** (YYYY-MM-DD) — not necessarily today's calendar date, since sessions that begin late at night may cross midnight before wrap-up runs. Use the date the session started, not the date wrap-up is invoked. Check the CMEM context header or the first user message timestamp if uncertain.

**First, list all same-day entries:** `ls $VAULT_DIR/journal/YYYY-MM-DD-*.md`

**Slug choice — always topic-specific:** The slug is the primary differentiator in the Bases table when multiple sessions share the same date and project. Never use generic slugs like `session`, `work`, or `session-2`. Pick words that describe *what this session actually did* (e.g. `shop-libs`, `palette-design`, `auth-bugfix`).

**Handling the stub (from SessionEnd hook):**
- If a stub exists and it's the only same-day file: fill it in. If the stub filename is generic (e.g. `YYYY-MM-DD-session.md`), rename it to the topic slug before writing: `mv YYYY-MM-DD-session.md YYYY-MM-DD-<topic>.md`.
- If other fully-written same-day entries already exist: this stub belongs to the current session — fill it in. If the stub filename is generic, rename it.
- If no file exists: create `$VAULT_DIR/journal/YYYY-MM-DD-<slug>.md`.

**Same-day cleanup:** If any existing same-day entries have generic slugs, suggest renaming them to match their content before closing.

Before writing, ask the user: **"Which project was the focus of this session?"** Use their answer as the `project` field.

Content:
```markdown
---
date: YYYY-MM-DD
project: <project-slug>
tags: [journal]
---
# YYYY-MM-DD: <topic>

## Summary
[One paragraph: what was worked on, what changed, what the session accomplished]

## Decisions
- [Each significant decision made this session]

## Open threads
- [Things left unfinished, questions unresolved, work deferred]

## Next session
- [Concrete things to pick up next time]
```

**Journal title format:** `# YYYY-MM-DD: <topic>` uses a colon, not an em dash. Em dashes violate the CLAUDE.md prose-hygiene rule and are caught by Step 5.5.

## Step 5.5: Prose-hygiene gate

Before the journal gets committed, lint it against the CLAUDE.md prose-hygiene rules. Run the `prose-hygiene` workflow on the journal entry you just wrote:

```
Workflow({ scriptPath: "~/.claude/skills/prose-hygiene/prose-hygiene.workflow.js", args: "<absolute path to the journal entry written in Step 5>" })
```

It returns `{ violations, filesChecked }`. Each violation is `{ file, rule, line, quote, suggestedFix }` covering em dashes, throat-clearing, meta-commentary, vague declaratives, false agency, filler words, and needless adverbs — the AI tells a single drafting pass misses.

For each confirmed violation, apply the `suggestedFix` to the journal entry yourself. You authored this prose, so tightening it is in scope — this is not unattended editing of user content. Use judgment: if a suggested rewrite changes meaning, prefer a fix that satisfies the rule while preserving intent. Re-read the file after edits if needed.

If `violations` is empty, the journal is clean — continue. Skip this step only if the user explicitly says to skip the prose check, or if `~/.claude/skills/prose-hygiene/` does not exist.

## Step 6: Write changelog

Read the session anchor:

```bash
cat ~/.claude/session-start-state.json
```

This gives you `cwd`, `head_commit`, and `timestamp` for the session that just ended.

If the file doesn't exist, ask the user:
> "I don't have a session anchor — how many recent commits should I include in the changelog? (e.g. 3, 5, or all)"

Then use `git log -N --oneline` (or `git log --oneline` for "all") and skip the path-selection logic below.

**Determine which commits belong to this session** using one of two paths:

- If `head_commit` is non-empty AND exists in the current repo (`git cat-file -t <head_commit>` outputs `commit`) → use:
  ```bash
  git log <head_commit>..HEAD --oneline
  ```
- Otherwise (empty `head_commit`, or commit not found, or CWD is not a git repo) → use:
  ```bash
  git log --since="<timestamp>" --oneline
  ```

**If the result is empty (no commits this session) → skip this step entirely. Write nothing.**

**Group commits by conventional prefix:**

| Prefix | Changelog header |
|---|---|
| `feat:` | `### Added` |
| `fix:` | `### Fixed` |
| `refactor:`, `perf:` | `### Changed` |
| `chore:`, `docs:`, `test:`, `style:`, `build:`, or no prefix | `### Other` |

Only include headers that have at least one commit.

**Write to the vault changelog only — repo CHANGELOG.md files are no longer maintained.**

Determine the target file by project:

- If the project belongs to a **family** (a group of related projects sharing a single changelog), write to the family's consolidated changelog: `$VAULT_DIR/projects/<family>/changelog.md`
- Otherwise, write to `$VAULT_DIR/projects/<project>/changelog.md` (create if missing)

**Format the entry** — heading carries a `(<project>)` tag so family changelogs stay scannable across sub-projects. Cross-cutting sessions use `(<projectA> + <projectB>)`. `session-slug` is the slug portion of the Step 5 journal filename (e.g. journal `2026-04-23-about-redesign.md` → slug `about-redesign`).

```markdown
## YYYY-MM-DD — <session-slug> (<project>)

<Copy the Summary paragraph from the journal entry written in Step 5 verbatim>

### Added
- feat: short description (abc1234)

### Fixed
- fix: short description (def5678)

### Changed
- refactor: short description (ghi9012)

### Other
- chore: short description (jkl3456)
```

**For cross-cutting family entries**, group commits by which sub-project they landed in when relevant — use sub-headers `### Added (sub-project-a)` / `### Added (sub-project-b)` etc. so the work stays distinguishable. Single-project entries use plain `### Added` etc.

**Prepend** the new entry below the file's `# Changelog` header (most-recent-first). If the file doesn't exist, create it with:

```markdown
# <project> — Changelog
```

(or `# website — Changelog` for the consolidated file).

**Do NOT commit** in the project repo — the repo CHANGELOG.md is no longer maintained. The vault commit happens in Step 8 below.

## Step 6.5: Maintenance checks

Two sub-steps that run silently unless they find work to do.

### 6.5a: Log skill usage

Record which skills were invoked this session to `~/.claude/skill-usage.json`.

1. Scan the conversation for Skill tool calls. Each call's `skill` parameter identifies the skill (e.g., `superpowers:brainstorming`). Collect the distinct set.
2. Read `~/.claude/skill-usage.json`. If the file doesn't exist, start with `{}`.
3. For each distinct skill invoked:
   - If the key exists: set `last_used` to today (YYYY-MM-DD), increment `count` by 1.
   - If the key doesn't exist: create it with `last_used` set to today, `count` set to 1.
4. Write the file back.

```json
{
  "superpowers:brainstorming": { "last_used": "2026-05-26", "count": 14 },
  "superpowers:verify-setup": { "last_used": "2026-05-25", "count": 3 }
}
```

No user interaction. No output unless the write fails.

### 6.5b: Memory consolidation

Check vault memory metrics. If any threshold is exceeded, consolidate automatically and report what changed. If all metrics are healthy, skip silently.

**Check metrics:**

```bash
# MEMORY.md line count
wc -l < $VAULT_DIR/memory/MEMORY.md

# feedback/ file count (exclude .base files)
find $VAULT_DIR/memory/feedback -name "*.md" -not -name "*.base" | wc -l

# project/ file count (exclude .base files and .archive/)
find $VAULT_DIR/memory/project -name "*.md" -not -name "*.base" -not -path "*/.archive/*" | wc -l
```

**Thresholds:**

| Metric | Warn | Consolidate |
|--------|------|-------------|
| MEMORY.md line count | 150 | 180 |
| feedback/ file count | — | 100 |
| project/ file count | — | 50 |

**If no thresholds exceeded:** skip silently, move to Step 7.

**If MEMORY.md exceeds 150 lines:**
1. Read MEMORY.md
2. Remove completed/struck-through items (lines containing `~~`)
3. Tighten entry descriptions to under 150 chars each
4. If still over 180 lines after tightening, flag for manual review instead of auto-consolidating further
5. Write the file back

**If feedback/ exceeds 100 files:**
1. Read all feedback file frontmatter to build a tag index
2. Group files that share at least one project tag AND cover related topics (use filenames and first-line titles to assess relatedness)
3. For each group of 3+ related files:
   a. Create a new merged file at `memory/feedback/<shared-topic>.md`
   b. Frontmatter tags: union of all source files' tags
   c. Body: each original rule as a bullet, preserving the most specific "Why" and "How to apply" from each source
   d. If two sources contradict, keep the more recent one (by file modification time)
   e. Delete the original files
   f. Update any MEMORY.md references that pointed to deleted files
4. Report: "Merged N feedback files into M consolidated files: [list new filenames]"

**If project/ exceeds 50 files:**
1. Read all project memory files
2. Identify files marked as completed/shipped: body contains `~~` strikethrough wrapping the main content, or contains "shipped" / "completed" as a status keyword
3. Create `memory/project/.archive/` if it doesn't exist
4. Move completed files to `memory/project/.archive/`
5. Remove any MEMORY.md index entries that referenced the archived files
6. Report: "Archived N completed project memories to .archive/"

**After any consolidation:** all changes are unstaged, visible in the Step 8 commit diff. The Step 8 commit message should include a line describing what was consolidated.

## Step 7: Save graphify result

**Skip this step entirely if `graphify-out/` does not exist in the project root.** Check with:

```bash
ls <project-cwd>/graphify-out/ 2>/dev/null
```

If the directory exists, **save by default — do not ask first.** Review the session for a *graph-worthy discovery*: an investigation or Q&A that answered a "why does X / how does Y work" question about this project's code, surfaced a non-obvious gotcha or root cause, or reached a design conclusion a future `graphify query` should recall. The completion criterion is concrete: a discovery is graph-worthy when it is (a) about the graphed project's code, and (b) not already fully captured in a memory or journal entry in a form the graph would merely duplicate.

For each graph-worthy discovery, synthesize the question and a one-sentence answer yourself from the session, then save it:

```bash
python3 -m graphify save-result \
  --question "<question>" \
  --answer "<answer>" \
  --type query
```

Report each saved Q&A in the Done summary so the user can correct or delete it.

**Prompt only when relevance is genuinely ambiguous** — a borderline discovery where you cannot tell whether it is graph-worthy. Then, and only then, ask:

> "This came up this session: <one-line>. Worth saving to the graph? (yes / skip)"

**Skip silently if** the session produced no graph-worthy discovery (routine edits, no investigation, or everything is already captured in memory/journal). Saving nothing is the correct outcome for a session with no codebase insight; do not invent a Q&A to fill the step.

## Step 8: Commit vault changes

After all storage writes (memory, journal, changelog, TODO hub updates, and any other vault edits made during the session), commit the resulting vault changes so the wrap-up output is durable. The vault is local-only per `vault-no-remote-push` — this is a local commit, not a push.

**Stage selectively.** Stage ONLY the files you wrote or modified during this wrap-up + the session itself. Use specific paths, not `git add -A` / `git add .` — those would sweep up unrelated dirty state from other sources (e.g. SessionEnd hook stub appends, in-progress edits in other projects). Typical wrap-up paths:

- `memory/<type>/<slug>.md` — any new memory files
- `memory/MEMORY.md` / `memory/user/USER.md` / `memory/reference/REFERENCE.md` — any hub updates
- `TODO.md` or other top-level hubs touched
- `inbox/<file>` removals (rehomed or discarded capture files) and `inbox/INBOX.md` if its protocol text changed
- `journal/YYYY-MM-DD-<slug>.md` — the session entry
- `projects/<name>/changelog.md` or `projects/website/changelog.md` — the changelog prepend
- `projects/<name>/<name>.md` — if the project hub was updated
- Any `projects/<name>/{specs,plans,design-handoffs}/...` files written or moved during the session

**Run:**

```bash
git -C $VAULT_DIR status --short
# verify the staged set looks right, then:
git -C $VAULT_DIR add <paths...>
git -C $VAULT_DIR commit -m "$(cat <<'EOF'
<project>: <one-line summary of session>

<2-4 line body summarizing what was written: which memories, journal slug,
hub updates, and any other artifacts. Match the level of detail in the
session's project-repo commits if any landed there.>
EOF
)"
# Do NOT hardcode a Co-Authored-By trailer here. It is appended automatically
# per the harness attribution settings, which track the current model version.
# A literal trailer (e.g. a pinned "Opus 4.X") goes stale at each model bump.
```

**Skip this step if** `git status --short` shows no staged or unstaged changes after Steps 1–7 — there's nothing to commit. Also skip if the only dirty files are unrelated to the session (e.g. only the SessionEnd hook's stub append on someone else's journal); those belong to whoever modified them, not to wrap-up.

If a pre-commit hook fails, fix the underlying issue and re-stage + re-commit (NEVER `--amend` here, NEVER `--no-verify`).

## Done

Report:
- How many memory entries were written (and their paths)
- Inbox triage result: `N rehomed (M memory, P todo, Q project), R discarded, S deferred`, or "inbox empty, skipped"
- Journal entry path
- Changelog path written (vault only), or "no commits this session"
- Graphify Q&A saved (the question(s) auto-saved this session), or "skipped" if no graph or no graph-worthy discovery
- Vault commit hash + paths included, or "no vault changes to commit"

