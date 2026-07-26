---
name: spelflow-loader
description: "Creates and manages Spelflow Tracker entities on spelflow.ru — projects, milestones, and issues — via the REST API. Also bridges to the pm-skills marketplace: detects when the user has a raw idea instead of ready backlog items, offers to install missing pm-skills plugins, and recommends which PM skill to run for the task at hand. Not a sprint-markdown parser first and foremost — sprint markdown and WWA backlog blocks are just two of its input formats."
trigger: /spelflow-loader
---

# /spelflow-loader — Spelflow Creation & PM Pipeline Bridge

**What this skill actually is:** a thin, general client for the Spelflow Tracker REST API (spelflow.ru) — it creates and reads projects, milestones, and issues. Parsing a specific input file format (a sprint markdown file, a set of WWA backlog blocks) is just how *data gets in* — it is not the point of the skill. Don't reach for this skill only when there's a `sprint.md` file to load; reach for it any time the task is "put this in Spelflow" or "what's in Spelflow right now," regardless of what form the source material is in.

> Renamed from `/llap` on 2026-07-23; reframed from "sprint loader" to "Spelflow creation + PM pipeline bridge" on 2026-07-23.

---

## Usage

```
/spelflow-loader                                   # figure out what the user needs from conversation context
/spelflow-loader <path-to-sprint.md>               # load a Day-N sprint file
/spelflow-loader <path-to-wwa-backlog.md>          # load a file of WWA (Title/Why/What/Acceptance Criteria) blocks
/spelflow-loader --dry-run                         # show what would be created, make no API calls
/spelflow-loader list projects|issues|milestones [--project X]   # read-only queries, no file needed
/spelflow-loader create project|milestone          # create a single entity interactively, no file needed
```

If the user's request doesn't match a known input format and doesn't look like a direct API operation (e.g. they just describe a large, unrefined idea in chat) — go to **Step 0** before anything else.

---

## Step 0 — Route the request

Before touching tokens or the API, figure out which of these the user actually needs:

| What the user gave you | What to do |
|---|---|
| A path to a file, or pasted content, matching the Sprint Markdown adapter (`### Day N` headings) | Go to Step 2 (token), then the **Sprint Markdown** input adapter |
| A path to a file, or pasted content, matching the WWA Blocks adapter (`**Title:**` / `**Why:**` / `**What:**` / `**Acceptance Criteria:**`) | Go to Step 2 (token), then the **WWA Blocks** input adapter |
| A direct request — "create a project called X", "what issues are in TSK", "create a milestone for this sprint" | Go to Step 2 (token), then **Core Capabilities** directly — no file needed |
| A raw, unrefined idea or a large/complex feature description with no structure yet (e.g. "I want to rework how Rabbit handles voice + task creation + model switching") | **Do not try to turn this into issues yourself.** Go to **Step 1 — PM Skills Bridge** first |

---

## Step 1 — PM Skills Bridge (only when the input is an unrefined idea)

The pm-skills marketplace (`phuryn/pm-skills` on GitHub) has the actual task-construction skills — `spelflow-loader` does not duplicate them. Check what's installed, offer to install what's missing, and recommend the right entry point.

**Check what's installed:**
```bash
cat "C:/Users/user/.claude/plugins/installed_plugins.json" | grep -i "pm-execution\|pm-ai-shipping\|pm-product-strategy\|pm-product-discovery"
```
(Or run `claude plugin list` if available in the session.)

**If the `pm-skills` marketplace itself isn't registered** (check `C:/Users/user/.claude/plugins/known_marketplaces.json` for a `"pm-skills"` key), tell the user:
> "The pm-skills marketplace isn't added yet. Run:
> ```
> claude marketplace add phuryn/pm-skills
> ```
> then install the plugin you need (see below)."

**If the marketplace is registered but the needed plugin isn't installed**, tell the user which command to run, e.g.:
```
claude plugin install pm-execution@pm-skills
```

**Which plugin/skill to recommend, based on what the user described:**

