---
name: spelflow-plan
description: "Orchestrates the full pm-skills pipeline for one epic — PRD, optional risk-check, per-feature WWA backlog generation, and Spelflow push via spelflow-loader — with project-convention file output and resumability. Use when a large, unrefined epic needs to go end-to-end from idea to Spelflow issues."
trigger: /spelflow-plan
---

# /spelflow-plan — Epic → PRD → Backlog → Spelflow Orchestrator

> **Working name.** `spelflow-plan` has not been confirmed by the user as final — if they want a different name, rename this directory and the `name:`/`trigger:` fields, nothing else depends on the name.

**What this is:** the missing connective tissue between the PM pipeline (`pm-execution` plugin — `create-prd`, `pre-mortem`, `wwas`) and [[spelflow-loader]] (Spelflow push). It takes one large, unrefined epic description, walks it through PRD → optional risk-check → per-feature backlog generation → Spelflow push, with checkpoints where it actually matters and none where it doesn't.

**Built before a manual dry-run** — several design decisions below are explicitly flagged as unverified. Treat the first real run as a test, not a guarantee. See "Unverified — check on first real run" at the bottom.

---

## Usage

```
/spelflow-plan                                        # ask for the epic conversationally
/spelflow-plan <epic description>                     # start immediately
/spelflow-plan <path to existing PM/features/<file-stem>.md>  # resume an in-progress epic
```

No argument: ask 2–3 questions up front — what/why, who it's for, why now — mirroring what `create-prd` itself needs as input. Don't front-load a full 6-question interrogation; the PRD step will surface the rest.

---

## Step 0 — Resume Detection (always runs first)

