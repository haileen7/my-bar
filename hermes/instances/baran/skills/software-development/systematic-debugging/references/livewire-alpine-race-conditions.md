# Livewire/Alpine.js Modal Race Conditions

## The Pattern

When a Livewire component uses an Alpine.js data component to manage modal state, a race condition occurs if:

1. User clicks a trigger that dispatches a custom event (e.g., `help-open`) with payload data
2. The same click also sets a Livewire property (e.g., `$set('showModal', true)`)
3. Livewire re-renders the component
4. The Alpine component **re-initializes** (because its root element is in the re-rendered DOM)
5. Alpine's `init()` reads static `data-*` attributes, **not** the event payload
6. The event payload is lost — modal shows default content

## Root Cause

**Alpine components re-initialize on every Livewire morph** that replaces their root element. The `init()` hook runs fresh each time. Event listeners registered in `init()` only catch events *after* that initialization. Events dispatched *before* the morph are lost.

## Evidence Checklist

- [ ] Trigger dispatches event with payload: `$dispatch('event-name', { key: value })`
- [ ] Same action sets Livewire property: `$set('property', true)`
- [ ] Alpine component root is inside Livewire's rendering scope
- [ ] Alpine `init()` reads from `this.$el.dataset.*` (static attributes)
- [ ] Alpine does NOT re-read the event payload after morph
- [ ] Default content renders instead of payload-specific content

## Reproduction Recipe

```blade
{{-- Trigger --}}
<button wire:click="$set('showModal', true)"
        x-on:click="$dispatch('help-open', { section: 'users' })">
    Help
</button>

{{-- Modal --}}
<div x-data="helpModal" data-section="dashboard">
    <x-modal wire:model="showModal">
        {{-- Content depends on helpModal.helpSection --}}
    </x-modal>
</div>
```

```js
// Alpine component
Alpine.data('helpModal', () => ({
  helpSection: 'dashboard',
  init() {
    // Runs on EVERY morph — reads static attribute only
    this.helpSection = this.$el.dataset.section || 'dashboard';
    
    // Listener registered AFTER init — misses events from previous render
    document.addEventListener('help-open', (e) => {
      this.helpSection = e.detail.section;
    });
  },
}));
```

Click sequence:
1. User clicks → dispatches `help-open` with `section: 'users'` + sets `showModal=true`
2. Livewire morphs → modal DOM replaced
3. `helpModal.init()` runs → reads `data-section="dashboard"`
4. Listener registered → but event already fired
5. Modal shows **dashboard** help

## Fixes (in order of preference)

### 1. Pass section via static attribute on modal (Recommended)

```blade
<x-help:modal wireModel="showHelpModal" section="{{ $helpSection }}" />
```

```blade
{{-- In modal component --}}
<div x-data="helpModal" data-section="{{ $section ?? 'dashboard' }}">
```

**Why it works:** The `section` value travels with the Livewire render, so when Alpine re-initializes, `data-section` has the correct value. No event dependency.

### 2. Persist section in Livewire property

```php
public string $helpSection = 'dashboard';
```

```blade
<button wire:click="$set('helpSection', 'users'); $set('showModal', true)">
```

```blade
<div x-data="helpModal" data-section="{{ $helpSection }}">
```

**Why it works:** Livewire property survives morph; Alpine reads it from static attribute.

### 3. Use Alpine's `$watch` on a Livewire-synced property

```js
init() {
  this.helpSection = this.$el.dataset.section || 'dashboard';
  this.$watch('helpSection', (val) => {
    if (val) this.fetchContent(val);
  });
}
```

```blade
<div x-data="helpModal" :data-section="$helpSection" x-init="helpSection = $helpSection">
```

**Caveat:** `x-init` also re-runs on morph — same problem. Only works if the property is already synced before Alpine initializes.

## Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails |
|--------------|--------------|
| Relying solely on `$dispatch` + `$set` in same click | Event fires before morph, listener registers after |
| Using `x-on:click` on trigger to set Alpine state directly | Trigger and modal may be in different components |
| Storing section only in Alpine state | Lost on every morph |
| Using `wire:model` on modal without static `section` attribute | Alpine has no source of truth after re-init |

## Generalization

This pattern applies to **any** Livewire + Alpine interaction where:
- Client event carries payload
- Server re-renders the Alpine root element
- Alpine `init()` reads static attributes
- Event listener registers in `init()`

**Fix principle:** Payload must travel through Livewire render cycle (as prop/attribute), not through event bus.

## Debugging Steps

1. Add `console.log('helpModal init', this.helpSection)` in `init()`
2. Add `console.log('help-open event', e.detail)` in listener
3. Click trigger → observe order: event fires → morph → init runs → listener registers
4. Verify `data-section` attribute on modal root in browser devtools after morph
5. Fix by ensuring correct value is in `data-section` attribute on render
