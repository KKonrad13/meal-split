# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

MealSplit is a personal PWA (Progressive Web App) for portioning a cooked meal across multiple containers. The user enters the total weight of the meal (including pot/pan), selects a source vessel tare, adds destination containers, and the app calculates the target weight for each container (including its own tare).

Deployed on GitHub Pages at `https://kkonrad13.github.io/meal-split/`. Installable on iPhone via Safari → Add to Home Screen.

## No build step

This is a zero-dependency, zero-build project. The entire app is four static files:

- `index.html` — all CSS, HTML, and JS in one file
- `manifest.json` — PWA metadata
- `sw.js` — service worker (cache-first, pre-caches all four files)
- `icon.svg` — app icon

To develop: open `index.html` directly in a browser, or serve the directory with any static server (e.g. `python3 -m http.server`). The service worker only registers over HTTPS or localhost.

To deploy: `git push` — GitHub Pages serves from `main` branch root automatically.

## Architecture

All state and logic lives in `index.html`. There is no framework, no component system, and no module bundler.

**Data layers:**

- `db` — persisted data synced to `localStorage` (keys: `ms_c` containers, `ms_s` sources, `ms_p` PIN hash). Mutate `db.*`, then call `saveDB(key)`.
- `session.containers` — in-memory array of containers added to the current weighing session. Each entry: `{id, name, weight, mult, count}`. Resets on page reload.

**Rendering pattern:** There is no virtual DOM or reactive system. Each section has a dedicated `render*()` function that rebuilds its innerHTML. Call the relevant render function(s) after any state change. Multiplier and count inputs are read directly from the DOM via `getMult(id)` / `getCount(id)` rather than kept in sync with `session.containers` — exception: `renderContainers()` syncs them back into `session.containers` before rebuilding so values survive list rebuilds.

**Core math** (in `computeResults()`):
```
net_food   = total_weight − source_tare
sum_units  = Σ (mult_i × count_i)
unit_share = net_food / sum_units
food_i     = unit_share × mult_i          // per individual container
target_i   = tare_i + food_i
```

**PIN protection:** Deletion of saved items requires an 8-digit PIN stored as a SHA-256 hex digest in `localStorage` (`ms_p`). Hashing uses the Web Crypto API (`crypto.subtle.digest`). The PIN modal is a state machine with modes `delete | set | change` and steps `verify | choose | confirm | current`.

**Modal system:** A single overlay `#overlay` + sheet `#modal` is reused for both the add-container flow and the PIN keypad. The caller renders into `#modal-title` and `#modal-body` before calling `showOverlay()`.

## iOS / PWA constraints to keep in mind

- Input `font-size` must be ≥ 16px to prevent iOS Safari auto-zoom on focus.
- Use `inputmode="decimal"` for gram inputs, `inputmode="numeric"` for integer inputs (count).
- `env(safe-area-inset-*)` is used for notch/Dynamic Island padding — keep this on header and modal bottom padding.
- The service worker cache name is `mealsplit-v1` — bump the version string in `sw.js` whenever cached files change so existing installs pick up updates.
