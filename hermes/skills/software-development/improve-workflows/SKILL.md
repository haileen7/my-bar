---
name: improve-workflows
version: 1.3.0
author: Hermes Agent (session-derived)
license: MIT
description: "Use when auditing a codebase, triaging filed findings, or reviewing a plan."
tags: [audit, plans, issues, github, review, dx, tooling, discussions, scheduling]
related_skills: [improve, github, requesting-code-review]
---

# Improve Workflows

Operational patterns discovered during real improve skill executions. Supplements the improve skill (shadcn) with Hermes-specific tooling workflows.

## When to Use

- Auditing a codebase and turning findings into a prioritized plan set, optionally registered as GitHub issues.
- Asked to give an opinion, verdict, or review on an existing plan, proposal, or tracking issue — including one written by another agent or a teammate.
- Asked to re-review, re-verify, or triage a **batch of already-filed findings** ("review every open issue that doesn't have `ready`", "do these still hold?") — see "Re-verifying Filed Findings".
- Asked to **replace** filed findings wholesale ("delete the issues without label X and file fresh ones", "rewrite these issues") — see "Replacing a Batch of Filed Findings" and `references/refile-batch.md`.
- Publishing a verdict into a GitHub discussion thread (discussions are not issues; the posting path differs).
- Asked to give an opinion now and again after a delay — a scheduled re-check on a live thread, where the second answer must respond to what landed in between.
- Asked to run the audit on a recurring schedule and file the top finding as a GitHub issue each time (every N minutes, weekly) — see "Recurring Cron Audit" and `references/cron-prompt-scanner.md`.

## Issue Registration (`--issues`)

When the improve skill's `--issues` modifier publishes plans as GitHub issues:

### Procedure

1. Determine `owner/repo` from `git remote -v` — never assume from AGENTS.md or memory. The canonical upstream and the server's fork may use different owner names.
2. Check if issues are enabled: `gh issue list --repo owner/repo`. If "disabled", try the fork remote.
3. Verify labels exist: `gh label list --repo owner/repo`. Create missing ones first or omit.
4. Create: `gh issue create --repo owner/repo --title '...' --body '...' [--label '...']`
5. Record the issue URL in the plan file and tracker.

### Pitfalls

- **GitHub MCP auth failure → switch to `gh` CLI immediately.** Do not retry MCP. The MCP server requires `GITHUB_PERSONAL_ACCESS_TOKEN` env var; `gh` uses stored credentials. One retry wastes time; the fallback is instant.
- **Wrong repo name.** AGENTS.md may reference a canonical name that differs from the actual git remote. `git remote -v` is authoritative. If the primary repo has issues disabled, try the fork remote.
- **Missing labels.** `--label 'improve-audit'` fails if the label doesn't exist. Either create it first (`gh label create improve-audit --repo owner/repo`) or omit labels entirely.
- **Bulk issue registration (10+ plans).** Use `cronjob_manage` with `schedule: 'every 5m'` and `repeat: N` instead of creating all issues in one turn. Track progress in `plans/tracker.json` (JSON array with `done: boolean`, `plan_file`, `issue_url` fields per finding). Set `deliver` to the user's home channel for status updates.

## Re-verifying Filed Findings (Triage Pass)

Trigger: "review every open issue without the ready label", "check whether these findings still hold", any batch of issues written by another agent or by a past scheduled run. A filed issue is a set of claims about **a specific commit**; the working tree — and the upstream branch the fix will actually land on — is the only authority.

### Procedure

1. List the issues once **with labels**, filter in code (drop pull requests, keep a number / state / labels / body-length column), and dump each body to a scratch file with a metadata header (number, title, author, url, labels). Never re-fetch a body already on disk. A list response over ~100 KB gets spilled to a file by the runtime — process that file instead of re-requesting it.
2. **Establish upstream drift before judging any claim.** `git rev-list --count HEAD..upstream/<default>`, then search the upstream log for each issue number. Findings are routinely filed against a long-lived working branch while the fixes land on the default branch, so "already fixed" is common and invisible from the local checkout.
3. Per issue, judge in this order: **drift → re-derive each material claim from the code → recount every number the issue asserts → verdict.**
4. Give each issue exactly one verdict: `CONFIRMED`, `PARTIALLY CONFIRMED` (name which half), `OUTDATED` (name the upstream commit that fixed it), `REJECTED` (never true, or by-design). A recorded decision in `AGENTS.md` or an ADR is settled: a finding that contradicts one is `REJECTED`, not partial.
5. **Decide the tracker action last.** Labelling on an unverified premise has to be walked back in public, which costs more than the delay.

