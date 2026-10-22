---
name: h-dashboard-debugging
version: 1.0.0
author: Hermes Agent (baran session)
license: MIT
description: Use for h-dashboard (baran) debugging and map/page crashes.
metadata:
  hermes:
    tags: [debug, laravel, maps]
    related_skills: []
---

## When to Use

Use when investigating a bug, console error, JS crash, or unexpected behavior in a page of the h-dashboard Laravel app running at http://baran:8000.

# h-dashboard debugging

- **Console errors are captured server-side** in `storage/logs/browser.log` (no laravel.log): grep `local.ERROR` + `"timestamp"` (UTC) + `user_agent`. Server local time is UTC+3:30, the log prefix differs from the embedded UTC timestamp by ~3.5h. Don't add a headless login just to see console errors — check this file first.
- **Dev login**: users seeded by `UsersTableSeeder` (db `h_dashboard`, users keyed by `n_code`, no email/username columns). Password `12345678` for all seeded dev accounts. Login form inputs `#n_code`, `#password`.
- **Leaflet is vendored+minified** at `public/js/leaflet/leaflet.js` (1.9.4). Stack bar:col offsets resolve cleanly with a substring read of that file (offsets minus ~300/+300 chars). Known mapping: `_clipPoints @85334` reads `this._renderer._bounds` then `.min`; `Renderer.onAdd @120386` does `this.getPane().appendChild`; `Map.remove @34569` resets `_panes=[]`, deletes `_mapPane/_renderer` (an add-to-removed-map throws appendChild first, then 'min' forever). Leaflet lifecycle is owned by `resources/js/map-store.js` (Alpine.store('map')); `<livewire:maps.map>` gets torn down + recreated when a parent Livewire re-render morphs it, and host pages binding layers in a single `onReady` closure keep adding to the destroyed instance.
- **Browser instrumentation**: `cdp('Page.addScriptToEvaluateOnNewDocument', source=hook)` survives full page loads and must be used when hooks must be installed before app JS runs (hooks added after load miss the map construction). Attach error hooks (`window.addEventListener('error'/'unhandledrejection')`) AND `L.map`/`L.Map.prototype._initPanes` wrappers there. Full page loads give fresh state; Livewire sidebar links do SPA (wire:navigate) navigation.
- Test accounts may have EMPTY `accessibleUnitIds()` scope — filters return 0 rows; seeder data matters for data-level debugging.
