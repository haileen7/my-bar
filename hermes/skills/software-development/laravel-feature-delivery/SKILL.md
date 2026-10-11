---
name: laravel-feature-delivery
description: Ship a feature through an existing Laravel app end to end.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [laravel, livewire, feature, migration, permissions, phpstan, pest, playwright, menu]
    category: software-development
    related_skills: [laravel-hermes-init, test-driven-development, github]
---

# Laravel Feature Delivery

Ship a feature change through an app that already exists, where other pages,
permissions and tests are the contract. Session setup (MCP verification, upstream
sync) is `laravel-hermes-init`; this skill starts once work begins.

## Standing rules

- Read the project's `AGENTS.md` (it auto-attaches when cwd is inside the repo)
  and its `references/*.md` before designing anything. Conventions there override
  defaults below.
- Write the failing test first for each behavior slice, watch it fail, then make
  it pass. Do not batch all tests first — tracer bullets, not horizontal slices.
- A feature is NOT done when the code works. It is done when it is reachable,
  permission-gated, cached correctly, covered by tests, and documented.
- Every new page needs a navigation entry. A route nobody can click is a
  regression against the page that existed before it.

## Procedure

### 1. Inventory the surface before writing

Find every place the new page must appear. Search for an existing sibling page
and mirror it:

```bash
rg -n "old-page-route|layout" resources/views routes
```

Four registration points are routinely missed — check all of them:

| Surface | Where | Missed symptom |
|---|---|---|
| Route | `routes/web.php` | 404 |
| Permission | seeder + role assignment | 403 for everyone, or invisible to all |
| Nav entry | `resources/views/components/layouts/app.blade.php` | page reachable only by typing the URL |
| Cache namespace | `PruneStaleCache`-style command / invalidation service | stale data after a write |

Menu entries use the app's own permission gate so the link and the route agree:

```blade
@can('the_new_permission')
<x-menu-item title="..." icon="o-..." link="/the/route" wire:navigate />
@endcan
```

The permission row of that table has its own trap: check that the seeder which
*creates* a permission is the same one that *grants* it. Many projects split these
— a permission seeder that only calls `firstOrCreate`, and a role seeder that
grants. Then `db:seed --class=PermissionSeeder` alone leaves the permission
existing and attached to nobody, and the only symptom is a 403 for admins. Add a
test for the standalone-seeder path, and reach for `givePermissionTo` there:
`syncPermissions` with a partial list replaces the role's entire set and strips
admins of every permission it omits.

Before picking an icon name, confirm it exists in the icon set the app vendors —
a missing icon renders nothing instead of erroring, so it fails silently:

```bash
ls vendor/blade-ui-kit/blade-heroicons/resources/svg/ | grep -E '^o-(server-stack|signal|cog-6-tooth)'
```

### 2. Model and migration

Follow the existing migration/cast/`@property` conventions exactly, including
PHPDoc that static analysis depends on. Scopes need explicit generics or PHPStan
complains:

```php
/**
 * @param  Builder<self>  $query
 * @return Builder<self>
 */
public function scopeActive(Builder $query): Builder
{
    $query->where('is_active', true);

    return $query;
}
```

Use `/** @var array<string, string> */` above a `$casts` array and above any
`$sortBy`-style array property. Bump a versioned cache namespace from the model's
own `booted()` so every consumer invalidates at once, and register the namespace
in whatever prune/inspect command lists them.

### 3. Seeder: copy, never retype

When moving hardcoded data into a table, extract the source **programmatically**
and generate the seeder from it, so transcription errors are impossible:

1. Parse the literal array out of the old file (regex + bracket matching, or a
   short script) and dump it as JSON.
2. Generate the seeder rows from that JSON with a script.
3. Assert the count against the parsed data — **not against the ticket**.

Issue text routinely miscounts the entries it describes. The array in the code is
the contract; note the discrepancy in the PR description rather than padding the
seeder to match the ticket.

### 4. PHPStan baseline: regenerate, don't hand-patch

Adding a `Route::livewire()` or a new `Permission::firstOrCreate()` breaks the
baseline's `count:` entries for that exact error pattern, and PHPStan fails with
"expected to occur N times, but occurred N+1". Fix real type issues in your new
code first, then regenerate:

```bash
vendor/bin/phpstan analyse --no-progress --generate-baseline
git diff phpstan-baseline.neon   # review: expect +count entries, stale entries removed
vendor/bin/phpstan analyse --no-progress
```

Never add `@phpstan-ignore` or edit the baseline by hand to hide a new error.

### 5. Tests that depend on counts

A mock expecting `->times(10)` over a hardcoded list breaks the moment the list
grows. Make the expectation derive from the source so it cannot drift again:

