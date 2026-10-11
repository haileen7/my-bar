---
name: laravel-feature-testing
description: "Add/fix Laravel tests, run quality gates, verify incoming PRs."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [laravel, livewire, pest, playwright, testing, phpstan, baseline, quality-gates]
    category: software-development
    related_skills: ["test-driven-development", "systematic-debugging", "requesting-code-review", "laravel-hermes-init"]
---

# Laravel Feature Testing

Writing tests for a new or changed Laravel + Livewire feature, repairing tests that
change broke, and running the gates that must be green before the work ships.

**This skill vs test-driven-development:** that one enforces RED-GREEN discipline;
this one carries the Laravel mechanics — fixtures, Livewire interaction, browser
specs, the static-analysis gates that run beside the suite, and the workflow for
reproducing an incoming change's suite in isolation.

**The project's AGENTS.md / CLAUDE.md overrides this skill** on commands, seeders,
and which tiers exist. Read it first; this skill carries only what it does not
document.

## When to Use

- "Add tests" / "fix the tests" for a feature
- A change touching routes, permissions, seeders, Blade components, cache namespaces,
  or factories — i.e. anything other tests may have pinned
- Before commit, when a gate must be shown green with real numbers
- **Verifying someone else's work** — "check PR #N", "is this change correct",
  "does this patch actually work" — where you must run their suite rather than your
  own working tree. See `references/verifying-an-incoming-change.md`

## Always-On Rules

1. **See it red first.** Write the test, run only that file, and watch it fail for
   the reason you intended. A test never observed red proves nothing — a passing
   test you just wrote may be passing because it asserts nothing about your change.
2. **Prove the test database exists before launching a suite.** Check it as a
   one-line query, and create it (from the project's own spatial/extension
   template) if absent. A missing test database does not abort the run at startup —
   it surfaces as a *subset* of tests failing with `does not exist` while the rest
   pass, which reads exactly like an infrastructure hiccup and invites a pointless
   re-run. Verify presence once, then the suite's results mean what they say.
   Create it through PDO using the credentials already in the project's env file
   when the `psql` CLI would otherwise demand an interactive password:
   `new PDO('pgsql:host=…;dbname=postgres', $user, $pass)` then
   `CREATE DATABASE "<name>" OWNER "<user>"`.
2. **Two tiers, both green.** The framework suite (fast, full) and the browser spec
   suite (slower, environment-swapping) exercise different layers; a feature is not
   done on one tier. Report both with their pass counts.
3. **Fixtures come from seeders/factories in the test**, never from whatever is
   manually seeded in a dev database — CI has none of that.
4. **Repair dependents before declaring done.** Any identifier you changed — a
   hardcoded array, a permission name, a route, a cache namespace, a seeded table —
   gets grepped across the *whole* test tree *and* the browser spec tree, not just
   the file that failed.
5. **Do not run the browser tier concurrently with the framework tier.** Browser
   suites commonly swap an env file or rebuild a DB; two suites sharing those
   resources corrupt each other's run.
6. **One test process at a time per test database.** Every framework-tier run
   shares the one `DB_DATABASE` from `phpunit.xml` and mutates it, so a background
   full suite plus an isolation loop (or a worktree run) will corrupt each other.
   Before believing any mass-failure count, run `pgrep -af 'pest|artisan test'`,
   stop the strays, and re-run once alone. See
   `references/red-test-attribution.md`.
7. **A test is evidence only if it fails for the stated reason.** Four ways a test
   lies: the stored data it needs is rewritten before it is ever persisted, so it
   exercises a state the application cannot reach; a multi-column match is
   already satisfied by a sibling field, so the assertion never reaches the code
   under test; it asserts only the **first** render of a stateful component, never
   the state after a round-trip; or the event/callback the bug lived on is
   **unreachable from how the test drives the code** — an assertion that can never
   fail is not a regression guard, it is decoration. Prove the fixture survives a
   write round-trip before writing the test, make every non-target field in it
   non-matching, drive one `->call(...)` for any stateful component, and trace the
   bug's own code path to confirm the test enters it. See the pitfalls.
8. **Assert the exact rendered value, and assert absence too.** A presence-only
   match (`toContainText('نفر')`) passes when the value is wrong (`0 نفر`), and
   an assertion that a negative marker exists (`خالی`, "empty") passes vacuously once a
   bug makes that marker appear on every row. Pin the number, and for negative
   markers assert both presence on the empty case and absence on the populated one.
