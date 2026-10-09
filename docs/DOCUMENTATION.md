# TamaScript — Care Responder Bot (Apps Script)

> **English** · [Bahasa Indonesia](DOCUMENTATION.id.md)

> **Living document.** Update this file whenever behavior changes, so anyone can
> understand the script quickly and change it safely.

---

## Table of contents

1. [What this is](#1-what-this-is)
2. [The script](#2-the-script)
3. [High-level flow](#3-high-level-flow)
4. [Setup & dependencies](#4-setup--dependencies)
5. [Main flow in detail](#5-main-flow-in-detail)
6. [How categorization works](#6-how-categorization-works)
7. [Status commands](#7-status-commands)
8. [Sheet structure](#8-sheet-structure)
9. [Deploy](#9-deploy)
10. [Maintenance checklist](#10-maintenance-checklist)

---

## 1. What this is

A Google Apps Script that powers a **Google Chat bot** named
`@careresponder-app`.

Customer-support agents report an issue in Google Chat; the bot reads the
message, guesses the issue category (fuzzy keyword scoring) and logs it into a
Google Sheet — one tab per month (`Jan` … `Dec`). Agents can also update a
report's status from the same thread.

**Runtime:** Apps Script V8 · **Timezone:** `Asia/Jakarta` · **Advanced service:**
Google Chat API (`Chat`) · **Endpoint:** Google Chat app.

---

## 2. The script

This repository holds a single Apps Script file:

| File | Role |
|---|---|
| `skriptdoublecase.js` | **The bot** — parsing, categorization, duplicate check, writing to the sheet, status commands |
| `README.md` / `docs/` | Documentation |

> In the Apps Script project the bot code may also be named `careskript.js`.
> This file is the source of truth for the bot logic.

---

## 3. High-level flow

```
 User posts a report in Google Chat
        │
        ▼
   onMessage(event)                     (skriptdoublecase.js)
        │
        ├─ parse text   → description, source, MSISDN, User ID, Unique ID
        ├─ fuzzy-match  → Issue Type (category)
        ├─ dedupe check → if already logged, reject with location
        │
        ▼
   appendRow to "<Month>" tab of the SHEET_ID spreadsheet
        │
        ▼
   reply confirmation in the same thread
```

---

## 4. Setup & dependencies

### 4.1 Required Script Property

The script reads the target spreadsheet ID from Script Properties:

```js
PropertiesService.getScriptProperties().getProperty("SHEET_ID")
```

If it is missing, the bot throws an error. To set it:

1. Open the Apps Script editor.
2. **Project Settings → Script Properties → Add a property**.
3. Key = `SHEET_ID`, Value = the spreadsheet ID (the long string in the sheet URL
   between `/d/` and `/edit`).

### 4.2 Advanced service

- **Google Chat API** (`Chat`, v1) must be enabled for the project.

### 4.3 OAuth scopes

- `https://www.googleapis.com/auth/spreadsheets`
- `https://www.googleapis.com/auth/chat.messages`

> The bot must be added to the target Google Chat space and configured to receive
> messages as a Chat app (`onMessage`).

---

## 5. Main flow in detail

Entry points:

- `onMessage(event)` — a message was sent in a space where the bot lives.
- `onAddToSpace(event)` — the bot was added to a space.
- `onRemoveFromSpace(event)` — the bot was removed.

`onMessage` decides what to do with the text:

1. **`input`** → re-read the replied-to message and process it.
2. **Typo suggestion** → if the text looks like a mistyped command, suggest the
   intended command (and do not log it).
3. **Short-message guard** → a 1–2 character message that mentions the bot is
   rejected (and not logged).
4. **Status command** (`checking`, `waiting`, `progress`, `done`, `closed`,
   `reopen`) → `careUpdateStatus()`.
5. **Anything else** → `careProcessReport()` (log a new report).

### `careProcessReport(rawText, spaceUrl, threadName, messageObj, spaceName)`

1. Skip if this exact message was already processed (script cache).
2. `careParseMessage()` → `{ description, source, msisdn, userId, uniqueId }`.
3. `careFuzzyMatch()` → category (issue type).
4. `careFindDuplicateReport()` → reject if already logged (by Unique ID, User ID,
   MSISDN, or legacy description + source match).
5. `getMonthSheet()` → current month tab (auto-creates it if missing).
6. `appendRow([...])` → write the report into the tab.
7. Reply a confirmation summary in the thread.

Concurrency is guarded with `LockService.getScriptLock()` around the
duplicate-check + append, so two messages cannot race.

### `careUpdateStatus(spaceName, threadName, newStatus)`

Finds every row whose **Thread URL*** equals this thread's URL (across all month
tabs) and sets its **Status*** column.

### Helpers

- `careParseMessage` — extracts description, source, MSISDN, User ID, Unique ID.
- `careSanitizeForSheet` — prefixes `= + - @` so the sheet does not treat text as
  a formula.
- `careBuildThreadUrl` / `careBuildDeepLink` — build `chat.google.com` links.
- `careAlreadyProcessed` / `careMarkProcessed` — script-cache based idempotency.

---

## 6. How categorization works

The brain is the `CATEGORIES` array. Each entry:

```js
{ name: "Account Problem", group: "Akun", keywords: ["akun", "account", "banned"] }
```

`careFuzzyMatch(text)` scores every category and returns the best match
(threshold ≥ 15). To force a mapping, add to `CATEGORY_OVERRIDES`.

---

## 7. Status commands

| Command | Status set on the row |
|---|---|
| `checking` | PIC Checking |
| `waiting` | Waiting User Reply |
| `progress` | In Progress Fixing |
| `done` | Solved |
| `closed` | Closed |
| `reopen` | Waiting PIC Reply |

---

## 8. Sheet structure

The bot works on the monthly tab by **position** — it uses the first six columns:

| # | Column | Written by the bot |
|---|---|---|
| 1 | `Date (Automatic)` | report timestamp |
| 2 | `Status*` | report status |
| 3 | `Issue Type*` | category |
| 4 | `Source*` | source |
| 5 | `Thread URL*` | link to the Chat thread |
| 6 | `Description*` | parsed description |

The live tracker ("C&R Tracker SUPER 2026") has more columns after these
(`Platform`, `Notes`, `Root Cause (Oncall)`, `PIC Oncall (Automatic)`,
`Escalation Status (Automatic)`, `Priority (Automatic)`,
`Date Solved/Closed (Automatic)`); the bot does not touch them.

> If a month tab does not exist, `getMonthSheet()` creates it and writes a simple
> header row:
> `Timestamp, Status, Issue Type, Source, Thread, Description, PIC, Note, Resolution, Closed At`.
> Note this header is simpler than the tracker layout above.

---

## 9. Deploy

1. Open the Apps Script project, paste `skriptdoublecase.js` (or `careskript.js`)
   and save.
2. Create a new version / update the deployment so the Chat app uses the new code.
3. Test in a test Chat space first.

> `clasp` can also be used (`clasp push` → `clasp version` → `clasp deploy`).

---

## 10. Maintenance checklist

When you make a change, confirm:

- [ ] `SHEET_ID` Script Property still points at the intended spreadsheet.
- [ ] Any new category/keyword is documented in §6.
- [ ] Month tabs keep the canonical header (see §8).
- [ ] The Chat app deployment uses the latest version.
- [ ] This document is updated.