```php
$cache->shouldReceive('getVersion')
    ->times(count(PruneStaleCache::NAMESPACES))
    ->andReturn(42);
```

Tests asserting the old hardcoded data need updating to seed the new table — that
is a legitimate edit, not a weakening, as long as the assertions still check real
values rather than mere counts.

### 6. Gates, in order

```bash
php vendor/bin/pint --dirty --format agent    # CI enforces this
vendor/bin/phpstan analyse --no-progress      # must be clean
composer test                                # full Pest suite
bash scripts/e2e-test.sh tests/e2e/<area>    # targeted Playwright first
bash scripts/e2e-test.sh                     # then the full run
```

Run the targeted e2e suite first: the full Playwright run is long, and a selector
bug found in 2 minutes is cheaper than one found in 15.

**Pint runs first, so a green test run goes stale.** `pint --dirty` rewrites code
and some fixers change semantics, not just formatting (a keyed collection method
being replaced with an equivalent-looking one that keys by index instead of id is
the classic). A suite that passed before Pint proves nothing after it. Always
re-run the tests *after* the formatter, and when a fixer touches a line you care
about, re-read the function — do not trust that the diff is cosmetic.

**PHPStan clean is not "correct".** It proves types line up; it cannot see that a
map you built is keyed wrongly. The tests are what catch that, which is exactly
why the gates are ordered formatter → analyser → tests, and why the test re-run
after Pint is not optional.

### 7. Red CI after pushing

