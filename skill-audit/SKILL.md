---
name: skill-audit
description: "Skill library audit SOP: run 9 checks on all skills, produce a decision table. Use when: auditing a skill library, governance, skill deduplication, skill slimming, dead-link detection, or after creating/editing/deleting skills. Negative triggers: writing a new skill (use skill-authoring-guide instead), single-skill cold-run (belong to authoring side)."
version: 1.0.0
license: MIT
metadata:
  tags: [skills, audit, governance, library, quality]
---

# Skill Library Audit (full-model detection)

Entire audit = model reads files + model judges. No scripts, no snapshots, no decision history. Three body blocks: checks / flow / decision table.

**Division of labor**: audit covers **existing library only** (on-demand governance of the stock). It does not replace the authoring guide. New/edit of any skill must go through `skill-authoring-guide` (including cold-run). Audit's cold-run role = **read each skill's frontmatter `cold_run` field to judge which have been cold-run and which need a run**, it does not re-run a single-skill cold-run.

> After any change to this skill itself, **REQUIRED SUB-SKILL:** load `skill-authoring-guide` first, run TDD + cold-run acceptance (zero-context new session invokes this skill once, confirm no misunderstanding in the 9 checks) before considering done.

## Checks (9 items, in dependency order)

> **Boundary**: audit only does static checks that a full-library / cross-skill view cannot catch (overlap, dead links, facts, bloat, routing). **Cold-run acceptance is NOT here** — cold-run belongs to `skill-authoring-guide`, done "when editing a skill". Audit runs fast, batchable, does not duplicate cold-runs.
> **Cold-run status (read, not run)**: scan each skill's frontmatter `cold_run` field. Format: `"v2.1.0 @ 2026-09-26 | sub-agent cold-run"` — extract the v number (drop the v prefix); == current `version` → that version is cold-run (skip); missing `cold_run` field, missing `version` field, or v number mismatch → not cold-run / stale, mark in report "**needs cold-run**" section (do not run it, hand off to `skill-authoring-guide` for TDD cold-run); do NOT put in "needs confirmation". This is the only place audit touches cold-run.

**Dependency order** (editing one affects another; run in this order):
#1 structure → #5 overlap → #6 facts → #8 goal-guard → #9 expression-direction → re-check #3/#4/#2/#7.
Coupling: after deleting a skill, re-run #1 (dead links change); after #8/#9 rewrite, re-run #3.

**Priority**: Concise language is the **top priority** — trim to concise first, then judge remaining items; verbose/wordy/causal-explanation/jargon hits #2 volume, #3 style preferentially. Judgment anchors must be sharp (give "how to judge", not vague pointers like "params = recipe"). Troubleshooting uses linear diagnosis flow (check X first, then Y), not "symptom→conclusion" lookup table.

Criteria in two tiers: **hard** = machine-falsifiable (dead link, line count, date stamp); **soft** = model-judged structural quality.

**Action on hit (auto-approve + exception only stops)**:
- **Routine fixes, no confirmation needed**: hard hits (#1/#2/#6) → fix + re-run; soft routine hits (#3 style trim, #7 split, #9 rewrite, #4 add trigger words, #6 update stale facts) → fix directly, log in report.
- **Stop and ask only for these two classes**: ① cannot auto-resolve (delete/merge skills, delete Cron, split sets, cross-skill reference changes); ② changes affecting overall functionality (change default recipe, change startup params, change a skill's core behavior / goal).
- Unclear whether a hit is "routine" or "affects overall" → put in "needs confirmation" section of report, do not change.

| # | Check | Criterion | Tier |
|---|---|---|---|
| 1 structure | frontmatter valid, name=dir name, related/references no dead links | YAML parse failure / dead link → fix | hard |
| 2 volume | description ≤200 chars, body <500 lines | over limit → split to references/ | hard |
| 3 style | no narrative, no date stamps, no redundancy | "in round X we found" / synonym repeats → trim | soft |
| 4 routing | description trigger-style not flow-style, trigger words not too broad | reword doesn't trigger → add keywords | soft |
| 5 overlap | pairwise function compare vs whole library | compare core functions (verb+object) of two skills' description+title: >70% overlap → merge; same tool different scenarios → split | soft |
| 6 facts | hard-coded paths/scripts/ports/APIs vs current disk & processes | tool renamed / API changed → update or delete | hard |
| 7 structure | main file only index+flow+criteria, heavy ref / big code already split | inline big code / should-split-not → split | soft |
| 8 goal-guard | content serves declared goal (goal = description first sentence + title) | unrelated function → delete | soft |
| 9 expression-direction | negation→positive (red-line sentences exempt); per-item "ratio criterion": delete → active action inferable? can't infer + has value = fixed; can't infer + no value = should-fix-not (value-pick: fix or mark needs-verify) / anchor-too-vague (pointer: sharpen cite value) (disposition in references #9); adult skills skip entire item | see references #9 | soft |

## Flow (walk in order; routine hit = fix that point, continue; "stop and ask" only for the two exception classes, see action-on-hit section)

**1. Scan**: traverse the skills directory (exclude `.archive/`), two levels: `*/` and `*/*/`, read each SKILL.md + references/; same pass: check `cold_run` field (per read logic above, missing/mismatch → mark "needs cold-run", do not run).

**2. Detect**: each skill passes 1–9 → soft hits read references to re-verify → produce decision table.

**3. Decision table**: follow the template below; routine hits are fixed and logged in report; only "needs confirmation" class (cannot auto-resolve / affects overall function) stops for user sign-off.

**4. Execute** (each "delete/merge" skill follows this judgment chain):
```
references/ referenced by surviving side? → copy in before deleting
Cron-bound? → warn scheduled task will be invalidated
Set member? → keep whole set, don't delete individually
Move to .archive/<batch>/ (can mv back) → re-check dead links across whole library after deletion
```

## Decision Table Template (with exclusion method)

| Skill | Verdict | Rationale (hit items) | Exclusion (why not another verdict) | Disposition |
|---|---|---|---|---|
| `x` | keep / merge / delete / tighten | 5-overlap: >70% overlap with `y` | not keep: >70% keeping = redundant | merge into `y`, archive `x` |

Exclusion method: judging "delete" write why not keep / not merge; judging "merge" write where to merge and why not delete; judging "keep" write hit items + fixable.

## Hard Caps (anti-bloat)
- Main file ≤80 lines (check #2's <500 lines is a library-wide floor; 80 lines is the daily standard for the main file); main file = index + flow + criteria; heavy refs split to references/; inline big code = rejected.
- No "active skill list" snapshot (computed live; snapshots always go stale).
- This skill exceeding its own caps = self-violation of the standard; slim itself before auditing others.

## Audit Report (output this summary after running, no process narrative)

Format + section conventions (optimized items / needs cold-run / needs confirmation — three sections) see `references/report-format.md`.