| Situation | Recommend |
|---|---|
| Default for a technical/engineering task (most common case here) | `pm-execution` plugin → `/write-prd` first, **then `wwa` format** for backlog items (`/write-stories wwa [feature]`) — WWA is the most technical-friendly of the three story formats: no forced "as a user" framing, and it explicitly keeps the "What" section short and non-prescriptive rather than a full spec |
| The feature is really several distinct situational triggers (e.g. "when X happens, the system should Y") | `/write-stories job [feature]` — job-stories format fits situational/JTBD framing better than a generic user role |
| The feature is genuinely role-driven (different behavior per user type/permission level) | `/write-stories user [feature]` — classic user-story format, and the only one of the three that explicitly names INVEST |
| Task is large/risky and touches core behavior before you commit to building it | Run `/pre-mortem` or `/red-team-prd` on the PRD before writing backlog items |
| Multiple features need sequencing against team capacity | `/sprint plan` after the backlog items exist |
| Pre-launch codebase audit (security, permissions, test coverage) — different plugin | `pm-ai-shipping` (not currently installed — offer `claude plugin install pm-ai-shipping@pm-skills` if this is what's needed) |

**Recommended sequence for one epic (what to tell the user):**
```
1. /write-prd [epic description]                    → PRD-[name].md (Key Features = your epic's sub-features)
2. /pre-mortem [paste the PRD]   (optional, if risky) → PreMortem-[name]-[date].md
3. /write-stories wwa [one Key Feature at a time]    → WWA blocks, one run per feature
4. Collect the WWA blocks into one file
5. /spelflow-loader <that file>                      → pushes them into Spelflow as issues
```

Once the user has run steps 1–4 and has a WWA file (or is pasting WWA blocks directly), proceed to **Step 2**.

---

## Step 2 — Get the Personal API Token (SPELFLOW_TOKEN)

The token is **not** stored as a Windows environment variable — it's tied to something visible/portable instead. Look for it in this order, stop at the first hit:

1. **`SPELFLOW_TOKEN` in a `.env` file in the current project's root directory.** Read it with the `Read` tool if `.env` exists; look for a line `SPELFLOW_TOKEN=hly_...`.
2. **`SPELFLOW_TOKEN` in a global file**: `C:\Users\user\.spelflow\token` (plain text, just the token, no key= prefix). Use this when no project `.env` has it.
3. **Already provided earlier in this conversation.**
4. **None found — ask the user:**
   > "Provide your Personal API Token from Spelflow (Settings → API Tokens). Format: `hly_...`"
   >
   > Then ask where to save it for next time:
   > "Save this for future runs? (1) project `.env` here, (2) global file `~/.spelflow/token` for use across all projects, (3) don't save — ask again next time"

**If (1) project `.env`:** append `SPELFLOW_TOKEN=hly_...` to `.env` in the project root (create it if missing). **Check `.gitignore` for a `.env` entry** — if missing, add it and tell the user: "Added `.env` to `.gitignore` — this token must never be committed." Skip the `.gitignore` step silently if the directory isn't a git repo.

**If (2) global file:** write the raw token (no `KEY=` prefix) to `C:\Users\user\.spelflow\token`, creating the `.spelflow` directory if needed. This lives outside any git repo by construction.

**If (3):** don't persist; use it for this session only.

**Never** write the token into `CLAUDE.md`, any wiki page, any `PM/` file, or anything else that gets read into context or committed to git. Don't print the token back to the user or into any log/summary output.

---

## Core Capabilities (the actual point of this skill)

Everything below works standalone — no sprint file, no WWA file, just a direct request ("create a project," "what's in TSK right now," "add a milestone").

### Base
```
https://app.spelflow.ru/account/api/v1
```
> **Domain note**: the REST API lives under `app.spelflow.ru`, not `spelflow.ru` — the old domain 308-redirects and breaks curl.

### Auth header
```
Authorization: Bearer hly_...
```

### Workspaces
```
GET  /workspaces
→ [{ id, url, name, role }]     role: owner | maintainer | user | guest

POST /workspaces
Body: { name, configuration?: { withDemoContent?: boolean } }
→ 201 { id, url, name }
→ 400 { error: "Missing required field: name" }  |  { error: "Invalid workspace name." }
→ 422 { error: "Account must have a confirmed email address." }                          (AccountNotConfirmed)
→ 422 { error: "workspace_limit_reached", message, limit, current, plan }                (per-account quota — free plan: 5 owned workspaces, pro: unlimited)
→ 429 { error: "Rate limit exceeded. Try again later." }                                 (>5 workspace creates within 60s for the same account)
```
Confirmed against `server/account-service/src/index.ts:637-691` (spelflow-app repo) 2026-07-23 — corrects an earlier version of this skill that claimed no create-workspace endpoint existed. The caller becomes the workspace's Owner automatically. Security is already fully handled server-side: same Bearer-token auth as every other endpoint, email-confirmation requirement, a real per-account quota, and in-memory rate limiting — no extra client-side confirmation step needed before calling this.

### Projects
```
GET  /workspaces/{ws}/projects                        → [{ id, name, identifier }]
POST /workspaces/{ws}/projects
Body: { name, identifier, description?, private?, autoJoin? }
→ 201 { id, name, identifier, description, private, archived }
→ 409 if identifier already exists
```
`identifier` = short uppercase prefix (e.g. "TSK" → issues become `TSK-1`, `TSK-2`...). Project type, default status (`Backlog`), and time-report settings are fixed to Spelflow UI defaults — not customizable via this endpoint.

### Milestones
```
GET  /workspaces/{ws}/projects/{project}/milestones   → [{ id, label, description, status, statusCode, targetDate }]
POST /workspaces/{ws}/projects/{project}/milestones
Body: { label, status?, targetDate?, description? }    status: Planned | InProgress | Completed | Canceled (default Planned)
→ 201 { id, label, description, status, statusCode, targetDate, project }
```

### Issues
```
GET   /workspaces/{ws}/projects/{prj}/issues                → [{ id, identifier, title, description, status, priority, dueDate, milestone, assignee, estimation }]
GET   /workspaces/{ws}/projects/{prj}/issues/{identifier}   → single issue (identifier e.g. TSK-5, not the internal _id)
POST  /workspaces/{ws}/issues
Body: { title, project, priority?, description?, dueDate?, status?, milestone?, assignee?, estimation? }
→ 201 { id, identifier, project, title, description, status, priority, dueDate, milestone, assignee, estimation }
PATCH /workspaces/{ws}/projects/{prj}/issues/{identifier}
Body (all optional): { title, status, priority, assignee, dueDate, milestone, estimation, description }
→ 200 { ok: true, identifier }
```
`priority`: `urgent | high | medium | low`. `description` is markdown, rendered with interactive checklists in Spelflow.

### Comments (optional — depends on whether the person wants a progress trail)
```
GET  /workspaces/{ws}/projects/{prj}/issues/{identifier}/comments   → [{ id, text, author, createdOn }], oldest first
POST /workspaces/{ws}/projects/{prj}/issues/{identifier}/comments
Body: { text }
→ 201 { id, text, author, createdOn }
```
Posting comments (announcing you're starting, asking a clarifying question, summarizing what you did) is available but not automatic — whether to use it is up to the person you're working with, not a default behavior. Ask once whether they want progress reported into the issue's comments; remember the answer for the rest of that task/session and don't ask again. Only bring this up on your own if the user's request already hints they want it (e.g. "let them know when you start", "leave a note when you're done") — otherwise don't mention it unprompted.

### Members
```
GET /workspaces/{ws}/members   → [{ id, name }]
```

### Error codes
| Code | Meaning |
|------|---------|
| 401 | Token invalid/expired/revoked |
| 404 | Workspace, project, or issue not found |
| 400 | Missing required field (title/project for issues, label for milestones, name/identifier for projects) |
| 409 | Project identifier already exists |
| 500 | Account has no verified social IDs |

### Cyrillic / non-ASCII encoding — critical for every write

Passing JSON with Cyrillic text directly via `curl -d '{...}'` on Windows corrupts multi-byte UTF-8 into `U+FFFD` (`◆◆◆◆◆◆`) — silent, curl returns 201, corruption only visible when you open the issue in Spelflow.

**Always write the JSON body to a temp file (UTF-8, via the `Write` tool) and send with `--data-binary @file`**:
```bash
curl -s -X POST \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary @/path/to/payload.json \
  https://app.spelflow.ru/account/api/v1/workspaces/WORKSPACE_SLUG/issues
```
Plain ASCII-only payloads are safe with inline `-d`, but using the file approach everywhere is simpler and consistent.

**Verify before sending**: `Read` the temp file back and confirm Cyrillic renders correctly (not `?`/`�`) — the Write/Read round-trip never corrupts text.

**Detect corruption after sending**: check `title`/`description` in the 201 response immediately. If you see `◆`, **stop the batch** — it's systemic, every subsequent issue will be corrupted too. Switch to the file method and re-verify with a test issue before continuing.

**Fix a corrupted issue in place**: PATCH with a corrected temp-file payload — no need to delete/recreate. Delete temp files after use.

---

## Input Adapter: Sprint Markdown

Use when the input has `### Day N — Title` headings (with optional `**Focus**`, `**Assignee**`, checkboxes).

**Structure mapping:**
| Sprint element | Spelflow element |
|---|---|
| H3 heading (`### Day N — Title`) | Issue title (strip `Day N — ` prefix) |
| **Focus**: line | Issue description first line |
| **Assignee**: line | `assignee` field |
| Checkbox items | Checklist in issue description |
| H2 heading (`## Week N`) | Ignored (grouping only) |
| Sprint success criteria table | Single summary issue: "Sprint Success Criteria" |
| "What is NOT in this sprint" | Single issue: "Sprint Deferred Items" (priority: low) |

**Title cleanup**: strip leading `Day N — ` / `Day N–M — `, keep everything after the dash.

**Description format**: `{Focus line}\n\n---\n\n{checklist items joined with \n}`.

**Priority from Focus keywords:**
| Keyword | Priority |
|---|---|
| `blocking`, `launch-blocking`, `T0`, `T1`, `критич` | `urgent` |
| `pre-mortem`, `T2`, `T3`, `required before` | `high` |
| `analytics`, `dogfooding`, `onboarding` | `medium` |
| everything else | `medium` |

**Status from checkboxes:**
| Checkbox state | Status |
|---|---|
| All `[x]` | `Won` (or `Done`) |
| Mix | `In Progress` |
| All `[ ]` or none | `Todo` |

**Assignee resolution** (only if `Assignee:` lines are present): `GET /workspaces/{ws}/members`, fuzzy-match on the **start** of a person's name (case-insensitive). No match → create without assignee, warn: "⚠ Could not resolve assignee 'Имя'." 404 on the endpoint → warn and skip assignees entirely.

**Skip**: H2 week headings, a "Key commands reference" section, empty sections.

**Due date**: unix ms, from sprint start date (H1 heading) + day offset.

---

## Input Adapter: WWA Blocks

Use when the input is one or more blocks in this shape (the exact output of pm-execution's `wwas` skill):

```
**Title:** [What will be delivered]

**Why:** [1–2 sentences]

**What:** [Short description]

**Acceptance Criteria:**
- [Outcome 1]
- [Outcome 2]
```

**Mapping — one issue per block, no day/sprint grouping needed:**
| WWA field | Spelflow field |
|---|---|
| `Title` | Issue `title` |
| `Why` + `What` | Issue `description`, formatted as `**Why:** {why}\n\n**What:** {what}\n\n---\n\n**Acceptance Criteria:**\n- {ac1}\n- {ac2}...` |
| — (WWA has no priority field) | Ask the user, or default to `medium` |
| — (no status field) | `Todo` |
| — (no assignee field) | Omit `assignee` unless the user specifies one |

**Grouping into a milestone**: if the WWA blocks came from one `/write-stories wwa` run for a single feature, offer to create a milestone named after that feature (same flow as Step 4b below) and attach all resulting issues to it — this is the closest approximation to "these belong together" available through this API (native sub-issue/parent-child linking is **not** exposed by this REST API — see Limitations).

**job-stories and user-stories blocks**: same mechanical mapping — `Description` (the "When.../As a..." line) plus `Acceptance Criteria` list map the same way as WWA's `Why`+`What`+AC. Detect by the presence of `**Description:**` instead of separate `**Why:**`/`**What:**` lines.

---

## Limitations (be upfront about these)

- **No native sub-issues.** The v1 REST API doesn't accept `attachedTo`/parent-issue on create or patch — every issue is flat, attached only to its Project. Spelflow's internal data model supports subtask hierarchy, but it's not exposed here. If the user wants a feature's WWA items visibly grouped, use a milestone (see above), not a parent-child structure.
- **No native checklist field.** Checkboxes live inside the markdown `description` string; there's no separate structured checklist object.

---

## Workflow — Project & Milestone Setup

**Scope principle (2026-07-24 decision — do not build around this):** this skill never models a "Company"/organization concept above Workspace, and it never auto-infers or auto-creates which Project a piece of work belongs to. Both are decisions the human makes deliberately — this skill's job is to ask at the right moment and execute the answer, not to classify or guess from content. This was a deliberate scope decision after surveying every PM-skill library available (phuryn/pm-skills, deanpeters, CPO Advisor) and finding none of them provide a working project-classification model worth copying — see [[pm-project-hierarchy-synthesis]] for the full comparison and [[pm-skills-workspace-concept]] for the underlying research. Concretely: when multiple Projects exist, always ask which one (never pick by keyword/content match); when creating a new Project, only do it on explicit user request, never as a silent fallback.

**Discover workspace and project** (unless given via context):
```bash
curl -s -H "Authorization: Bearer TOKEN" https://app.spelflow.ru/account/api/v1/workspaces
curl -s -H "Authorization: Bearer TOKEN" https://app.spelflow.ru/account/api/v1/workspaces/WORKSPACE_SLUG/projects
```
One workspace/project → use it automatically, tell the user. Multiple → ask. None → offer to create (below).

**Create a project** (only if none exists or user wants a new one): ask for `name` and a short uppercase `identifier` (≤5 chars). Suggest one derived from the feature/workspace name if the user has no preference.
```bash
curl -s -X POST -H "Authorization: Bearer TOKEN" -H "Content-Type: application/json" \
  -d '{"name": "NAME", "identifier": "IDENT"}' \
  https://app.spelflow.ru/account/api/v1/workspaces/WORKSPACE_SLUG/projects
```
201 → use the returned identifier. 409 → identifier taken, ask for a different one (don't auto-suffix). Other errors → show and ask whether to retry/rename/abort. Cyrillic name → use the temp-file method above.

**Offer a milestone** before creating issues:
> "Create a Milestone for this? (yes/no)" — name suggestion from the sprint/feature name, target date = last sprint day or user-specified.

Check existing milestones first (avoid duplicate labels); if one matches, ask whether to reuse it. `targetDate` = unix ms (`new Date('2026-06-10').getTime()`).

**Deduplication check** before writing: `GET /workspaces/{ws}/projects/{prj}/issues` — if non-empty, warn "Project already has N issues. Did you already import this?" and confirm before continuing. This skill does not deduplicate automatically.

---

## Workflow — Creating Issues

**Dry run first** (always, unless `--dry-run` alone was requested and you're only doing the dry run):
```
Will create N issues in WORKSPACE / PROJECT:
  #  Title                     Priority   Status   Assignee
  1  ...
```
Ask: "Create all N issues? (yes/no/edit)". `--dry-run` flag → print and stop, no confirmation prompt.

**Create sequentially** (not parallel, to preserve ordering). Use the Cyrillic-safe temp-file method for every write. After each: `  ✓ TSK-42  Title  [date]`. On failure (non-201): show the error, ask "Continue with remaining issues? (yes/no)".

**JSON body**:
```json
{ "project": "TSK", "title": "TITLE", "priority": "PRIORITY", "milestone": "ID_OR_OMIT", "assignee": "ID_OR_OMIT" }
```

**Final summary**:
```
Created N/N issues in WORKSPACE / PROJECT:
  TSK-42  Title           ✓
  ...
Next steps:
  • Open https://spelflow.ru (Tracker → PROJECT)
  • Set status/checklists manually where the API doesn't cover it
```
Failures listed separately with their error.

---

## Handling Edge Cases

**Token expired**: re-ask, don't retry the same token. If it was saved to `.env`/global file, ask whether to overwrite the saved copy.

**Project/workspace not found (404)**: show available options from discovery.

**Project identifier taken (409)**: ask user to pick a different one, no auto-suffix.

**Input matches neither adapter**: tell the user plainly ("this doesn't look like a sprint file or WWA blocks") and ask whether to proceed differently, or route to **Step 1 — PM Skills Bridge** if it looks like an unrefined idea instead.

**Already-imported file**: this skill doesn't deduplicate — warn if re-running on a file that looks already loaded.

---

## Related

- [[pm-skills-execution-deep]] and [[pm-skills-project-structure]] in the wiki — full detail on what `wwas`/`create-prd`/etc. produce and where those files should live.
- [[spelflow-account-service-api]] in the wiki — the account-service source backing this API, including confirmation that `attachedTo` is not exposed (see Limitations above).
- `spelflow-plan` (`~/.claude/skills/spelflow-plan/SKILL.md`) — orchestrates the whole PM pipeline (PRD → risk-check → per-feature WWA → this skill) end-to-end for one epic. Use that instead of running each pm-execution command by hand when starting from a large, unrefined idea.
