User prefers to call the assistant "Celin" (Celin) instead of Hermes.
§
h-dashboard project: Laravel 13.x health dashboard for hospital hardware inventory. Upstream canonical: asgarimehdi/h-dashboard (beta); each server has its own fork — never hardcode, read `git remote -v` first. On this box origin is this account's own h-dashboard fork; working branch `celin` (tracks origin/beta). Uses Livewire 4 (single-file anonymous-class components), MaryUI (DaisyUI), Alpine.js, PostgreSQL+PostGIS, Redis, Sanctum auth. Persian/RTL. AGENTS.md is authoritative project instructions.
§
User profile
§ Wants progress visibility on long multi-step work: an initial full status report, then a terse status every 5 minutes until the task finishes (Persian, labelled sections, no filler). Reports should stop as soon as the work is complete.
§
User commands
§ When the user asks to wake up or hand work to another machine of this account, use the Hermes peer commands (`hermes peer …`) where peers are configured — the commander instance manages the other instances of this account.