9. **Prove a claimed bug yourself before reporting it.** A probe test whose expected
   value is hardcoded (rather than read from the database) will happily certify broken
   code as correct when a seeder or factory made the real count differ. Print the DB
   count in the same run; a `BEFORE` mismatch means the probe is wrong, not the code.
10. **Never assert how a third-party library behaves from memory — read the shipped
   source.** Claims about a library's internals (which classes override a method,
   what a flag disables, whether a version string means what it looks like) are
   cheap to make and easy to get wrong, and a wrong one published as a review
   finding costs more credibility than a missed defect. Locate the vendored or
   bundled file, grep the actual method, and quote the line you relied on.

## Procedure

1. **Find the seam.** Locate the existing test class or spec covering the same
   component or page and mirror its setup — base `TestCase`, `RefreshDatabase`,
   shared seed helpers, existing fixtures. A test that re-invents setup diverges
   from its neighbours and rots first.
2. **Seed the authorization layer deliberately.** With Spatie-style permissions,
   granting a permission to a *role* happens in the role seeder (usually by syncing
   every permission). Seeding only the permission seeder leaves roles without it, so
   an authorization assertion fails while the permission visibly exists — seed the
   seeder that wires roles, and assert with `hasPermissionTo()`, not `can()`.
3. **RED.** Run only the new file and confirm the failure:
   `php artisan test tests/Feature/ThingTest.php` — or for a spec,
   `npx playwright test tests/e2e/thing.spec.ts`.
4. **GREEN.** Make the smallest change that satisfies it.
5. **Repair dependents** (rule 4 above), then re-run the touched files.
6. **Gates in order:** formatter on changed files → static analysis → full suite →
   browser suite. Each gate's output is evidence; keep its tail.
7. **Isolate before theorizing.** A failure seen only in a full/randomised run must
   be reproduced alone before any code is read: if it passes alone it is order- or
   seed-dependent, and the next probe is re-running the suite with the seed the
   runner printed — not a source-code hypothesis.
8. **Report numbers, not adjectives**: counts of passed/failed per tier, plus the
   formatter and static-analysis verdicts.

## Pitfalls

- **"Only some tests failed with a missing-database error" is a database that
  genuinely does not exist, not a flake.** Those two explanations are
  indistinguishable from the failure text alone, and re-running the suite to "rule
  out a hiccup" burns minutes to reach the same state. List the test databases
  first; a missing one is fixed with the create-database step, an existing one is
  the real race worth investigating.
- **A Livewire component that is correct on first render can be wrong on every
  render after.** Public properties holding Eloquent models/collections are
  rehydrated from primary keys only, via a bare `select *` — so `withCount`,
  `withSum`, eager loads, and appended attributes are all gone after the first
  interaction. A `?? 0` fallback turns that into "every row silently reads as
  empty" rather than an error. Keep computed data in a scalar `array $counts`
  keyed by id instead. Full mechanism, inspection recipe, and the before/after probe
  in `references/livewire-hydration.md`.
- **A test that only ever asserts the first mount of a stateful component proves
  nothing about its behaviour.** `Livewire::test('x')->assertSee('1 نفر')` with no
  `->call()` never crosses a dehydrate/rehydrate boundary, so the whole class of bugs
  above is invisible to it. Drive one interaction first.
- **A "negative" assertion is vacuous once the bug makes the marker universal.** If a
  defect causes the empty/vacant badge to render on every node, then
  `assertSee('خالی')` on "the empty unit" still passes. Assert the marker is absent
  on the populated row, or assert the exact total count of markers.
- **A model write hook can make the "bad data" unreachable.** Before writing a
  regression test that persists malformed stored data — mixed script, an
  unnormalised character, a legacy spelling — check whether the model already
  rewrites it on write: `static::saving` / `creating` hooks, `Attribute` mutators,
  casts, observers. A `saving` hook that normalises a name makes that name
  impossible to persist, so the test can only fail through a path the app never
  takes and any "fix" for it is dead code. Reachability is then limited to rows
  written before the hook, imports, or fixtures that bypass the ORM — decide which
  of those you mean before writing the test, and say so in the report. Prove it
  with a round-trip (`Model::create([...])`, then read the column back and inspect
  its code points); do not reason from the attribute setter alone.
- **Never insert probe rows with `DB::table()->insert()` when the question is
  about what the application stores.** The query builder skips model events and
  accessors, so it will happily write states no code path can produce and you will
  "discover" a bug that does not exist. Use the model, or `Model::insert()` if you
  need a single insert, and re-read through the model to confirm what landed.