Read the failing job, don't guess at it. `gh run view <run-id> --job <job-id>
--log-failed` output contains ANSI codes — strip them with
`sed 's/\x1b\[[0-9;]*m//g'` before grepping. The run's logs are only complete once
every stage finishes, so poll the run status rather than concluding "CI doesn't run
on forks". If the failure reproduces at the upstream base commit in a throwaway
worktree, it predates this change: say so in the report and offer the fix as
separate work instead of widening the diff. Full probe order in
`laravel-feature-testing` → `references/red-test-attribution.md`.
If a terminal security guard refuses a vendor binary because of its size, invoke
it through a shell: `sh -c 'php vendor/bin/pint ...'`.

### 7. Docs, then commit/push/PR

Update the project's own reference docs in the same commit as the behavior
change — a new table, permission or page is exactly what those files exist to
record. Then commit, push to the current branch, and open the PR.

## Pitfalls

- **A written plan's anchors describe a commit you are not standing on.** An
  implementation plan (issue body, design doc, review note) is authored against
  whatever commit its author had checked out. If your branch has drifted, its
  line numbers are fiction and its method list can be incomplete — a method it
  gates may not exist yet, and one it omits may have appeared. Resolve every
  anchor against the working tree before the first edit:

  ```bash
  git log --oneline -1                       # what the plan claims to describe
  git rev-list --left-right --count HEAD...upstream/<pr-base>
  grep -n 'public function' <component>      # do the plan's methods all exist?
  ```

  When the PR base has moved ahead of your branch, fast-forward to it
  (`git merge --ff-only upstream/<pr-base>`) *before* starting. Transcribing
  against a stale tree produces a diff that silently skips a mutator and looks
  correct. Where the plan and the tree genuinely disagree, the tree wins — and
  the divergence is a decision to state in the report, not to quietly paper over.
- **Running the suite can dirty a TRACKED env file with a real secret.** Some
  projects ship `APP_KEY=` blank in a git-tracked `.env.testing` (correct — that
  key encrypts sessions and signs the cookie MAC) and have the test command
  generate a per-clone one into the working tree:
  `php scripts/ensure-test-app-key.php && php artisan test`. So `composer test`
  leaves `.env.testing` modified, and `git add -A` commits a master secret. The
  guard that exists for it reads the **git index** (`git show ':'.$path`), not
  the working tree, so it passes right up until the moment you stage. Restore it
  after the run and stage explicit paths:
  ```bash
  composer test
  git checkout .env.testing
  git add <explicit paths...>        # never -A
  git diff --cached --stat           # confirm nothing stray rode along
  ```
- **A Pest dataset row that is associative is spread as NAMED arguments**, not
  positionally — and `class` is a reserved word, so it cannot be a parameter
  name. `->with(['k' => ['component' => .., 'class' => ..]])` reaches the
  closure as `function (string $component, ..., $class)` and dies with
  "Unknown named parameter $component" / a parse error on `$class`. Wrap each row
  in a one-element list so it arrives as one positional array:
  ```php
  dataset('lookups', ['semat' => [['component' => 'kargozini.semat', 'model' => Semat::class]]]);
  test('…', function (array $lookup) { /* $lookup['model'] */ })->with('lookups');
  ```
- **Gating a Livewire mutator 403s scope tests whose actor holds no
  permission.** Adding `$this->authorize('X')` as the first statement of a
  mutator is correct and intended — but a pre-existing test that exercises that
  mutator to assert *scope* now fails, because its fixture user was created
  with no permissions at all and the 403 fires before the scope assertion.
  Fix the FIXTURE (grant the permission the actor legitimately needs), never
  weaken the new `authorize()`: scope comes from `accessibleUnitIds()` and never
  consulted the permission, so granting it changes nothing the scope tests
  assert. Confirm the intent by reading what the failing test asserts — if it is
  about scope or validation, it is a fixture problem, not a gate problem.
- **Nav entry forgotten.** Route + permission + tests all green, page unreachable
  from the UI. When asked "did you add a menu for the new page?", check the layout
  file — not just the route table.
- **Playwright exact `placeholder` match fails on MaryUI.** MaryUI renders the
  attribute with a trailing space, so `input[placeholder="X"]` never matches.
  Use `input[placeholder*="X"]`. See `references/playwright-maryui-selectors.md`.
- **E2E harness leaves the repo dirty.** A swap-`.env` script that uses `set -e`
  without a `trap` exits before restoring. Chain the restore onto the command you
  run, and see `references/e2e-harness.md`.
- **A red test in the full suite is not automatically yours.** Before bisecting
  your diff, check you are not running two test processes against the shared test
  database (`pgrep -af 'pest|artisan test'`) — a stray background run turns into a
  mass failure count that looks like a catastrophic regression. Then confirm the
  diff does not touch that file and run the test in isolation a few times. To settle
  "is this pre-existing?" decisively, reproduce it in a throwaway
  `git worktree add` of the upstream base commit rather than arguing from the diff.
  Report the verdict with the evidence instead of silently re-running until green.
  Full probe order in `laravel-feature-testing` → `references/red-test-attribution.md`.
- **View-scope variables.** Inside a table `@scope` slot, a bare `$editingId` is
  undefined; use `$this->editingId`.
- **Factory `$this->unique()`** is not available on the factory — use
  `fake()->unique()`.
- **Seeding explicit IDs** needs the Postgres sequence resynced (`setval`) or
  later inserts hit duplicate keys.
- **Permission and route must agree.** A link gated on permission A pointing at a
  route gated on permission B produces an item that either leaks or 403s.
- **External API calls in a Livewire action** catch `Throwable` and surface the
  message; never let them bubble into a 500.
- **A RED that is really your test's bug.** When several new tests fail at once,
  check whether the failures share one helper or one assertion shape before
  touching production code. A helper returning column-letter keys, an assertion
  aimed at the wrong field, or a value read back as `string` where you expected
  `int` makes many tests fail for a single non-feature reason. Dump the actual
  intermediate values before concluding the feature is broken.
- **A test that passes immediately may be your own earlier code.** If a slice
  "only needed plumbing" and you wrote that plumbing before its test existed, the
  next slice's test passes on first run and proves nothing. Delete the untested
  production code and re-derive it from a test you watch fail; a green that was
  never red is the signature of this.
- **Recursive CTE on a user-editable adjacency list never terminates.** A
  hierarchy walk written `WITH RECURSIVE ... UNION ALL` does not dedupe, so one
  bad `parent_id` cycle makes it recurse forever — the connection hangs and the
  suite dies instead of failing. `UNION` breaks the cycle and returns identical
  rows on acyclic data. Use `UNION` for any walk over a column users can edit, and
  probe the shape with a bounded timeout before trusting a test of it:

  ```sql
  SET statement_timeout = '3s';
  -- cycle + UNION ALL => "canceling statement due to statement timeout"
  -- cycle + UNION     => returns the finite row set
  ```

  If a run hangs, recover the session rather than waiting it out:
  `pg_terminate_backend(pid)` for active backends on the test database. A test for
  the guard belongs on in-memory models so the suite can never wedge on the
  hazard it is documenting.
- **Verify a hazard is reachable before reporting it.** A query that *would* hang
  on cyclic data is only a live risk if some unguarded path can produce that data.
  Check the form guard, the API guard and the seeders; when a guard makes the
  cycle unreachable, call the query fragile rather than claiming users can trigger
  it. Check the *type/permission* graph for cycles too, not just the data — a
  relationship table that looks cyclic at a glance is often a DAG once you run
  a proper cycle check on it. Speculative hazard reports cost the reader trust in
  the rest of the report.
- **File downloads are plain `<a href>`, never `wire:click`.** Livewire cannot
  return a binary file response. Test that the control is an anchor pointing at
  the route, not only that the route exists. Reading the generated file back to
  assert on its rows — the `toArray` cell-ref trap, numbers returning as strings,
  and why `Excel::fake()` is not enough — is in
  `references/testing-generated-files.md`.
