# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Deployment

This site deploys automatically to **GitHub Pages** on every push to `main` (or the `claude/run-schedule-planner-qarsK` branch). The entire repository root is uploaded as the static site — no build step.

- `index.html` redirects immediately to `planner.html`
- `family-calendar.html` is the main deliverable and can be accessed directly at its URL path

## Files

| File | Purpose |
|------|---------|
| `family-calendar.html` | Full-featured encrypted family calendar (primary work) |
| `planner.html` | Older personal schedule planner (monospace terminal aesthetic) |
| `index.html` | Redirect shim → `planner.html` |

## Architecture — `family-calendar.html`

Everything lives in a single self-contained HTML file with no external dependencies (no npm, no build, no CDN). All CSS and JS are inline.

### Security model
- **First run**: user sets a family name + password. A random salt is generated; `SHA-256(password + salt)` is stored in `localStorage` for future login verification.
- **Encryption key**: derived via `PBKDF2` (150 000 iterations, SHA-256) from the password + salt, producing an `AES-GCM 256-bit` key held only in memory for the session.
- **Storage**: all events and tags are serialised as `{events, tags}`, encrypted with `AES-GCM`, and stored as a base64 blob under `localStorage['fc_data']`. Nothing is stored in plaintext. The key is never persisted.
- **Backwards compatibility**: the loader handles the legacy `fc_ev` key (plain events array) from earlier versions.

### JavaScript object map

| Object | Role |
|--------|------|
| `Cry` | Web Crypto helpers — `hash`, `key` (PBKDF2), `enc`, `dec` (AES-GCM) |
| `S` | Global state — `key`, `events[]`, `tags[]`, `view`, `cur` (Date), `members` (Set), `famName` |
| `M` | Transient modal state — `tags[]`, `attach[]` for the currently open form |
| `MC` | Member colour config — `{donal, nicola, aoife, family}` each with `color`, `bg`, `name` |
| `Store` | `save()` / `load()` — encrypts/decrypts `{events, tags}` to/from `localStorage` |
| `Auth` | `setup()`, `login()`, `logout()`, `showLogin()`, `init()` |
| `Ev` | `add()`, `upd()`, `del()` — events CRUD, calls `Store.save()` |
| `Tags` | `add()`, `del()` (propagates removal to all events), `get()` |
| `Cal` | All calendar rendering — `render()` dispatches to `month()`, `week()`, `day()`, `year()`; also `chipHtml()`, `timedEv()`, `nowLineHtml()`, navigation helpers |
| `Detail` | Read-only event detail popup — `show(id)`, `close()`, `edit()` |
| `Mod` | Add/edit modal — `openNew()`, `openEdit()`, `save()`, `del()`, tag and attachment management |

### Event data shape
```js
{
  id, title, member, createdBy, startDate, endDate,
  allDay, startTime, endTime, description,
  tags: [tagId, ...],
  attachments: [{ path, name }, ...],
  createdAt   // ISO string
}
```

### Tag data shape
```js
{ id, name, color }  // color is a CSS hex string
```

### Timed-event positioning
Week and day views position events using `top = startHour*60 + startMin` (pixels) with `hr-cell` height fixed at `60px`. Changing `hr-cell` height requires updating the same formula in `Cal.timedEv()` and `Cal.scrollTo8()`.

### Click flow
- Clicking a calendar day cell → `Mod.openNew(dateStr)`
- Clicking an event chip → `Detail.show(id)` (read-only popup)
- "Edit Event" in detail popup → `Detail.edit()` → `Mod.openEdit(id)`

### Adding a new family member
1. Add an entry to `MC` with `color`, `bg`, `name`.
2. Add an `<option>` to `#ev-mem` and `#ev-by` selects in the HTML.
3. Add a `.mem-pill` button in the header with `data-m="newmember"`.
4. Add CSS rules for `.mem-pill[data-m="newmember"]` and `.mem-pill[data-m="newmember"].on`.
5. Update `Cal.togMem()` — the `allOn` check array and the family set initialisation in `S.members`.
