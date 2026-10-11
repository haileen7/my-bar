# Livewire mutator authorization: route middleware runs exactly once

A page gated by route middleware (`role_or_permission:x`, `can:x`, custom
`Handle` closures) is gated **only on the page load**. Every follow-up action
from that open tab is served by a *different* route, so the gate never runs
again. A user whose permission was revoked mid-session keeps full CRUD on the
tab until they navigate away.

## The mechanism

Follow-up actions post to `/livewire/update`. That is not the route the tab was
opened on, so its middleware never runs. Livewire has a partial mitigation: a
`PersistentMiddleware` mechanism re-runs a **hardcoded vendor list** on
`/livewire/update`. Third-party permission middleware is not in that list, and
nothing puts it there unless the app opts in.

Before asserting this for your app, read the shipped source rather than
trusting the version:

```bash
grep -n 'persistentMiddleware' -A 12 vendor/livewire/livewire/src/Mechanisms/PersistentMiddleware/PersistentMiddleware.php
grep -rn 'addPersistentMiddleware\|setPersistentMiddleware' app/ bootstrap/ config/
```

A non-empty second result changes the analysis entirely — the global one-line
opt-in may already cover every component, and per-mutator `authorize()` calls
become redundant defence rather than the fix.

## The rule

**Every public method on a gated component that writes must re-check its own
permission as its first statement** — before `validate()`, before any query,
before any scope check.

```php
// The route gate (`role_or_permission:manage_hardware`) does not run on
// `/livewire/update`, so this mutator re-checks it here. It must come
// FIRST — nothing below is scoped by anything a later check could undo.
$this->authorize('manage_hardware');
```

`$this->authorize()` needs no import — `Livewire\Component` already uses
`AuthorizesRequests`. A permission check and a scope check answer different
questions; keep both, permission first.

### Inventory before editing

```bash
grep -n 'public function' resources/views/livewire/<component>.blade.php
grep -n "role_or_permission:\|->middleware(" routes/web.php
```

Cross-reference: every mutator on a route-gated component with no
`authorize()`. Read-only methods that fill a form are a judgement call, not an
oversight — say which you decided and why.

## The generic catch swallows the 403

This is what made the bug invisible. A mutator wrapped in `catch (\Exception $e)`
turns a 403 into a friendly toast, indistinguishable from a real failure:

```php
try {
    $row->delete();
} catch (\Exception $e) {              // ← swallows AuthorizationException
    $this->error('امکان حذف وجود ندارد زیرا در جدول دیگری استفاده شده است.', ...);
}
```

Catch and re-throw **before** the generic clause:

```php
} catch (\Illuminate\Auth\Access\AuthorizationException $e) {
    // Without this the generic catch below swallows the 403 into a friendly
    // toast, making a revoked holder's delete look like a harmless failure.
    throw $e;
} catch (\Exception $e) {
    $this->error('…', ...);
}
```

Order is load-bearing — the specific clause must precede the generic one.

## The behavioural test: revoked-permission mutator

Mount while the permission is **held**, seed against the mounted instance,
revoke, then call the mutator on that same instance:

```php
function revokedHardwareTab(?Closure $seed = null): array
{
    ['user' => $user, 'unit' => $unit] = test()->createUserWithUnit(['manage_hardware']);
    test()->actingAs($user);

    $component = Livewire::test('hardware.index')->assertStatus(200);   // 1. mount

    $seed($user, $unit);                                                // 2. fixture

    $user->revokePermissionTo('manage_hardware');                       // 3. revoke
    app(PermissionRegistrar::class)->forgetCachedPermissions();

    return [$user, $unit, $component];
}
```

Then assert **both** halves:

```php
$component->set('pc_name', 'Hijacked')->call('updateHardware')->assertForbidden();
$this->assertDatabaseHas('hardwares', ['id' => $id, 'pc_name' => 'Original']);
```

| Rule | Why |
|---|---|
| Seed **after** mount, **before** revoke | A row created after the revoke may sit outside the actor's scope, so the *scope* check refuses it and the test passes for the wrong reason — authorization was never consulted. |
| Assert `assertForbidden()` **and** the row | A mutator that failed *validation* also leaves the row untouched. Together they prove the refusal came from authorization. |
| Give the seed closure the **component** | If the mutator's `edit*` is itself gated, filling the form after the revoke 403s on the form and the update under test never runs. |

### Positive controls are mandatory

**Spatie answers `false` for an unknown permission name instead of throwing.**
`$this->authorize('manage_hardwares')` (one `s`) therefore fails **closed**: the
page is locked for everyone who legitimately holds it, and every negative test
still passes. Two defences, both required:

