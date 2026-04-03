---
name: desloppify
description: >
  Multi-language codebase health scanner and fix orchestrator. Combines
  mechanical detection (dead code, duplication, complexity) with LLM-based
  subjective review (naming, abstractions, error handling, module design) into
  a scored improvement loop. Use when the user explicitly asks to run
  desloppify, scan for technical debt, get a codebase health score, or
  create a systematic cleanup plan. Do NOT trigger for general code review,
  renaming, or fixing individual bugs.
tags: [code-quality, refactoring, technical-debt, linting, codebase-health]
---

# 🧹 Desloppify

> A codebase health scoring + fix orchestration tool. 29 languages supported.
> **Score > 98** = *"a codebase a seasoned engineer would call beautiful."*

**Scoring:** 25% mechanical (auto-detected issues) + 75% subjective (LLM design review).
The strict score is the north star — wontfix items count against you, re-reviews can lower scores. Gaming-resistant by design.

---

## Installation

```bash
pip install --upgrade "desloppify[full]"
desloppify update-skill hermes   # installs the Hermes-specific overlay into context
```

> ⚠️ **ALWAYS do this first, on every codebase, before scanning:**

```bash
grep -q '.desloppify' .gitignore 2>/dev/null || echo '.desloppify/' >> .gitignore
git add .gitignore && git commit -m "chore: add .desloppify to .gitignore" 2>/dev/null || true
```

