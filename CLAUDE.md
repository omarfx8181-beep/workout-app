# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A 100-day workout tracker built as an offline-capable PWA, used by the owner's wife on her phone (installed to the home screen via "Add to Home Screen"). It tracks a 100-day challenge running **Jun 12 – Sep 19, 2026**, organized as 10 "weeks" of 10 days each, with workout slots rotating A→J every 10 days.

Features: mark days complete, log weight (lb/kg) and notes per day, streak/completed/remaining stats, per-block and overall weight-change summaries, JSON export/import backup, reset. Checking off a day plays a pop animation; crossing 25/50/75/100 completed days triggers a confetti celebration overlay (`celebrate()` / `confettiBurst()` in `index.html`, detected by comparing `countDone()` before/after a save — in both `saveModal()` and `toggleToday()`).

The app has two views switched by a fixed bottom tab bar (`showView()`): **Today** (default on open — big check-in button that toggles today via `toggleToday()`, an "Add weight & notes" shortcut into the shared modal, and a weight-trend chart) and **All days** (the original 10-week grid, stats, and export/import/reset). The chart is a hand-built inline SVG (`drawChart()`, no chart library); it plots all logged weights by day index, displays in the unit of the most recent entry, and shows an empty-state message until two weights exist. `render()` calls `renderToday()` at the end, so every save path refreshes both views.

## Structure

There is no build system, no dependencies, no tests. The entire app is **one file**:

- `index.html` — all markup, CSS, and JavaScript. CSS is in a `<style>` block, app logic in a `<script>` block at the bottom.
- `sw.js` — service worker. Network-first with cache fallback so the app works offline and updates arrive when online.
- `manifest.json` — PWA manifest (standalone display, icons, theme color `#1a1825`).
- `icon-192.png` / `icon-512.png` — app icons.

## Running / testing changes

Serve the folder over HTTP (service workers don't register from `file://`):

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

After editing, hard-refresh; the service worker is network-first so changes show up on reload. If caching ever gets sticky, bump the `CACHE` version suffix in `sw.js` (currently `workout100-v3`) to force old caches to be purged on activate.

## Key implementation details

- **Data model:** all state lives in `localStorage` under the key `workout100`. It's an object keyed by day index `0`–`99`, each value `{done: bool, weight: string, unit: 'lb'|'kg', note: string}`. The `load()` function migrates an older format where values were `1`. Entries that are fully empty are deleted on save rather than stored.
- **Date math:** `START` in `index.html` (`new Date(2026, 5, 12)`, month is 0-indexed) anchors everything. Day index = days elapsed since START; day number shown to the user = index + 1. Today is highlighted via `todayIndex()`. The date range in the header `<div class="sub">` is hardcoded text — keep it in sync if START changes.
- **Cycle slots:** slot letter for a day is `CYCLE[idx % 10]` (A–J).
- **Weight change:** all comparisons convert to lb via `toLb()` (kg × 2.20462). `blockChange(w)` compares first vs. last logged weight within a 10-day block; the overall total compares first vs. latest logged across all 100 days.
- **Rendering:** `render()` rebuilds the entire `#weeks` DOM from scratch and recomputes stats; it's called after every save. There's no framework or state library — keep it that way unless asked.

## Deployment

Live at **https://omarfx8181-beep.github.io/workout-app/** via GitHub Pages, served from the `main` branch root of https://github.com/omarfx8181-beep/workout-app (public repo). To deploy: commit and `git push` — Pages rebuilds automatically in ~1 minute. When shipping user-visible changes, bump the `CACHE` version in `sw.js` so installed phones fetch the new files. The `gh` CLI is at `~/.local/bin/gh`, authenticated as `omarfx8181-beep`.

## Constraints

- **Don't break existing saved data.** The app is in active daily use; any change to the data shape needs a migration in `load()` (see the existing `done[k]===1` migration as the pattern).
- Phone-first: it's used on a mobile screen. The day grid collapses from 5 columns to 2 below 480px; check narrow-viewport layout when touching CSS.
- Keep it a single self-contained `index.html` — no build step, no npm.