1. A **positive** test per component group — a user who still holds the
   permission calls one mutator and it succeeds.
2. A **static permission-name sweep** asserting every `authorize('…')` in the
   file names exactly the permission its route gate requires:

```php
$source = [
    'resources/views/livewire/hardware/index.blade.php' => 'manage_hardware',
    'resources/views/livewire/units/index.blade.php'   => 'organization',
    'resources/views/livewire/kargozini/radif.blade.php' => 'kargozini',
];
foreach ($source as $file => $permission) {
    preg_match_all("/authorize\('([^']+)'\)/", file_get_contents(base_path($file)), $m);
    $this->assertNotEmpty($m[1], "{$file} has no authorize() call");
    foreach ($m[1] as $found) {
        $this->assertSame($permission, $found, "{$file} authorizes '{$found}'");
    }
}
```

The sweep is cheap but **it passes on an `authorize()` placed after the write**,
so it does not replace the behavioural test. Both, or neither.

### Control test for the page itself

A fresh `GET /page` must still 403 while revoked — that proves the per-mutator
work did not disturb the route gate.

## When the gate breaks tests that were already green

Adding `authorize()` to a mutator that an existing test drives will 403 that
test if its actor never held the permission. **Fix the fixture, never the
gate.** Weakening or reordering the new check to keep a green suite is the one
outcome this whole recipe exists to prevent.

```php
function makeUserInUnit(Unit $unit): User
{
    // … create user …
    // The tests below exercise mutators that now re-check the permission
    // themselves, so the actor must legitimately hold it — otherwise they 403
    // before reaching the scope assertion they exist to make. `findOrCreate`
    // keeps this file independent of the project's permission seeder.
    $user->givePermissionTo(Permission::findOrCreate('manage_hardware'));
```

Diagnose by symptom, in order:

| Symptom | Meaning |
|---|---|
| `assertForbidden()` expected but a *validation* assertion got 0 errors | The 403 pre-empted the validation — the actor lacks the permission. |
| A "scope refuses this" toast is missing entirely | Same: authorization threw before the scope check ran. |
| An update assertion sees the **old** value | Same: the write never happened. |

Confirm granting the permission weakens nothing: scope comes from the access
service's reachable-id set, which never consults the permission table. If a
test genuinely models a user who should NOT hold it, that test is asking for
the authorization behaviour, not the scope behaviour — split it rather than
loosening the gate.

## A static scan cannot see catch ORDER — assert it separately

A scan that asserts the `AuthorizationException` clause exists and contains
`throw $e;` **passes on the exact regression it is named for.** Put the generic
clause first and PHP binds it first (`AuthorizationException extends
Exception`), so the 403 is swallowed into the toast while every other
assertion still holds. Assert the positions:

```php
$authCatch    = strpos($contents, 'AuthorizationException $e');
$genericCatch = strpos($contents, 'catch (\\Exception $e')
    ?: strpos($contents, 'catch (Exception $e');   // unimported variant

$this->assertIsInt($authCatch,    "{$file} does not catch AuthorizationException");
$this->assertIsInt($genericCatch, "{$file} has no generic catch to order against");
$this->assertLessThan($genericCatch, $authCatch, "{$file} swallows a 403");
```

Scan assertions are code, so they need the same red as production code: prove
the teeth before shipping it.

1. Copy the file aside.
2. Delete one gate (or swap the two catches).
3. Run only that test — it **must** fail, and only that one.
4. Restore and confirm green.

A scan that survives its own mutation is decoration. "It passed" means nothing
until you have watched it fail.

## Sibling components: apply one edit, verify all

Structurally identical components (the same lookup-table CRUD four times) are
best edited by scripting the identical patch across all of them — but
**syntax-check every file afterwards**. A batch patch reports success per file
while a templating mistake writes the same broken text into all of them:

```bash
php -l resources/views/livewire/<a>.blade.php
php -l resources/views/livewire/<b>.blade.php   # …all of them, not just the first
```

When scripting edits from Python, remember `{{` escapes a brace only inside the
*f-string* prefix. In `f"...{{" + "plain {{"` the second segment keeps both
braces and writes `{{` into the source.

## Done criteria worth grepping

```bash
grep -c 'authorize('            resources/views/livewire/<component>.blade.php
grep -c 'AuthorizationException' resources/views/livewire/<component>.blade.php
git diff --numstat <component>  # expect additions only — no scope check removed
```

A permission fix must be **additions only**. Any deleted line in a component's
diff means a pre-existing scope guard was dropped or reordered below a write.