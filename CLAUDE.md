# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**飲食記帳本** — A single-file diet tracking web app deployed on GitHub Pages at `https://tony050444669911.github.io/diet-tracker/`.

- `index.html` — the live app (always keep this up to date)
- `sullyoon.html` — snapshot backup; update this before major changes with `cp index.html sullyoon.html`
- `index-v2.html`, `飲食記帳本.html` — older archived versions, do not modify

## Architecture

Everything is in one HTML file (`index.html`, ~1970 lines):

1. **CSS** (lines 12–500 approx) — CSS custom properties for dark blue theme (`--bg`, `--surface`, `--accent`, etc.), Tailwind CDN for utilities, Chart.js from cdnjs
2. **HTML** (lines 500–808) — 4 tabs: 日常 (daily), 分析 (analysis), 食物庫 (food library), 設定 (settings); modals for add food, meal cart, delete confirm, rename, serving size
3. **Apps Script comment** (lines 809–859) — The Google Apps Script source is stored as an HTML comment. This is documentation only — changes here require manually copying into the Apps Script editor and creating a new deployment version. The URL never changes when redeploying an existing deployment.
4. **JavaScript** (lines 861–1970) — all app logic in vanilla JS

## Backend: Google Apps Script + Google Sheets

- Deployed as a Web App, called via GET requests with `URLSearchParams`
- `SHEET_API_URL` is hardcoded at line 865 — never move it to localStorage
- Two sheets: `飲食記錄` (diet entries) and `食物庫` (food library/templates)
- `食物庫` also stores goals as a special row with id `config_goals`, name = `JSON.stringify(goals)`
- **Sheet is always truth**: on init, remote data fully replaces local data (not merged)

### Apps Script actions
| action | sheet | description |
|--------|-------|-------------|
| `getAll` | 飲食記錄 | fetch all entries |
| `add` | 飲食記錄 | append entry row |
| `delete` | 飲食記錄 | delete row by id (uses TextFinder) |
| `getAllTemplates` | 食物庫 | fetch all templates + config rows |
| `addTemplate` | 食物庫 | append template row |
| `deleteTemplate` | 食物庫 | delete row by id (uses TextFinder) |

## Critical Pitfalls

### Chinese column headers
The spreadsheet may have Chinese headers (`第 1 欄`, `食物名稱`, `熱量`, etc.) instead of English (`id`, `name`, `calories`). Always use these helpers:
- `getRowId(r)` — checks `r.id || r['Column 11'] || r['欄 11'] || r['第 1 欄']`
- `getRowName(r)` — checks `r.name || r.food || r['食物名稱'] || r['名稱']`
- `parseSheetRow(r)` — converts any sheet row format to app entry object
- `parseSheetDate(s)` — handles "Wed Mar 25 2026 00:00:00 GMT+0800..." → "2026-03-25"

### Date/timezone
Never use `new Date().toISOString().slice(0,10)` — this returns UTC and shows yesterday in UTC+8 before 8am. Always use:
- `localDateStr(d)` — timezone-safe YYYY-MM-DD from a Date object
- `todayStr()` — calls `localDateStr(new Date())`

### Preset templates
`DEFAULT_FOOD_TEMPLATES` (ids `preset_1`…`preset_17`) are always local-only. Never push them to the sheet. The `confirmDelete` guard `!presetIds.has(deleteTarget.id)` must remain.

### Swipe-to-delete
Both food entry cards and food library template cards use `.swipe-wrap` + `.swipe-del-btn`. `setupSwipeDelete(wrap, inner, onDeleteClick)` handles both touch and mouse drag. `activeSwipeCard` tracks the currently open swipe globally.

## State Variables (global JS)
- `allData` — diet entries array `{id, date, food, calories, protein, carbs, fat, price, photo, detail}`
- `foodTemplates` — template array including presets; `preset_*` ids are built-in
- `goals` — `{calories, protein, carbs, fat, budget}`
- `currentDate` — YYYY-MM-DD string, always reset to `todayStr()` on init
- `currentMeal` — meal cart items array
- `deleteTarget` — `{type:'entry'|'template', id, name}` for the confirm modal

## Deployment Workflow

```bash
# After any change:
git add index.html
git commit -m "..."
git push
# GitHub Pages auto-deploys from main branch
```

When updating Apps Script:
1. Copy the code from the HTML comment (lines 809–859)
2. Paste into script.google.com editor
3. Deploy → Manage deployments → Edit → **New version** → Deploy
4. URL stays the same — no code change needed