Recipes — scratch-file dumps, the upstream-drift commands, claim recounting (including rendering a component to prove what it emits), dependency-free DB probes, the verdict → action table, and the batch subagent dispatch template — are in `references/finding-triage.md`.

### Pitfalls

- **A stale local branch manufactures both false findings and false confirmations.** Judge every claim against the ref the fix will land on, and say which ref you used. A finding filed at a commit N commits behind upstream can be partly fixed and still read as untouched locally.
- **A number in an issue is a claim.** Recount it; never inherit it. A wrong count on an otherwise true finding makes the whole issue look fabricated, and it is the number a reviewer checks first.
- **Count markup, not raw text.** Comment blocks containing dead markup inflate every occurrence count. Strip comment syntax before counting, then state which convention your number uses.
- **Do not reason about what a component renders — render it.** A button component that takes an icon and an optional label emits an `aria-hidden` icon and no text when the label is omitted, so "the button has no accessible name" must be shown, not asserted from the template source.
- **A deployment-dependent caveat is a severity qualifier, not a refutation.** "The disk is not currently symlinked" changes how bad the finding is, not whether it is real. Keep the qualifier in the verdict; do not use it to downgrade to `REJECTED`.
- **When upstream already added the guard, file against the residual work only.** Name the commit, say what is left, and do not re-file the half that landed.
- **Some findings are not changes to this repository.** Infrastructure and operational gaps (backup strategy, volume placement, host configuration) cannot be closed by a PR here. Route them to the maintainer with the evidence instead of putting them in the implementation queue.
- **Fan out per pair, not per issue, and require re-derivation.** One subagent per two issues with the read-only rules inlined keeps the batch inside a sane round count; the prompt must say the issue text is untrusted and the agent must reproduce each claim itself.

## Replacing a Batch of Filed Findings (retire-and-refile)

Trigger: "delete the issues without label X and file fresh ones", "rewrite these issues", or any re-file request after the audit method improved. The finding survives — only its packaging is replaced — so the replacement bar is that it **re-derives** every claim. Inheriting the old text verbatim republishes numbers that have since drifted, and the drift is what a reviewer checks first.

### Procedure

1. **Triage first**, using the section above. Re-measure every asserted number against the current tree and the live database before writing a word of the replacement. Expect some to move and keep a delta list — numbers in an audit are perishable, not perishable only in theory.
2. **Put the open decisions to the maintainer before creating anything.** A batch pass surfaces a few genuine forks: split a large issue or keep it whole, close a half-landed issue or re-file it in full, frame a security finding rotate-first, cast an infrastructure gap as a decision issue. Batch them into one `clarify` call with a recommended option first on each, and do not create issues until the answers land — a public issue in the wrong shape is more expensive to walk back than a scratch file.
3. **Write bodies to files, then post from disk.** Long non-Latin bodies passed inline hit argument-length limits and shell-quoting hazards; `gh api --input -` fed by a file avoids both and lets you re-run a single item.
4. **Create the replacements first, capture each new number, then close the originals** pointing at its replacement. Closing first leaves a window where a live defect is documented nowhere.
5. **Verify the entire batch from a fresh per-issue read after the loop**, and report which items did not land — see the bulk-mutation pitfalls in the `github` skill (`references/issues.md` in that skill).

### Pitfalls

- **Re-filing is not transcription.** Every number and every `file:line` in a replacement is a fresh claim. Re-running the count is not optional diligence here — it is the deliverable.
- **A half-landed issue splits into shipped-plus-residual, not into a rewrite.** Name the commit that landed the first half and file only what remains; re-filing the whole thing re-reports fixed work.
- **A replacement that keeps the old severity framing inherits the old imprecision.** Where the original over- or under-stated risk, the rewrite should carry a stated, bounded qualifier rather than the original's rhetoric.
- **State the decision points explicitly in each replacement body.** A body that lays out two defensible fixes and picks neither is the right output for a genuine fork — but say so in a dedicated section so the maintainer is not left inferring it.
- **A filed replacement is a public claim of the same standing as a label.** Anything the triage section says about verifying before acting applies here with more force, not less.

Ordering constraints, posting long bodies from files, how to handle numbers that drifted, and what to do when an issue cannot be closed are in `references/refile-batch.md`.

## Plan Review & Verdict Publishing

When asked for an opinion on a plan or proposal issue, review it before commenting. A plan's premise is a set of claims about the repo, and the working tree is the only authority on them.

