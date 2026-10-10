# Replacing a Batch of Filed Findings

Depth for the "Replacing a Batch of Filed Findings" section of SKILL.md. Covers retiring filed issues and publishing rewrites of them.

## The ordering constraint

Publish replacements **before** closing originals. A closed original plus a not-yet-created replacement is a window where a live defect is documented nowhere — and defects in this class get acted on asynchronously by whoever reads the tracker first.

Within one batch: create all replacements, capture the returned numbers, then close each original with a comment naming its own replacement specifically (not "see the new issue"). A close comment that does not name the number is a dead end for the next reader.

## Post long bodies from files

Long non-Latin bodies passed as inline arguments hit argument-length limits and shell-quoting hazards. Write each body to a file, then post from disk:

```bash
gh api repos/<owner>/<repo>/issues -X POST --input -   # payload JSON on stdin
gh issue create --repo <owner>/<repo> --title '...' --body-file body.md
```

This also makes a single item re-runnable without regenerating the rest, and keeps a scratch copy of exactly what was published for diffing later.

## Numbers drift; treat them as claims

The replacement body asserts counts as public claims, so re-measure rather than carry forward. Between two passes on the same finding, expect movement in both directions — row counts shift as data changes, and "how many units are deletable under the current guards" moves when unrelated rows land. Keep the delta list; it goes in the close comment, because it is the answer to "why did the wording change?"

Quote only reproducible queries in the published body, and let the reader re-run them. A pasted number with no query is an assertion the next reviewer has to take on trust.

## Genuine forks are the maintainer's call

Some items in a batch carry a real product or architecture decision that the code cannot settle. Common shapes:

- **A large issue that spans two independent fixes** — split, or keep whole and accept a slow PR.
- **An issue whose first half shipped** — close as fixed and file the residual, versus re-file the whole thing.
- **A security finding where prevention and remediation are different urgencies** — frame rotate-first rather than as a cleanup chore.
- **An infrastructure gap no PR can close** (backup, host configuration) — cast as an explicit decision issue with the option set and a recommendation, not as an implementation task.

Batch these into one `clarify` call, recommended option first on each, and create nothing until the answers land. Do not resolve a fork unilaterally to keep the batch moving; a public issue in the wrong shape costs more to walk back than a scratch file.

## Label discipline

If the user says not to label, label nothing — including the replacements. Consistency with the instruction matters more than any house convention for the batch.

## When an issue cannot be closed

Some issues are authored by an account the current token cannot close; the write token has the repo scope but not admin/author rights. Posting the close comment usually succeeds while the state change fails with a permissions error — so the two must be issued and checked **separately**, and the leftover handed back to the user as the exact command to run. Report which items did not land; a partially executed batch reported as complete is worse than one reported as partial.
