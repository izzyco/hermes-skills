     1|---
     2|name: desloppify
     3|description: >
     4|  Multi-language codebase health scanner and fix orchestrator. Combines
     5|  mechanical detection (dead code, duplication, complexity) with LLM-based
     6|  subjective review (naming, abstractions, error handling, module design) into
     7|  a scored improvement loop. Use when the user explicitly asks to run
     8|  desloppify, scan for technical debt, get a codebase health score, or
     9|  create a systematic cleanup plan. Do NOT trigger for general code review,
    10|  renaming, or fixing individual bugs.
    11|tags: [code-quality, refactoring, technical-debt, linting, codebase-health]
    12|---
    13|
    14|# Desloppify
    15|
    16|A codebase health scoring + fix orchestration tool. Combines mechanical detection (dead code, duplication, complexity) with LLM subjective review (naming, abstractions, error handling, module design) into a persistent scored improvement loop. 29 languages supported. Score > 98 = "a codebase a seasoned engineer would call beautiful."
    17|
    18|**Scoring:** 25% mechanical (auto-detected issues) + 75% subjective (LLM design review). The strict score is the north star — wontfix items count against you, re-reviews can lower scores. Gaming-resistant by design.
    19|
    20|---
    21|
    22|## Installation
    23|
    24|```bash
    25|pip install --upgrade "desloppify[full]"
    26|desloppify update-skill hermes   # installs the Hermes-specific overlay into context
    27|```
    28|
    29|⚠️ **ALWAYS do this first, on every codebase, before scanning:**
    30|
    31|```bash
    32|grep -q '\.desloppify' .gitignore 2>/dev/null || echo '.desloppify/' >> .gitignore
    33|git add .gitignore && git commit -m "chore: add .desloppify to .gitignore" 2>/dev/null || true
    34|```
    35|
    36|This ensures the cache folder (local state, diffs, review packets, session files) is never committed — even if the repo has no `.gitignore` yet. If you forget and `.desloppify/` files are already committed to the branch, see the **Pitfalls** section for recovery steps.
    37|
    38|---
    39|
    40|## 1. Your Job
    41|
    42|Maximise the **strict score** honestly. Main cycle: **scan → plan → execute → rescan**.
    43|
    44|Follow the scan output's **INSTRUCTIONS FOR AGENTS** — don't substitute your own analysis. **Don't be lazy.** Large refactors and small fixes deserve equal energy. No task is too big or too small — fix things properly, not minimally.
    45|
    46|---
    47|
    48|## 2. The Workflow
    49|
    50|### Phase 1: Scan & Review — understand the codebase
    51|
    52|```bash
    53|# Before scanning, check for dirs to exclude (vendor, build output, generated code, worktrees)
    54|desloppify exclude <path>     # exclude obvious ones; share questionable candidates with user first
    55|
    56|desloppify scan --path .      # analyse the codebase (use "." for whole project, or "src/" etc.)
    57|desloppify status             # check current scores
    58|desloppify next               # shows exactly what to do next — always run this after scanning
    59|```
    60|
    61|`next` is the execution queue from the living plan, not the full backlog. Follow it; don't substitute your own analysis.
    62|
    63|If subjective dimensions need review, the scan will say so. To trigger manually:
    64|```bash
    65|desloppify review --prepare   # then follow the Hermes review workflow below
    66|```
    67|
    68|### Phase 2: Plan — decide what to work on
    69|
    70|After reviews, triage stages and plan creation appear in the queue via `next`. Complete them in order:
    71|
    72|```bash
    73|desloppify next               # shows the next execution workflow step
    74|desloppify plan triage --stage observe  --report "themes and root causes..."
    75|desloppify plan triage --stage reflect  --report "comparison against completed work..."
    76|desloppify plan triage --stage organize --report "summary of priorities..."
    77|desloppify plan triage --complete --strategy "execution plan..." --attestation "..."
    78|```
    79|
    80|For automated triage: `desloppify plan triage --run-stages --runner claude`
    81|
    82|Shape the queue — **the plan drives everything `next` gives you**:
    83|
    84|```bash
    85|desloppify plan               # see the living plan details
    86|desloppify plan queue         # compact execution queue view
    87|desloppify plan reorder <pat> top     # reorder — what unblocks the most?
    88|desloppify plan cluster create <name> # group related issues to batch-fix
    89|desloppify plan focus <cluster>       # scope next to one cluster
    90|desloppify plan skip <pat>            # defer — hide from next
    91|desloppify backlog                    # inspect broader open work (not execution queue)
    92|```
    93|
    94|### Phase 3: Execute — grind the queue
    95|
    96|**Branch first.** Never commit health work directly to main:
    97|
    98|```bash
    99|git checkout -b desloppify/code-health   # or desloppify/<focus-area>
   100|desloppify config set commit_pr 42       # optional: link a PR for auto-updated descriptions
   101|```
   102|
   103|**The loop:**
   104|
   105|```bash
   106|desloppify next                          # 1. get next item
   107|# 2. fix the issue in code
   108|# 3. resolve it (next shows the exact resolve command + required attestation)
   109|git add <files> && git commit -m "desloppify: fix 3 deferred_import findings"
   110|desloppify plan commit-log record        # moves findings uncommitted → committed, updates PR
   111|git push -u origin desloppify/code-health
   112|# repeat until queue is empty
   113|```
   114|
   115|Score may temporarily drop after fixes — cascade effects are normal, keep going. If `next` suggests an auto-fixer:
   116|
   117|```bash
   118|desloppify autofix <fixer> --dry-run    # preview
   119|desloppify autofix <fixer>              # apply
   120|```
   121|
   122|**When the queue is clear, return to Phase 1.** New issues will surface, cascades will have resolved. This is the cycle.
   123|
   124|---
   125|
   126|## 3. Hermes-Specific: Parallel Review with delegate_task
   127|
   128|Hermes has built-in parallel subagent support via `delegate_task` (up to 3 concurrent children). Use this for subjective review batches and triage stages.
   129|
   130|### Review Workflow
   131|
   132|```bash
   133|# Step 1: generate prompt files and blind packet
   134|desloppify review --run-batches --dry-run
   135|# → writes .desloppify/subagents/runs/<run-id>/prompts/batch-N.md
   136|# → prints the run directory path
   137|```
   138|
   139|Launch subagents in batches of 3:
   140|
   141|```python
   142|delegate_task(tasks=[
   143|  {
   144|    "goal": "Review batch 1. Read the prompt at .desloppify/subagents/runs/<run-id>/prompts/batch-1.md, follow it exactly, inspect the repository, and write ONLY valid JSON to .desloppify/subagents/runs/<run-id>/results/batch-1.raw.txt.",
   145|    "context": "Repository root: <cwd>. Blind packet: .desloppify/review_packet_blind.json. The prompt file defines the required output schema. Do not edit repository source files.",
   146|    "toolsets": ["terminal", "file"]
   147|  },
   148|  { "goal": "Review batch 2 ...", "context": "...", "toolsets": ["terminal", "file"] },
   149|  { "goal": "Review batch 3 ...", "context": "...", "toolsets": ["terminal", "file"] }
   150|])
   151|```
   152|
   153|Wait for each group of 3 to finish before launching the next group.
   154|
   155|```bash
   156|# Step 3: import after all batches have matching results
   157|desloppify review --import-run .desloppify/subagents/runs/<run-id> --scan-after-import
   158|```
   159|
   160|**Key constraints:**
   161|- `delegate_task` supports at most **3 concurrent children** at a time
   162|- Subagents do not inherit parent context — prompt file + blind packet must provide everything
   163|- Subagents cannot call `delegate_task`, `clarify`, `memory`, or `send_message`
   164|- Results must be named `batch-N.raw.txt` (NOT `.json`) — the importer expects this extension
   165|- The blind packet intentionally omits score history to prevent anchoring bias
   166|
   167|### Triage Workflow (Parallel)
   168|
   169|Run triage stages sequentially. For each stage:
   170|
   171|```bash
   172|desloppify plan triage --run-stages --runner claude   # prints orchestrator instructions
   173|```
   174|
   175|For each stage (observe → reflect → organize → enrich):
   176|1. Get the stage prompt: `desloppify plan triage --stage-prompt <stage>`
   177|2. If it benefits from parallel work, use `delegate_task(tasks=[...])` in groups of 3
   178|3. Record output: `desloppify plan triage --stage <stage> --report "..."`
   179|4. Confirm: `desloppify plan triage --confirm <stage> --attestation "..."`
   180|5. Complete: `desloppify plan triage --complete --strategy "..." --attestation "..."`
   181|
   182|---
   183|
   184|## 4. Review Output Format
   185|
   186|Return machine-readable JSON for review imports:
   187|
   188|```json
   189|{
   190|  "session": {
   191|    "id": "<session_id_from_template>",
   192|    "token": "***"
   193|  },
   194|  "assessments": {
   195|    "<dimension_from_query>": 0
   196|  },
   197|  "findings": [
   198|    {
   199|      "dimension": "<dimension_from_query>",
   200|      "identifier": "short_id",
   201|      "summary": "one-line defect summary",
   202|      "related_files": ["relative/path/to/file.py"],
   203|      "evidence": ["specific code observation"],
   204|      "suggestion": "concrete fix recommendation",
   205|      "confidence": "high|medium|low"
   206|    }
   207|  ]
   208|}
   209|```
   210|
   211|Use `"findings": []` when no defects found. Import is fail-closed: invalid findings abort unless `--allow-partial` is passed.
   212|
   213|**Integrity rules:** Score from evidence only — no prior chat context, score history, or target-threshold anchoring. When evidence is mixed, score lower and explain uncertainty. Assess every requested dimension; never drop one.
   214|
   215|---
   216|
   217|## 5. Reference
   218|
   219|### Key Concepts
   220|
   221|- **Tiers**: T1 auto-fix → T2 quick manual → T3 judgment call → T4 major refactor
   222|- **Auto-clusters**: related findings are auto-grouped in `next`. Drill in with `next --cluster <name>`
   223|- **Zones**: production/script (scored), test/config/generated/vendor (not scored). Fix with `zone set`
   224|- **Wontfix cost**: widens the lenient↔strict gap. Challenge past decisions when the gap grows
   225|
   226|### Score Types
   227|
   228|| Score | Meaning |
   229||-------|---------|
   230|| overall (lenient) | wontfix items excluded |
   231|| **strict** | wontfix items count as open — **use this** |
   232|| objective | mechanical only |
   233|| verified | confirmed fixes only |
   234|
   235|### Import Paths
   236|
   237|- **Robust session flow (recommended)**: `desloppify review --external-start --external-runner claude` → use generated prompt/template → run printed `--external-submit` command
   238|- **Durable scored import (legacy)**: `desloppify review --import findings.json --attested-external --attest "I validated this review was completed without awareness of overall score and is unbiased."`
   239|- Import first, fix after — import creates tracked state entries for correlation
   240|- Target-matching scores trigger auto-reset to prevent gaming
   241|
   242|---
   243|
   244|## Pitfalls
   245|
   246|- **Don't rescan mid-queue** — finish the queue first, then rescan
   247|- **Score drops are normal** after fixes — cascade effects resolve in the next scan cycle
   248|- **Batch result filenames must be `batch-N.raw.txt`** — `.json` extension breaks the importer
   249|- **Always use the blind packet** (`review_packet_blind.json`) for subagents, not `query.json` — the blind packet omits score targets to prevent anchoring
   250|- **Exclude generated/vendor dirs before scanning** or you'll get noise that inflates the issue list
   251|- **`delegate_task` max 3 concurrent** — always group into batches of 3, wait, then launch next batch
   252|- **Don't skip triage** — `next` without a triaged plan is noisy and unordered; triage shapes the queue
   253|- **CLI not on PATH on macOS** — `desloppify` binary may not be discoverable; use `python3 -m desloppify` throughout
   254|- **Initial score ~18 is normal** — that's just the 25% mechanical baseline before subjective reviews; jumps to 60–75 after the first review cycle
   255|- **Stale dimensions after code fixes** — scores show `[stale — re-review]` after structural changes; run a full `--force-review-rerun` cycle to capture the real improvement
   256|- **`--force-review-rerun` required for second cycle** — desloppify blocks reruns if the backlog isn't drained; add `--force-review-rerun` to `review --run-batches --dry-run`
   257|- **Enrich stage is strict** — step `--detail` strings must be 80+ chars, reference files that exist on disk, and be concrete. Iterate with `plan cluster update <name> --update-step N --detail "..."` when rejected; the attestation on `--confirm enrich` must also name at least one cluster explicitly
   258|- **Triage subagent hits max_iterations** — the full 6-stage triage is too long for one subagent at 50 iterations; run strategize in one subagent, then stages 2–6 in a second subagent with `max_iterations=60`
   259|- **Second review pass finds new issues** — don't expect a clean re-review after fixes; the second pass often surfaces things the first missed (e.g. type safety erosion in the component layer only visible after lib-layer enums were fixed)
   260|- **Use Claude Code for execution, not delegate_task** — for the actual fix loop (build + test + commit per cluster), `claude --dangerously-skip-permissions` running in background PTY mode is more reliable than delegate_task subagents, which time out on long builds
   261|- **`desloppify` may not be in PATH** — if `desloppify: command not found`, use `python3 -m desloppify` instead (common on macOS with pip3 installs)
   262|- **Triage enrich stage has strict validation** — blocks confirmation until ALL of these pass:
   263|  - Every step needs `--detail` (80+ chars with specific file paths) OR `--issue-refs`
   264|  - File paths in `--detail` must exist on disk — don't reference future/output files by path; describe them by intent ("create new file alongside X" rather than "write to X.ts")
   265|  - `--report` must be 100+ chars
   266|  - `--attestation` on `--confirm enrich` must name at least one cluster explicitly (e.g. "dep-cleanup", "auth-consolidation")
   267|  - `--attestation` on `--confirm sense-check` must also name specific clusters
   268|- **Enrich requires logged cluster update ops** — recording enrich without prior `cluster update` commands since organize will block. Pass `--attestation` to override, but the confirm will still validate step detail quality
   269|- **Queue is empty until triage completes** — `desloppify next` returns "Nothing to do" even with 200+ open issues if triage hasn't been run. Run the full 6-stage triage first: strategize → observe → reflect → organize → enrich → sense-check → complete
   270|- **Triage takes many subagent iterations** — the enrich stage alone required ~15 back-and-forth fix cycles due to path validation. Budget time for it.
   271|- **Use Claude Code for execution** — `claude --dangerously-skip-permissions` handles the full fix loop (next → fix → build → test → resolve → commit) autonomously. Launch as a background PTY process and monitor with `git log --oneline` rather than the garbled PTY output
   272|- **External review token truncated in template** — `review_result.template.json` shows token as `bee583...7e31` (truncated for display). Subagents copy this truncated value and get a token mismatch on submit. Fix with sed after subagent writes the file: `sed -i '' 's/bee583\.\.\.7e31/FULL_TOKEN/' review_result.json`. Full token is in `claude_launch_prompt.md` under "Session token:".
   273|- **`--run-batches --dry-run` fails with empty `investigation_batches`** — caused by `--prepare` defaulting to `src/` path which may not exist. Always use `desloppify review --prepare --path .` for root-level TS/JS projects.
   274|- **External submit blocks if dimension scored below 85 with no issue** — add at least one concrete issue per under-85 dimension to `issues[]` before calling `--external-submit`.
   275|- **`plan resolve` uses `--confirm` not `--attest`** — individual and cluster resolves use `--confirm`. Only triage stage commands use `--attestation`.
   276|- **Enrich stage: steps enriched before `organize` is confirmed are ignored** — after confirming organize, run at least one `cluster update --update-step N --detail TEXT` or `--issue-refs` per cluster before recording the enrich stage.
   277|- **Enrich `--confirm` blocks on missing `--effort` tags** — every step needs `--effort trivial|small|medium|large`; set with `cluster update NAME --update-step N --effort small`.
   278|- **`review --prepare` path must match actual source root** — use `--prepare --path .` for projects where source lives at repo root (not under `src/`).
   279|- **`scorecard.png` and `pbcopy` accidentally committed to repo root** — the desloppify cron job generates a `scorecard.png` report and may pipe diffs via `pbcopy`. Both can land as untracked files in the repo root and get swept up in a `git add .` commit. Fix: `git rm scorecard.png pbcopy && git commit -m "chore: remove accidental artifacts" && git push`. To prevent: add `scorecard.png` and `pbcopy` to `.gitignore`, and never use `git add .` in desloppify fix commits — always stage specific files.
   280|- **`.desloppify/` files accidentally committed** — if the cache folder landed in git history before `.gitignore` was updated, do this: (1) add `.desloppify/` to `.gitignore`, (2) stage the removal with `git update-index --force-remove $(git ls-files .desloppify/)`, (3) add `.gitignore` with `git add .gitignore`, (4) commit with `git commit -m "chore: remove .desloppify cache, add to .gitignore"`, (5) push. Note: `git rm -r --cached .desloppify/` silently fails (exit -1) when the working tree has modified tracked files in that folder; `update-index --force-remove` is the reliable alternative.
   281|- **`gh` CLI is at `/opt/homebrew/bin/gh` on macOS** — not in the default cron PATH. Use `PATH=/opt/homebrew/bin:$PATH gh pr create ...`.
   282|- **Strategize stage requires JSON via `--report-file`, not inline `--report`** — the `--stage strategize` subcommand requires valid JSON (the full StrategistBriefing object), not a plain text string. Write the JSON to a temp file and pass `--report-file /tmp/strategize_report.json`. Passing prose via `--report` gives: `Strategize report must be valid JSON`.
   283|- **Strategize subagent: write JSON to a file, then parent passes it** — have the strategize subagent write the full JSON briefing to a local path (e.g. `.desloppify/triage_strategize.json`), then the parent reads it and calls `--report-file`. Do NOT have the subagent call the desloppify CLI itself — it won't have the right working directory context reliably.
   284|- **Triage with 6 stages needs 3 subagent passes, not one** — observe+reflect+organize fit in 50 iterations (one subagent), but enrich+sense-check+complete need another 30-50 iterations. Recommended split: subagent 1 = strategize (JSON output only), subagent 2 = observe+reflect+organize, subagent 3 = confirm organize + enrich + sense-check + complete.
   285|- **`plan focus --clear` required after cluster resolves** — after a cluster is resolved via `plan resolve`, `next` may still be scoped to that cluster. Run `python3 -m desloppify plan focus --clear` to unblock the queue and see the next cluster.
   286|- **Queue order violation when resolving cluster items individually** — if a cluster has 2 items and you resolve them one at a time, desloppify may reject the second with "queue order violation: these items are not next in the plan queue." Fix: run `python3 -m desloppify plan reorder <pattern> top` to move the item to the front, then resolve it.
   287|- **`desloppify/code-health` remote branch left over from prior nightly run** — force-push is required: `git push -u origin desloppify/code-health --force`. The branch is not auto-deleted after a PR is merged.
   288|- **TS baseline inflation from early `tsc` exit on syntax errors** — if `tsc` hits a fatal syntax error it exits early and reports fewer errors than actually exist. After that syntax error is fixed in a subsequent merge, `tsc` runs to completion and surfaces pre-existing `@types` declaration errors. This looks like a regression but isn't — verify by running `npx tsc --noEmit` on `main` and comparing counts before flagging.
   289|- **`plan reorder <cluster> top` required before working non-first clusters** — the queue enforces ordering. If you want to work cluster 3 before cluster 2 is done, run `python3 -m desloppify plan reorder <name> top` first or the resolve will be blocked.
   290|- **20 review batches for a medium codebase (~70 files) takes ~7 rounds of 3 concurrent subagents** — budget accordingly. Each batch subagent takes 3-8 min. Total wall time ~30-60 min for parallel review of a 70-file TypeScript repo.
   291|- **`--run-batches --dry-run` saying "Prepared packet reuse rejected: missing investigation_batches; rebuilding"** — this is normal on first run or after config changes. It rebuilds the packet and proceeds correctly, no action needed.
   292|- **Open PR already exists for `desloppify/code-health`** — check with `gh pr list --head desloppify/code-health` before calling `gh pr create`. If a PR exists, use `gh pr edit <number> --body "..."` to update it instead of creating a duplicate.
   293|- **`jest-expo` requires `jest` installed separately** — `jest-expo` does not bundle jest; running `npx jest` will fail with `Cannot find module 'jest/package.json'` unless you also `npm install --save-dev jest`. Install all at once: `npm install --save-dev jest jest-expo @testing-library/react-native @types/jest --legacy-peer-deps`.
   294|- **Expo repos need `--legacy-peer-deps` for test library installs** — `@testing-library/react-native` and `jest-expo` often have peer dep conflicts with Expo's locked versions; always add `--legacy-peer-deps`.
   295|- **State file may have corrupt JSON** — `~/.hermes/cron/desloppify-state.json` can accumulate write corruption. Always wrap the state load in try/except and fall back to fresh state; never trust the file without validating json.loads.
   296|