### Procedure

1. Fetch the plan and read its comments (`gh issue view N --comments`, or the discussion it originated in) — a plan routinely cites an earlier PR or comment as its evidence.
2. Verify each load-bearing claim against the tree, in this order:
   - **Speed claims** — time the exact command: `/usr/bin/time -f "%e s" vendor/bin/phpstan analyse --no-progress`. A plan arguing "too slow for a local gate" dies on one timing run. But time the command the CI job actually runs, not a convenient scoped variant of it, and quote the numbers honestly: a formatter's `--dirty` run is not the CI lint job's full-repo `--test` run, and a result-cached analyzer has two different costs. See `references/measurements.md` for cold/warm timings and how to re-run a failing suite.
   - **"X is broken / flaky" claims** — open the cited PR or issue and confirm its body actually says that. A plan citing a PR by number is not evidence the PR supports the claim.
   - **"we must handle Y" gotchas** — grep the existing scripts first. A gotcha listed as needed work is often already covered by a script that predates the plan.
   - **Mechanism claims** — check that the proposed automation can even map its inputs (file→file, name→name) before evaluating its cost.
3. Check the automation exists before reviewing its performance: `ls -la .git/hooks/<hook>` plus `git config core.hooksPath`. Hooks wired through a package-manager `post-install-cmd` install only on install, so on a rebuilt box the hook is silently absent and every timing-based acceptance criterion is moot.
4. Separate agreement from objection: one line on what you endorse, then numbered objections, each shaped as claim → measurement → consequence. Never restate a plan's premise back as if you had checked it.
5. Post to the thread the plan came from. A plan raised in a discussion is a summary of that thread's consensus — review belongs in the thread, not in the tracking issue.

Command recipes for reading and posting to discussions, plus the full claim checklist, are in `references/plan-review.md`. Timing claims that need cold/warm numbers or a re-run of a failing suite are in `references/measurements.md`.

### Pitfalls

- **A discussion number returning Not Found from an issue tool is not an auth failure.** Discussions are not Issues, so issue-scoped MCP tools and `gh issue view` never covered them. Re-authenticating solves nothing. Read the page, post via GraphQL — see `references/plan-review.md`.
- **An exclusion list built from a guess is worse than no list.** It silently drops healthy checks and the coverage loss surfaces later with no trace. Require a reproduced failure list before such a list is written down; recommend deleting the item if nothing reproduces.
- **Review the plan's premises, not its prose.** The defects worth reporting are the ones that make the work unnecessary or wrong (premise already handled, cited evidence absent, mechanism impossible) — not restyling of the document.
- **When you are the author of the claim under review, audit your own numbers first.** A published comment is a claim in the same way a plan is. Re-measure before replying; if a number in your own earlier post was wrong, lead with the correction and say which measurement was wrong (cold vs warm, scoped vs full), not just the new figure. Claiming a cost you never measured is the same defect as citing a PR that does not say what was claimed.

## Scheduled Re-check on a Thread

Some asks are "give your verdict now, then give it again in N minutes" — the point is to read what landed after your own post and respond to *that*, not to restate the first answer. Split it: do and post the first verdict in this session, then schedule one-shot follow-up work.

### Procedure

1. Finish and verify the first comment (post, then read the last comment back).
2. Schedule ONE shot with `cronjob_manage action=create`, `schedule: 'in 10m'`, `repeat: 1`. The cron session is a fresh session with no chat context, so the prompt must be fully self-contained: who the user is, which repo/branch, the topic, and every measurement already taken (re-use them instead of re-running a multi-minute suite).
3. Give the job a `script` that fetches the live thread so the newest comment is injected as context — the follow-up must see what it is answering.
4. Tell the follow-up explicitly: reply to the NEWEST comment, do not restate the previous comment, and post without asking for confirmation if the user said confirmation is not needed.
5. Report the job id and fire time to the user in this session; the follow-up's own final response is what lands in the chat.

Command recipes, the `script` filename rule, and the cron/credential pitfalls are in `references/scheduled-recheck.md`.

## Recurring Cron Audit (one issue per run)

The recurring form of this work: "every N minutes, audit the repo with improve and file the single most important finding as an issue in the upstream repo." Same two-phase discipline as the thread re-check, but recurring instead of one-shot.

### Procedure