- **A new test that passes against the unfixed code is not a passing test.** It is
  passing through a branch you did not mean to exercise: an `OR` group whose
  sibling column also matches, a component that renders the value in a second
  place, or a default that already satisfies the assertion. Make every non-target
  field in the fixture non-matching so only the code under test can satisfy it, and
  pin values the factory draws at random — localized names collide with localized
  search terms often enough to matter. If a test still cannot be made to fail
  without the fix, it documents existing behaviour: label it as such or drop it
  rather than filing it as a regression guard.
- **`syncPermissions` replaces the role's whole set; it does not add to it.**
  Calling it with a partial list inside a permission seeder strips the role of
  every permission left out, so adding one new permission to a seeder can lock
  admins out of unrelated sections. Use `givePermissionTo` for a targeted grant
  and reserve `syncPermissions` for the seeder that deliberately owns the role's
  entire set.
- **Pest spreads an associative dataset row as NAMED arguments, and `class` is a
  reserved word.** `dataset('k', ['a' => ['component' => 'x', 'class' => Foo::class]])`
  fails with `Unknown named parameter $component` — or cannot be destructured at
  all — because Pest spreads the row's keys as argument names. Wrap each row in
  a one-element list to pass one positional array:
  `['a' => [['component' => 'x', 'model' => Foo::class]]]`, then
  `function (array $lookup)` in the test. Keeping every row's shape identical
  also keeps every closure signature identical.
- **Pest `--filter` needs the method name.** A humanized name containing spaces
  (`--filter "filter unit applies"`) prints `INFO No tests found.` and exits 0 —
  a silent no-op that reads as a green run. Use `--filter=test_filter_unit_applies`.
  When looping one test to measure a flake rate, match on a stable substring, not a
  column-aligned pattern: Pest pads its summary differently for passing and failing
  rows, so a fixed-width `grep` silently drops half the results.
- **PHPStan baseline churn is not an app error.** A baseline entry carries an
  `ignore.count`; your code starting to hit an ignored error shifts the count
  (`expected N, found N+1`), and code that stops hitting it becomes an *unmatched*
  entry. Fix the real typing errors your change introduced first, then regenerate
  the baseline once with the repo's baseline command (`composer phpstan-baseline`
  when present) and re-run to confirm zero errors. Regenerating to silence an error
  your own change added buries a regression instead of baselining legacy debt.
- **Static-analysis blind spots are style, not permission.** Without a package
  bridge, analyser config often hides model static calls, so `Model::find()` or
  `Model::firstWhere()` lints clean while `Model::query()->find()` is the typed,
  analyser-visible form — use the query-builder form when a baseline entry or a
  `Model::query()` return-type error points at it. Likewise annotate `$casts` with
  an explicit `array<string, string>` var tag when the analyser insists on `mixed`.
- **Component state inside Blade scope/slot closures must be read via `$this->`.**
  Inside `@scope(...)` / `<x-slot>` callbacks the closure captures view scope, not
  the Livewire component's public props, so a bare `$editingId` raises
  "Undefined variable" at render time. Always `$this->editingId`.
- **Attribute selectors must survive component whitespace.** Blade UI components pad
  rendered attributes (a placeholder can come out with a trailing space), so an
  exact `input[placeholder="…"]` match silently fails while the field is on screen.
  Match by substring — `input[placeholder*="…"]` — and use role+name queries with
  `exact: false` for labels.
- **Factory builders don't proxy `unique()`.** `$this->unique()` on a factory builder
  throws `BadMethodCallException`; call `fake()->unique()->…` in factory attributes
  instead.
- **A red browser run often skips its own teardown.** Wrapper scripts written with
  `set -e` exit at the first failing command, leaving a swapped `.env`, a stale dev
  server, or a borrowed port behind. Wrap the run in your own restore/cleanup
  (copy the original env back, kill the test server) so the developer's environment
  is intact whether the suite is green or red.
