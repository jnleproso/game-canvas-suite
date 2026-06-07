## Goal

Adopt only the **template/schema engine** from `app.html` (game template registry, JSON schema fields, dynamic form generator, schema-bound config storage, campaign record on approval) and graft it onto the existing `JNL_PlayBoost.html`. No UI, navigation, or workflow replacement.

## Current architecture (JNL_PlayBoost.html)

- `LS = { users, session, games, requests, metrics }` in `localStorage`.
- `games[]` items: `{ id, name, category, thumb, path, status, featured, popular }`.
- `requests[]` items: `{ id, userId, businessName, gameId, gameName, category, notes, status, date, adminNotes?, campaignId?, campaignUrl?, qrUrl? }`.
- `openGameModal` / `saveGame` — admin add/edit a game (name, category, thumb, path).
- `requestGame(id)` → modal with `Category` + `Notes` only → `submitRequest` pushes to `requests`.
- `openReqManage` / `applyReqStatus` — admin reviews, sets status, auto-generates `CMP-####` + campaign URL + QR on Approve.
- `renderMyRequests` shows status blocks (Pending / Under Review / Approved / Rejected) with adminNotes.

## Proposed architecture

Layer the schema engine on top of the existing data model. No screens are renamed, no flows are reordered.

```text
games[].schema (NEW)        ──► drives requestGame() dynamic form
requests[].config (NEW)     ──► stores schema-shaped answers
requests[].businessInfo (NEW) ► business fields kept separate
campaigns[] (NEW LS key)    ──► created on Approve, mirrors request.config
```

### 1. Game template registry (data, not UI)
- Add `GAME_TEMPLATE_PRESETS` constant with built-in schemas for `scratch` and `tower` (ported verbatim from `app.html`).
- Extend each game record with: `templateType` ('scratch' | 'tower' | 'custom'), `description`, `demoUrl` (separate from internal `path`), and `schema: Field[]`.
- On load, migrate existing games: if no `schema`, assign by `templateType` (default `scratch`) and seed `description` / `demoUrl` from existing fields.

### 2. Configuration schema system
- Field shape (identical to app.html): `{ key, label, type, required?, default?, min?, max?, placeholder?, hint?, dependsOn?:{key,equals} }`.
- Supported types: `text`, `number`, `image` (base64), `boolean`, `prize-list`, `textarea`.

### 3. Dynamic form generation
- New helpers `fieldHTML(f)`, `renderSchemaFields(host, schema)`, `readConfigFromForm(schema)`, plus `dependsOn` show/hide and prize-list add/remove rows.
- `requestGame(gameId)` modal becomes:
  - **Game configuration** section — fields generated from `game.schema`.
  - **Business information** section — existing Category + Notes (kept) + auto-filled business name/email from `currentUser`.
- `submitRequest` validates via schema, then writes `requests.push({ ...existing fields, config, businessInfo })`. All existing fields (`gameId`, `gameName`, `category`, `notes`, `status:'Submitted'`, `date`) stay so the current admin/customer pages keep rendering.

### 4. Admin Game Management
- `openGameModal` / `saveGame` extended with: **Description**, **Demo URL**, **Template Type** (select: Scratch / Tower / Custom), and a **Schema** section.
- Template Type = preset → schema auto-loads (read-only preview list).
- Template Type = Custom → inline schema editor: add field rows (`key`, `label`, `type`, `required`). Stored as JSON on the game.
- Existing `path` field preserved unchanged (still hidden from customers, still used by `playDemo`).

### 5. Admin request review
- `openReqManage` modal gains a **Submitted Configuration** block that pretty-prints `request.config` against the game's schema (label: value). Admin Notes textarea + Approve/Reject/Under-Review buttons unchanged.
- `applyReqStatus(id,'Approved')` keeps existing `CMP-####` + URL + QR generation, and additionally appends to a new `campaigns` array: `{ id:campaignId, requestId, gameId, templateType, config, url, qrUrl, approvedAt }` saved under `LS.campaigns = 'jnl_campaigns'`.

### 6. Campaign runtime loading (light)
- No new player page is added (out of scope per "do not migrate UI"). The existing campaign URL + QR continue to be the customer-visible artifact.
- A small `getCampaign(id)` helper is exposed on `window` so a future runtime can read `campaigns[]` by id — matches the app.html pattern without porting the player.

## Data structure changes

| Store | Before | After |
|---|---|---|
| `games[]` | name, category, thumb, path | +description, +demoUrl, +templateType, +schema |
| `requests[]` | ...fields, notes | +config, +businessInfo |
| `campaigns[]` | — | NEW: id, requestId, gameId, templateType, config, url, qrUrl, approvedAt |
| `LS` | users/session/games/requests/metrics | +campaigns: `jnl_campaigns` |

One-time migration runs on load: any `game` without `schema` gets the Scratch preset; any `request` without `config` gets `config = { notes: r.notes }` so old records still render.

## Pages affected (logic only — no visual redesign)

- **Admin → Game Management** (modal): new fields (Description, Demo URL, Template Type, Schema editor).
- **Customer → Request Game** (modal): dynamic schema-driven fields above the existing Category/Notes block.
- **Admin → Review Request** (modal): new read-only Submitted Configuration block.
- **Customer → My Requests**: unchanged structurally; Approved card already shows URL + QR.
- All other pages (Landing, Auth, Dashboards, Profile, Accounts, navigation) untouched.

## Risks & migration strategy

- **Risk: old games with no schema break the request modal.** → Mitigation: load-time migration assigns Scratch preset; `requestGame` falls back to legacy (Category + Notes only) if `game.schema` is empty.
- **Risk: existing requests have no `config`.** → Mitigation: migration sets `config = { notes }`; admin review block tolerates empty config.
- **Risk: admin saves invalid custom schema JSON.** → Mitigation: schema editor is a structured row builder, not raw JSON; validated before save.
- **Risk: regressions in approval flow.** → Mitigation: `applyReqStatus` keeps the exact existing CMP-####/URL/QR code path; campaign record is an additive write only.

## Files

- `/mnt/documents/JNL_PlayBoost.html` — single-file patch. No new files, no removed features.

Awaiting approval before patching.