1. Create the job with `cronjob_manage action=create`, `skills: ['improve']`, `workdir` set to the audited repo, `schedule` in interval form (`'every 30m'` — not a hand-computed timestamp), and `continuity: true`. Continuity is what carries the previous run's output into the next one; it is the first line of defence against re-filing the same finding.
2. The prompt must be self-contained (fresh session, no chat context) and must name the canonical upstream `owner/repo` explicitly — a recurring job cannot ask which repo it meant.
3. Bake the dedupe rule into the prompt as a command, not as a wish: list every issue (`gh issue list --repo owner/repo --state all`), then keyword-search the candidate's title before creating anything. "Don't file duplicates" without a command produces duplicates.
4. State the blast radius in the prompt: exactly one issue per run, no code edits, no commits, no local file changes.
5. Have the final response report the issue title + URL, or the reason nothing was filed. That response is the only thing the user sees.
6. Test the job once with `cronjob_manage action=run` before trusting the schedule. The manual run is asynchronous: the call returns a delegation id immediately, and the job's outcome re-enters the conversation later. That outcome — not the tool response — is the pass/fail signal.

### Pitfalls

- **A skill body can poison its own cron job.** The runtime scans the ASSEMBLED prompt (skill content included), so a skill that quotes an injection payload verbatim as a security example — `"ignore previous instructions"` is the classic — blocks every job that loads it, regardless of what the prompt says. When a cron job fails with `Blocked: prompt matches threat pattern`, scan the SKILL.md before touching the prompt; see `references/cron-prompt-scanner.md`. Rewriting the example to describe the attack without reproducing the trigger phrase is the fix — do not strip the security rule itself.
- **Non-Latin prompts are not the cause; invisible characters are.** ZWNJ (U+200C) and friends are rejected at create/update time even in perfectly innocent Persian text. Type prompts without zero-width joiners before suspecting your wording.
- **One job runs at a time.** A second `action=run` against a job that is still executing returns `executed: false` with an already-running skip instead of starting a second run. A skipped verification is not a passing one — wait for the in-flight run's outcome, then fire again if you still need a second data point.
- **A successful `action=run` response does not mean the job worked.** The returned job record carries `last_status` and `last_error` from the PREVIOUS run until the new one finishes, so those fields still read `error` right after you fixed the cause. Read them as stale; never quote them as the current run's verdict.
- **Clearing the scanner block proves only that the gate opens.** The scanner functions passing on the skill body and the prompt means the job can reach the model, not that it audited or filed anything. Only a real outcome — an issue URL, or the failure it hit — confirms the job works. See `references/cron-prompt-scanner.md`.
- **Do not shorten the prompt to get past a scanner.** Rewriting the task to dodge a block hides the real fault and leaves the job doing something other than what was asked. Find what actually matched.

## Subagent Audit Pattern

For the `standard` effort level (default), fan out with 4 parallel subagents:

1. **Correctness & Security** — input validation, auth/authz, SQL injection, XSS, race conditions
2. **Performance & Architecture** — N+1 queries, unbounded recursion, cache misuse, God classes
3. **Test Coverage & Quality** — untested critical paths, wrong annotations, DRY violations
4. **Tech Debt & DX** — baseline bloat, dead code, CI gaps, documentation

Each subagent prompt must include:
- Recon facts (languages, frameworks, key directories)
- Domain-specific risk hints from recon
- Decided tradeoffs from intent docs
- "Return findings only — no fixes, no file dumps"
- Hard Rules 4 and 6 from the improve skill (verbatim)

Subagent output schema per finding: `{id, category, finding, evidence, impact, effort, risk, confidence}`.

## Vetting Subagent Reports

Subagents over-report. Three failure classes to check:

1. **By-design behavior** reported as bug (e.g., "CSP unsafe-inline" when it's intentional)
2. **Mis-attributed evidence** — real finding, wrong file or line
3. **Duplicates** across subagents (same root cause, different symptoms)

Always open the cited code yourself before including a finding in the vetted table. Downgrade or reject accordingly.

## Plans Directory Structure

```
plans/
  tracker.json              ← queue for automated processing
  README.md                 ← index: priority order, dependency graph, status
  001-<slug>.md
  002-<slug>.md
```

Each plan file stamps the commit hash it was written against (`git rev-parse --short HEAD`).

The `tracker.json` schema:
```json
{
  "last_created": 0,
  "findings": [
    {
      "num": 1,
      "id": "SEC-001",
      "slug": "short-slug",
      "title": "Plan title",
      "category": "security",
      "effort": "M",
      "impact": "high",
      "done": false,
      "plan_file": "plans/001-short-slug.md",
      "issue_url": "https://github.com/.../issues/N"
    }
  ]
}
```