- **An assertion the bug's code path cannot reach is decoration, not a guard.**
  Symptom: the test passes, and it would also pass on the unfixed code. The usual
  cause is that the test drives the code through a branch that skips the machinery
  the defect lived in — asking a library to animate with animation explicitly
  disabled, so the event that carries the failure is never emitted. Before
  accepting any regression test (yours or someone else's) as evidence, trace the
  reported failure to the exact branch that produces it, then check the test
  actually enters that branch. If it cannot, say so plainly and propose the driver
  that would (e.g. trigger the UI affordance rather than calling the API with the
  flag off) — do not let a green run stand in for a guard that does not exist.
- **A vendored library's version is not the first `version` string in the file.**
  Bundled/minified builds often carry unrelated version numbers — a WMS/tile
  parameter default looks exactly like a library version. Find the export
  assignment that names the library (`<ns>.version=`) and read the file that ships
  with the app; a vendored lib is frequently absent from `package.json` entirely, so
  the manifest cannot confirm it. Pin the version in the review so the reader knows
  which semantics you checked.
- **A route gate on a Livewire page is not a gate on its buttons.** Route
  middleware runs on the page load only; follow-up actions post to
  `/livewire/update`, a different route, so a revoked holder keeps full CRUD on
  an open tab. Any mutator that writes must re-authorize as its first statement
  — and `catch (\Exception $e)` around a delete swallows that 403 into a friendly
  toast unless a specific `AuthorizationException` re-throw clause precedes it.
  Spatie returns `false` (not an exception) for an unknown permission name, so a
  typo in `authorize('…')` fails closed: every negative test passes while the
  page is locked for legitimate holders. Always pair the negatives with positive
  tests and a static permission-name sweep. Full recipe in
  `references/livewire-mutator-authorization.md`.
- **A batch patch applied to sibling files reports success per file while
  writing the same broken text into all of them.** One templating mistake —
  notably a `{{` that only escapes inside a Python f-string prefix, not in the
  plain second segment of a `+` concatenation — replicated across N components
  is N identical parse errors discovered late. Run a syntax check over *every*
  file in the batch, not the first one.
- **A tracked env file with a blank `APP_KEY` makes every test that renders a
  view fail identically, and the failures name encryption rather than the key.**
  Repositories that guard against leaking a real key blank it in the tracked env
  files and generate one in CI; locally you get `MissingAppKeyException` /
  `ViewException: No application encryption key has been specified` on every
  test that boots a session, while pure static-scan tests still pass. Read the
  tracked file's key length rather than guessing from the failure, and export the
  key from the gitignored working env file for the session instead of writing a
  real key back into a tracked file — a guard job in CI fails the build on exactly
  that. `export APP_KEY=$(grep '^APP_KEY=' .env | cut -d= -f2-)`.
  **And the repo's own test script writes it back for you.** Projects that solve
  this with a generate-if-blank helper call it from the `composer test` script, so
  every gate run re-dirties the tracked file and it sits in `git status` looking
  like part of your change. It is a master secret — it encrypts session payloads
  and signs the cookie MAC. Re-check `git status` immediately before staging,
  restore it (`git checkout -- <tracked env file>`), and stage **explicit paths**,
  never `git add -A`. Stage first, then read the staged diff's file list: a
  secret is invisible in `git status` output you skimmed.
- **View/config caches hide fixes.** After touching routes, config, or Blade, a
  stale cached view or route can make a passing change fail with a bogus error —
  `php artisan view:clear` / `route:clear` / `config:clear` before re-running.

## Integration

- `test-driven-development` — the RED-GREEN rhythm this procedure sits inside.
- `systematic-debugging` — Phase 1 when a gate stays red after isolation.
- `requesting-code-review` — the pre-commit verification pipeline these gates feed.
- `read-the-damn-docs` — before asserting a package's own API (Livewire, MaryUI,
  Spatie, Playwright) rather than assuming it from memory.
- `references/red-test-attribution.md` — the ordered probes for deciding whether a
  red run is yours, self-inflicted contention, or a pre-existing infrastructure bug,
  plus how to read a failing CI job.
- `references/livewire-mutator-authorization.md` — why route middleware gates a
  Livewire page exactly once, the revoked-permission test recipe (mount → seed →
  revoke → assert both the 403 and the row), why Spatie's fail-closed handling of
  an unknown permission name makes positive controls mandatory, and the
  additions-only grep that proves no scope guard was removed.
- `references/livewire-hydration.md` — what Livewire drops on every round-trip
  (`withCount`, eager loads, appended attributes), how to spot it by inspection, and
  the before/after probe template that exposes state that is correct on first render
  and wrong after any click.
- `references/verifying-an-incoming-change.md` — reproducing someone else's suite in
  a detached worktree to decide whether a change is sound: the `vendor/` symlink that
  produces a false green, missing asset builds, the test-database step, and how to
  separate a regression from a pre-existing defect.