This ensures the cache folder is never committed. If you forgot and `.desloppify/` files are already committed, see the [Pitfalls](#-pitfalls) section for recovery steps.

---

## Your Job

Maximise the **strict score** honestly. Main cycle: **scan → plan → execute → rescan**.

Follow the scan output's **INSTRUCTIONS FOR AGENTS** — don't substitute your own analysis. Large refactors and small fixes deserve equal energy. Fix things properly, not minimally.

---

## The Workflow

### Phase 1 — Scan & Review

```bash
# Check for dirs to exclude first (vendor, build output, generated code, worktrees)
desloppify exclude <path>

desloppify scan --path .      # analyse the codebase
desloppify status             # check current scores
desloppify next               # always run this after scanning — it tells you exactly what to do
```

`next` is the execution queue from the living plan, not the full backlog. Follow it.

To trigger a subjective review manually:
```bash
desloppify review --prepare   # then follow the Hermes review workflow below
```

---

### Phase 2 — Plan

After reviews, triage stages appear in the queue via `next`. Complete them in order:

```bash
desloppify plan triage --stage observe  --report "themes and root causes..."
desloppify plan triage --stage reflect  --report "comparison against completed work..."
desloppify plan triage --stage organize --report "summary of priorities..."
desloppify plan triage --complete --strategy "execution plan..." --attestation "..."
```

For automated triage: `desloppify plan triage --run-stages --runner claude`

**Shape the queue — the plan drives everything `next` gives you:**

```bash
desloppify plan                          # see the living plan details
desloppify plan queue                    # compact execution queue view
desloppify plan reorder <pat> top        # reorder — what unblocks the most?
desloppify plan cluster create <name>    # group related issues to batch-fix
desloppify plan focus <cluster>          # scope next to one cluster
desloppify plan skip <pat>              # defer — hide from next
desloppify backlog                       # inspect broader open work
```

---

### Phase 3 — Execute

**Branch first. Never commit health work directly to main:**

```bash
git checkout -b desloppify/code-health
desloppify config set commit_pr 42       # optional: link a PR for auto-updated descriptions
```

**The loop:**

```bash
desloppify next                          # 1. get next item
# 2. fix the issue in code
# 3. resolve it (next shows the exact resolve command + required attestation)
git add <files> && git commit -m "desloppify: fix 3 deferred_import findings"
desloppify plan commit-log record        # moves findings uncommitted → committed, updates PR
git push -u origin desloppify/code-health
# repeat until queue is empty
```

Score may temporarily drop after fixes — cascade effects are normal, keep going.

```bash
desloppify autofix <fixer> --dry-run    # preview auto-fixer
desloppify autofix <fixer>              # apply auto-fixer
```

**When the queue is clear, return to Phase 1.** New issues will surface as cascades resolve. This is the cycle.

---

## Hermes-Specific: Parallel Review with `delegate_task`

Hermes has built-in parallel subagent support via `delegate_task` (up to 3 concurrent children).

### Review Workflow

```bash
# Generate prompt files and blind packet
desloppify review --run-batches --dry-run
# → writes .desloppify/subagents/runs/<run-id>/prompts/batch-N.md
```

Launch subagents in batches of 3:

```python
delegate_task(tasks=[
  {
    "goal": "Review batch 1. Read the prompt at .desloppify/subagents/runs/<run-id>/prompts/batch-1.md, follow it exactly, inspect the repository, and write ONLY valid JSON to .desloppify/subagents/runs/<run-id>/results/batch-1.raw.txt.",
    "context": "Repository root: <cwd>. Blind packet: .desloppify/review_packet_blind.json. Do not edit repository source files.",
    "toolsets": ["terminal", "file"]
  },
  { "goal": "Review batch 2 ...", "context": "...", "toolsets": ["terminal", "file"] },
  { "goal": "Review batch 3 ...", "context": "...", "toolsets": ["terminal", "file"] }
])
```

Wait for each group of 3 to finish before launching the next.

```bash
# Import after all batches have matching results
desloppify review --import-run .desloppify/subagents/runs/<run-id> --scan-after-import
```

**Key constraints:**
- `delegate_task` supports at most **3 concurrent children** at a time
- Subagents do not inherit parent context — prompt file + blind packet must provide everything
- Results must be named `batch-N.raw.txt` (NOT `.json`) — the importer expects this extension
- The blind packet intentionally omits score history to prevent anchoring bias

### Triage Workflow (Parallel)

```bash
desloppify plan triage --run-stages --runner claude   # prints orchestrator instructions
```

For each stage (`observe → reflect → organize → enrich`):
1. Get the stage prompt: `desloppify plan triage --stage-prompt <stage>`
2. If it benefits from parallel work, use `delegate_task(tasks=[...])` in groups of 3
3. Record output: `desloppify plan triage --stage <stage> --report "..."`
4. Confirm: `desloppify plan triage --confirm <stage> --attestation "..."`
5. Complete: `desloppify plan triage --complete --strategy "..." --attestation "..."`

---

## Review Output Format

Subagents must return machine-readable JSON:

```json
{
  "session": {
    "id": "<session_id_from_template>",
    "token": "***"
  },
  "assessments": {
    "<dimension_from_query>": 0
  },
  "findings": [
    {
      "dimension": "<dimension_from_query>",
      "identifier": "short_id",
      "summary": "one-line defect summary",
      "related_files": ["relative/path/to/file.py"],
      "evidence": ["specific code observation"],
      "suggestion": "concrete fix recommendation",
      "confidence": "high|medium|low"
    }
  ]
}
```

Use `"findings": []` when no defects found. Import is fail-closed — invalid findings abort unless `--allow-partial` is passed.

**Integrity rules:** Score from evidence only. No prior chat context, score history, or target-threshold anchoring. Assess every requested dimension; never drop one.

---

## Reference

### Score Types

| Score | Meaning |
|-------|---------|
| **Strict** ⬅ use this | wontfix items count as open |
| Lenient | wontfix items excluded |
| Objective | mechanical only |
| Verified | confirmed fixes only |

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Tiers** | T1 auto-fix → T2 quick manual → T3 judgment call → T4 major refactor |
| **Auto-clusters** | Related findings are auto-grouped in `next`. Drill in with `next --cluster <name>` |
| **Zones** | production/script (scored), test/config/generated/vendor (not scored) |
| **Wontfix cost** | Widens the lenient↔strict gap — challenge past decisions when the gap grows |

### Import Paths

- **Robust (recommended):** `desloppify review --external-start --external-runner claude` → use generated prompt/template → run printed `--external-submit` command
- **Legacy:** `desloppify review --import findings.json --attested-external --attest "I validated this review was completed without awareness of overall score and is unbiased."`

---

## ⚠️ Pitfalls

<details>
<summary><strong>Scanning & Setup</strong></summary>

- **Initial score ~18 is normal** — that's just the 25% mechanical baseline before subjective reviews; jumps to 60–75 after the first review cycle
- **Exclude generated/vendor dirs before scanning** or you'll get noise that inflates the issue list
- **CLI not on PATH on macOS** — use `python3 -m desloppify` throughout if `desloppify: command not found`
- **`--run-batches --dry-run` fails with empty `investigation_batches`** — caused by `--prepare` defaulting to `src/` path. Always use `desloppify review --prepare --path .` for root-level TS/JS projects
- **`--run-batches --dry-run` saying "Prepared packet reuse rejected"** — this is normal on first run or after config changes; it rebuilds and proceeds correctly
- **`scorecard.png` and `pbcopy` accidentally committed** — add both to `.gitignore` and never use `git add .` in desloppify fix commits; always stage specific files
- **`.desloppify/` accidentally committed** — fix: `git update-index --force-remove $(git ls-files .desloppify/)`, then add to `.gitignore` and commit. Note: `git rm -r --cached` silently fails here; `update-index --force-remove` is the reliable alternative

</details>

<details>
<summary><strong>Triage & Planning</strong></summary>

- **Queue is empty until triage completes** — `desloppify next` returns "Nothing to do" even with 200+ open issues if triage hasn't been run. Run the full 6-stage triage first: strategize → observe → reflect → organize → enrich → sense-check → complete
- **Don't skip triage** — `next` without a triaged plan is noisy and unordered; triage shapes the queue
- **Triage with 6 stages needs 3 subagent passes** — observe+reflect+organize fit in 50 iterations; enrich+sense-check+complete need another 30-50. Split: subagent 1 = strategize, subagent 2 = observe+reflect+organize, subagent 3 = enrich+sense-check+complete
- **Triage takes many subagent iterations** — the enrich stage alone can require ~15 back-and-forth fix cycles due to path validation. Budget time
- **Strategize stage requires JSON via `--report-file`**, not inline `--report` — write JSON to a temp file and pass `--report-file /tmp/strategize_report.json`
- **Strategize subagent: write JSON to a file, then parent passes it** — do NOT have the subagent call the desloppify CLI directly; it won't have the right working directory reliably
- **`plan focus --clear` required after cluster resolves** — after `plan resolve`, `next` may still be scoped to that cluster; run `python3 -m desloppify plan focus --clear`
- **Queue order violation when resolving cluster items individually** — fix: `python3 -m desloppify plan reorder <pattern> top` to move the item to the front first
- **`plan reorder <cluster> top` required before working non-first clusters** — the queue enforces ordering; reorder first or the resolve will be blocked
- **`plan resolve` uses `--confirm` not `--attest`** — only triage stage commands use `--attestation`

</details>

<details>
<summary><strong>Enrich Stage (Strict Validation)</strong></summary>

Blocks confirmation until ALL of these pass:

- Every step needs `--detail` (80+ chars with specific file paths) OR `--issue-refs`
- File paths in `--detail` must exist on disk — describe future files by intent, not path
- `--report` must be 100+ chars
- `--attestation` on `--confirm enrich` must name at least one cluster explicitly
- `--attestation` on `--confirm sense-check` must also name specific clusters
- Every step needs `--effort trivial|small|medium|large` — set with `cluster update NAME --update-step N --effort small`
- Steps enriched before `organize` is confirmed are ignored — run at least one `cluster update --update-step N --detail TEXT` per cluster after confirming organize

</details>

<details>
<summary><strong>Review & Scoring</strong></summary>

- **Don't rescan mid-queue** — finish the queue first, then rescan
- **Score drops are normal** after fixes — cascade effects resolve in the next scan cycle
- **Stale dimensions after code fixes** — scores show `[stale — re-review]` after structural changes; run `--force-review-rerun` cycle
- **`--force-review-rerun` required for second cycle** — add it to `review --run-batches --dry-run` if the backlog isn't drained
- **Second review pass finds new issues** — don't expect a clean re-review after fixes; second pass often surfaces things the first missed
- **External review token truncated in template** — `review_result.template.json` shows token as `bee583...7e31`. Fix with sed after subagent writes: `sed -i '' 's/bee583\.\.\.7e31/FULL_TOKEN/' review_result.json`. Full token is in `claude_launch_prompt.md`
- **External submit blocks if dimension scored below 85 with no issue** — add at least one concrete issue per under-85 dimension before calling `--external-submit`
- **Always use the blind packet** (`review_packet_blind.json`) for subagents, not `query.json`
- **Batch result filenames must be `batch-N.raw.txt`** — `.json` extension breaks the importer
- **`delegate_task` max 3 concurrent** — always group into batches of 3, wait, then launch next batch
- **20 review batches for a medium codebase (~70 files)** takes ~7 rounds of 3 concurrent subagents; 30–60 min total wall time

</details>

<details>
<summary><strong>Execution & Git</strong></summary>

- **Use Claude Code for execution, not delegate_task** — `claude --dangerously-skip-permissions` in background PTY mode is more reliable for the fix loop; monitor with `git log --oneline`
- **`desloppify/code-health` remote branch left over from prior run** — force-push required: `git push -u origin desloppify/code-health --force`; the branch is not auto-deleted after PR merge
- **Open PR already exists** — check with `gh pr list --head desloppify/code-health` before `gh pr create`; use `gh pr edit <number> --body "..."` to update instead
- **`gh` CLI is at `/opt/homebrew/bin/gh` on macOS** — not in default cron PATH; use `PATH=/opt/homebrew/bin:$PATH gh pr create ...`
- **TS baseline inflation from early `tsc` exit** — if `tsc` hits a fatal syntax error it exits early; after the fix, `tsc` runs to completion and surfaces pre-existing errors. Verify with `npx tsc --noEmit` on `main` before flagging as regression

</details>

<details>
<summary><strong>Expo / React Native Projects</strong></summary>

- **`jest-expo` requires `jest` installed separately** — install all at once: `npm install --save-dev jest jest-expo @testing-library/react-native @types/jest --legacy-peer-deps`
- **Always use `--legacy-peer-deps`** for test library installs — peer dep conflicts with Expo's locked versions are common

</details>

<details>
<summary><strong>Miscellaneous</strong></summary>

- **State file may have corrupt JSON** — `~/.hermes/cron/desloppify-state.json` can accumulate write corruption; always wrap the state load in try/except and fall back to fresh state
- **Wontfix cost** — widens the lenient↔strict gap; challenge past wontfix decisions when the gap grows

</details>