1. Derive `<slug>` (kebab-case) from the epic description or the given file path. Confirm it with the user in one line before writing anything — every later path depends on it.
2. **Resolve `<FILESTEM>` — the actual filename prefix to use. Never hardcode `spelflow-` (or this skill's own name, or any other fixed string) as a project-name prefix.** This skill is installed into many different projects via the Claude Code plugin marketplace and `npx skills` — a prefix that made sense for one person's own Spelflow-tracking project is noise (or actively wrong) in anyone else's. Resolve it by checking, in order:
   - Does the target project have its own `CLAUDE.md` (or equivalent agent-instructions file) with an explicit PM file-naming rule? If yes, follow that rule exactly — it always wins.
   - Otherwise, look at the existing files already in `PM/features/` (if any) for a consistent prefix pattern already in use in *this* project, and match it.
   - Otherwise (the common case — a fresh project with no established convention yet): `<FILESTEM>` = `<slug>`, with **no prefix at all**. Don't invent one.
   If genuinely unsure which convention applies, ask the user once rather than guessing — this choice is baked into every filename for the rest of the epic.
3. Check what already exists:

| State | Action |
|---|---|
| Neither `PM/features/<FILESTEM>.md` nor `PM/features/<FILESTEM>-tasks.md` exists | Fresh run — go to Step 1 |
| Only `<FILESTEM>.md` exists | PRD is done. Re-show its §7.2 Key Features list for confirmation, then go to Step 3 (feature loop) |
| Both exist | Parse `## Задача N` / `## Task N` headers in the tasks file, fuzzy-match each against §7.2 Key Feature names. Matched features → report "already done, skipping." Unmatched → queue into the loop. If every feature matches, skip straight to Step 4 (Spelflow push readiness) |
| A feature name in the PRD doesn't cleanly match any existing task section | **Don't guess.** Ask: "Feature 'X' doesn't match any existing section — new feature, or does it correspond to existing section 'Y'?" |

---

## Step 1 — PRD

Try direct sub-invocation first:
```
Skill(skill: "pm-execution:create-prd", args: "<epic description + any answers gathered>")
```

**If this errors** (tool/skill name not resolvable): tell the user plainly — "Direct sub-invocation of pm-execution's create-prd isn't available in this session — applying its template inline instead." Then apply the template from **Fallback Templates → PRD** below, in this same conversation, without the sub-invocation.

**Save to `PM/features/<FILESTEM>.md`** (not pm-execution's own default `PRD-[name].md` — the target project's own convention, resolved in Step 0, wins). If the epic description was written in Russian, write the PRD in Russian; otherwise English — match the input language, don't hardcode one.

**Checkpoint (required, highest-leverage gate):** show at minimum §4 Objective, §5 Market Segment, §7.2 Key Features. Ask the user to approve, edit, or request rework. Do not write the file or proceed until they do.

---

## Step 2 — Risk-check (conditional)

**Only `pre-mortem` is available** — `strategy-red-team` is referenced in some pm-skills documentation but is **not installed** in this environment (verified against the actual `pm-execution` skill directory). If the user asks for a red-team pass, check `C:\Users\user\.claude\plugins\installed_plugins.json` for it; if absent, tell them it isn't installed rather than silently skipping or failing.

**Default recommendation**: if the epic reads as high-blast-radius (touches core/shared behavior, not an isolated feature — e.g. "rework how the agent handles X, Y, Z at once"), default to **yes, run pre-mortem**, same judgment call `spelflow-loader`'s own routing table already makes for risky work. Otherwise ask once, opt-in.

If running:
```
Skill(skill: "pm-execution:pre-mortem", args: "<the PRD content>")
```
Same fallback rule as Step 1 if sub-invocation fails (see Fallback Templates → Pre-Mortem).

Save to `PM/ideas/<slug>/premortem-<date>.md`.

**Checkpoint (lightweight):** show the Tigers list, ask "proceed to backlog generation, or does this change the PRD first?"

---

## Step 3 — Feature loop

Extract §7.2 Key Features from the PRD as a discrete list. **If it isn't cleanly list-shaped** (prose instead of bullets), show your extraction and ask the user to confirm/edit it before proceeding — don't silently guess at boundaries.

For each Key Feature not already done (per Step 0):
```
Skill(skill: "pm-execution:wwas", args: "PRODUCT: <epic/product name>\nFEATURE: <this feature's name + its §7.2 paragraph>\nDESIGN: <any relevant §7.1 references>\nASSUMPTIONS: <relevant §7.4 items>")
```
Fallback rule same as above (Fallback Templates → WWA) if sub-invocation fails.

**Split signal** (heuristic — untested, expect to retune after the real run): a single feature's `wwas` run returns more than ~6–8 items, or any one item's Acceptance Criteria list exceeds ~6 entries, or its "What" section runs past `wwas`'s own "1–2 paragraph" cap. When triggered: **show the oversized output and ask** whether to split into two narrower `wwas` reruns (and how) — never auto-split.

No checkpoint per feature inside this loop — batch review happens once, at Step 4.

---

## Step 4 — Collect backlog & checkpoint

Write all collected WWA items into `PM/features/<FILESTEM>-tasks.md`. This file must satisfy **two shapes at once**:

1. The target project's own convention for this kind of file (if it has prior examples — match them; otherwise use this default): a `## Статусы задач` (or `## Task Status` — match language) table up top listing every item with a status checkbox, followed by one `## Задача N — <Title>` (or `## Task N`) section per item.
2. Inside each section's body, **preserve the literal `**Title:**` / `**Why:**` / `**What:**` / `**Acceptance Criteria:**` field labels unmodified** — do not reformat or "clean up" this markup, even though it's nested under a numbered header. `spelflow-loader`'s WWA Blocks input adapter pattern-matches on these exact labels regardless of surrounding structure — reformatting them silently breaks the Spelflow-push step.

**Checkpoint (required):** "N Key Features → M backlog items total, saved to `<FILESTEM>-tasks.md` — ready to hand off to spelflow-loader for the Spelflow dry-run?" This is a go/no-go on entering the push stage — distinct from `spelflow-loader`'s own per-issue dry-run, which still happens after this and is not duplicated here.

---

## Step 5 — Spelflow push handoff

**Scope principle (2026-07-24 decision):** this skill never decides which Spelflow Project (or Company/org-level grouping) an epic belongs to — that's a human decision, executed by `spelflow-loader`'s own discovery-and-ask flow, not inferred here from the epic's content. Do not add project-classification logic to this step. See [[pm-project-hierarchy-synthesis]] for why.

**Default: one milestone for the whole epic**, not one per feature — the only grouping primitive available given `spelflow-loader`'s documented lack of native sub-issues; fragmenting across per-feature milestones would lose the epic-level view entirely. Ask once: "Group all items under one milestone '<epic name>', or one per feature (<list>)?"

Hand off:
```
Skill(skill: "spelflow-loader", args: "<path to <FILESTEM>-tasks.md>, milestone: <name>")
```
Do not reimplement token lookup, dry-run, confirmation, or Cyrillic-encoding handling here — all of that stays inside `spelflow-loader`.

---

## Step 6 — Final report

Fixed template — keep consistent across runs:

```
## spelflow-plan: <epic name> — complete

**PRD**: PM/features/<FILESTEM>.md
**Risk-check**: PM/ideas/<slug>/premortem-<date>.md   (or "skipped")
**Backlog**: PM/features/<FILESTEM>-tasks.md  (N features → M items)

**Spelflow**:
- Workspace: <slug>   Project: <IDENT>
- Milestone: <name> (id: <milestone-id>)
- Issues created: <IDENT>-NN … <IDENT>-MM  (M issues)

**Follow-ups**:
- <any deferred features>
- <any unresolved oversized-feature splits>
```

---

## Fallback Templates

Use these **only if** `Skill(skill: "pm-execution:...")` sub-invocation fails. Sourced verbatim from [[pm-skills-execution-deep]].

### PRD (create-prd)

8 sections: 1. Summary (2–3 sentences) · 2. Contacts · 3. Background (context, why now) · 4. Objective (what+why+benefit+SMART Key Results) · 5. Market Segment (by problem/JTBD, not demographics) · 6. Value Proposition (jobs/gains/pains/competitive advantage) · 7. Solution (7.1 UX/Prototypes · 7.2 Key Features · 7.3 Technology, optional · 7.4 Assumptions) · 8. Release (relative timeline, MVP vs. later). Accessible language, no jargon; flag every assumption explicitly.

### Pre-Mortem

```
## Pre-Mortem Analysis: [Product Name]

### Tigers (Real Risks)
[Each: category, mitigation, owner, due date]

### Paper Tigers (Overblown Concerns)
[Each: what it is, why it's not real]

### Elephants (Unspoken Worries)
[Each: what it is, recommended investigation]

### Action Plans for Launch-Blocking Tigers
| Risk | Mitigation | Owner | Due Date |
```
Tiger = real, evidenced risk. Paper Tiger = overblown, document why not real. Elephant = uncertain, undiscussed, needs investigation.

### WWA (wwas)

```
**Title:** [What will be delivered]

**Why:** [1–2 sentences connecting to strategic context]

**What:** [Short description; 1–2 paragraphs max]

**Acceptance Criteria:**
- [Observable outcome 1]
- [Observable outcome 2]
- [Observable outcome 3]
```
INVEST: Independent, Negotiable, Valuable, Estimable, Small (one sprint), Testable.

---

## Unverified — check on the first real run

1. Whether `Skill(skill: "pm-execution:create-prd", ...)` resolves as designed — fallback templates exist precisely because this isn't confirmed.
2. `strategy-red-team`'s absence is confirmed for this environment, not assumed permanently true — re-check if the plugin is later updated/reinstalled.
3. The 6–8-item / 6-AC split heuristic is invented, not sourced from pm-execution — expect to retune against real output.
4. Fuzzy feature-name matching for resumability (Step 0) — spot-check the first resumed run manually rather than trusting it blind.
5. Output-language matching — untested against pm-execution's actual behavior, which may default to English regardless of input language.
6. Whether `create-prd`'s §7.2 is reliably list-shaped for mechanical parsing, or needs the "ask user to confirm" fallback every time.
7. **`<FILESTEM>` resolution (Step 0.2, added 2026-10-01)** — the resolution order (target project's own `CLAUDE.md` rule → existing `PM/features/` naming pattern → no-prefix default) hasn't been exercised end-to-end on a real run yet. Before this fix, the skill hardcoded `spelflow-<slug>.md` for every installation, which produced wrongly-prefixed files in every project this skill was used on besides its author's own Spelflow-tracking project.

---

## Related

- [[spelflow-loader]] — does the actual Spelflow API work; this skill only prepares its input and hands off.
- [[pm-skills-execution-deep]] — full detail on every pm-execution skill's template and behavior.
- [[pm-skills-project-structure]] — the `PM/` file convention this skill writes into.
