# Yazz / VisualJS Architecture

## Overview

Yazz (also called VisualJS) is a low-code web application builder. Users drag and drop components onto a canvas, wire them together, and write small JavaScript snippets as glue code. The result is a reusable app identified by the SHA-256 hash of its source.

The system is a single-process Node.js server (`electron.js`) that serves a Vue.js 2 single-page application (`go.html`). All state lives in a local SQLite database. There is no build step and no separate frontend build artifact—`go.html` is both the editor and the app runner.

---

## Source layout

```
VisualJS/
├── src/
│   ├── electron.js                           # Express server + all HTTP routes (~5,540 lines)
│   ├── yazz_helper_module.js                 # DB helpers, code storage, HTML generation (~3,219 lines)
│   ├── compile.js / compile2.js              # Component compilation
│   ├── exeProcess.js                         # Child-process execution of server-side components
│   ├── extraFns.js                           # Utility functions
│   ├── perf.js                               # Performance monitoring
│   ├── import_mdb.js                         # MS Access import
│   ├── node_url_require.js                   # Dynamic module loader
│   └── runtimePipeline*.js                   # Vue skeleton and method pipelines (see below)
├── public/
│   ├── go.html                               # Main SPA: editor + home screen + app runner (~15,050 lines)
│   ├── visifile_drivers/                     # Built-in .vjs components
│   │   ├── all_system_components/            # Core system components
│   │   ├── controls/                         # UI control components
│   │   ├── services/                         # Service components
│   │   ├── *.vjs                             # Standalone driver files (csv, excel, pdf, sqlite, etc.)
│   └── driver_icons/                         # Component icon images
└── package.json
```

---

## Key concepts

### Components and the `.vjs` format

Every piece of code in Yazz is a **component** stored as plain text in a `.vjs` file. A component is essentially a JavaScript function that receives inputs and produces outputs. Components can be UI controls (rendered in the browser) or server-side services (executed in a child Node.js process).

### Content-addressed code storage

Yazz uses a two-level identity scheme inspired by IPFS and Unison:

| Field | Meaning | How computed |
|---|---|---|
| `base_component_id` | Stable identity of a component across versions | `COMP_` + SHA-256 of the *original* source code when first created |
| `id` (code_id) | Identity of a specific version of the source | SHA-1 (IPFS content hash) of that version's source text |

This means `base_component_id` never changes for the lifetime of a component. When code is edited and saved, a new row is inserted with the same `base_component_id` but a new `id`. The "current" version is always the row with the highest `rowid`.

---

## Database schema

The SQLite database uses a numbered-level naming convention that reflects conceptual tiers.

### Core code storage

**`level_2_system_code`** — the central table; one row per code version:

```
id                TEXT   -- IPFS content hash of source (the code_id)
base_component_id TEXT   -- stable component identity (COMP_ + SHA-256)
display_name      TEXT
component_type    TEXT   -- 'app', 'ui_control', 'service', etc.
creation_timestamp INTEGER
parent_id         TEXT   -- previous version's code_id (forms a DAG)
stamped_as        TEXT
fk_user_id        TEXT
code              TEXT   -- full source text of the component
logo_url          TEXT
```

**`level_2_released_components`** — published/released component snapshots.

**`level_2_comments_and_ratings`** — per-component community feedback.

### Schema / property metadata

**`level_3_component_property_types`** — property type definitions for components.

**`level_3_property_accept_types`** — which property types a component accepts.

### Application-level tables

**`level_4_users`**, **`level_4_sessions`**, **`level_4_cookies`** — authentication.

**`level_4_metamask_logins`** — Web3 wallet logins.

**`level_4_installed_apps_table`** — apps installed for a user.

**`level_4_code_tags_table`** — tags applied to components.

**`level_4_global_vars_table`** — global key-value store.

**`level_4_app_allow_co_access`** — collaborative access grants.

**`level_4_app_db_latest_ddl_revisions`** — per-app SQLite schema versions.

### System / infrastructure

**`level_0_cached_content`** — raw content cache keyed by hash.

**`level_1_content_metadata`** — IPFS-style content metadata.

**`level_8_system_process_info`** / **`level_8_system_process_errors`** — worker process tracking.

**`level_8_download_content_queue`** / **`level_8_upload_content_queue`** — async content sync queues.

---

## The runtime pipeline system

Rather than embedding all Vue.js logic directly in `go.html`, the app/editor behavior is split into six small pipeline files. These are loaded on demand and stored in `yz.runtimePipelines` keyed by name:

| File | Key | Contents |
|---|---|---|
| `runtimePipelineYazzAppVueSkeleton.js` | `APP` | Vue component skeleton: `props`, `template` placeholder, `mounted`, `watch`, `methods`, `data` |
| `runtimePipelineYazzAppVueMethods.js` | `APP_UI_METHODS` | App-specific Vue methods |
| `runtimePipelineYazzAppVueHtmlTemplate.js` | `APP_UI_TEMPLATE` | App HTML template (~86 lines) |
| `runtimePipelineYazzEditorVueMethods.js` | `EDITOR_UI_METHODS` | Editor-specific Vue methods |
| `runtimePipelineYazzEditorVueHtmlTemplate.js` | `EDITOR_UI_TEMPLATE` | Editor HTML template (~2,678 lines) |
| `runtimePipelineYazzCommonVueMethods.js` | `COMMON_UI_METHODS` | Shared Vue methods (~6,618 lines) |

At runtime the skeleton is filled in with the method and template pipelines to assemble a complete Vue component.

---

## App lifecycle

### Creating an app

1. User opens `go.html` in a browser.
2. User drags components onto the canvas in the editor view.
3. On save, the browser POSTs to `/http_post_save_code_v3`.
4. `yazz_helper_module.saveCodeV3` computes the SHA-1 of the source, inserts a row into `level_2_system_code`, and returns the new `base_component_id` and `code_id`.

### Running an app

1. User clicks the **Play** button on an app card in the home screen.
2. The browser navigates to `/app/{base_component_id}.html`.
3. The `/app/*` route in `electron.js` calls `yazz_helper_module.generateAppHtml(db, baseComponentId)`.
4. `generateAppHtml`:
   - Looks up the latest row for `base_component_id` in `level_2_system_code`.
   - Reads `go.html` from disk.
   - Replaces `isStaticHtmlPageApp: false` → `true` and injects placeholder values (`***STATIC_NAME***`, `***STATIC_BASE_COMPONENT_ID***`, `***STATIC_CODE_ID***`).
   - Injects all six runtime pipeline files as escaped strings into `yz.runtimePipelines`.
   - Injects the app's own source via `yz.cacheThisComponentCode(...)`.
   - Recursively injects all sub-components via `yz.universalSaveStaticUIControl(...)`.
   - Returns the complete HTML string — no disk writes.
5. The server streams the generated HTML directly to the browser.
6. In the browser, `go.html` detects `isStaticHtmlPageApp: true` and uses the baked-in component data instead of fetching from the server.

### The `isStaticHtmlPageApp` flag

`go.html` uses this boolean to switch between two modes:

- `false` (normal): loads component code dynamically from the server via API calls.
- `true` (static/app mode): uses component code already embedded in the page; no server calls needed. This is how exported single-file apps work and how `/app/*` serves running apps.

---

## Server routes (key endpoints)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/` | Serves `go.html` (the main SPA) |
| `GET` | `/app/*` | Generates and serves a running app as a self-contained HTML page |
| `POST` | `/http_post_save_code_v3` | Saves a new code version; returns `base_component_id` and `code_id` |
| `POST` | `/http_post_load_ui_components_v3` | Returns component list for the home screen |
| `POST` | `/http_post_load_pipeline_code` | Returns a runtime pipeline file by name |
| `POST` | `/http_post_call_component` | Executes a server-side component in a child process |
| `POST` | `/http_post_generate_component` | Generates a new component (AI-assisted or template) |
| `GET` | `/http_get_load_version_history_v2` | Returns version history DAG for a component |
| `POST` | `/http_post_commit_code` | Commits a code version |
| `POST` | `/http_post_release_commit` | Publishes a component version |
| `GET` | `/http_get_live_check` | Health check |
| `GET` | `/http_get_readiness_check` | Readiness probe (Kubernetes) |
| `POST` | `/http_post_file_upload` | File upload handler |

---

## Key modules

### `electron.js` (~5,540 lines)

The single Node.js entry point. Responsibilities:

- Sets up Express with compression, sessions, cookies, and Keycloak middleware.
- Defines all HTTP routes.
- Manages the child-process pool for server-side component execution (`exeProcess.js`).
- Initialises `yazz_helper_module` and passes it the DB handle, user data directory, and port.
- Handles WebSocket connections for real-time collaboration.

### `yazz_helper_module.js` (~3,219 lines)

A module (`yz`) shared across the server. Key functions:

- `saveCodeV3(db, code, options)` — inserts a new code version; computes `base_component_id` and `code_id`.
- `generateAppHtml(db, baseComponentId)` — builds a self-contained HTML app page in memory and returns `{ html }`.
- `getSubComponents(code)` — parses component source to find referenced child components.
- `getPipelineCode({pipelineFileName})` — reads a runtime pipeline file.
- `getQuickSqlOneRow(db, sql, params)` — Promise-wrapped single-row SQLite query.
- `helpers.getValueOfCodeString(code, key)` — extracts a named value from component source text.
- `helpers.replaceBetween(html, startMark, endMark, value)` — template string replacement utility.

### `go.html` (~15,050 lines)

The entire frontend in one file. Contains:

- The Vue 2 app instance with all data, computed properties, methods, and watchers.
- The home screen (app card grid with Edit and Play buttons).
- The drag-and-drop editor canvas.
- Static app runner (when `isStaticHtmlPageApp: true`).
- Inline CSS and JavaScript; no separate bundle.

### `compile.js` / `compile2.js`

Transpiles component source from Yazz's DSL notation into executable JavaScript before it is stored or run.

### `exeProcess.js`

Manages a pool of child Node.js processes. Server-side components are executed in isolation here so a crashing component cannot bring down the main server.

---

## Data flow summary

```
Browser (go.html)
    │
    │  POST /http_post_save_code_v3
    ▼
electron.js
    │  yazz_helper_module.saveCodeV3()
    ▼
SQLite: level_2_system_code
    (id=sha1(code), base_component_id=COMP_+sha256(original), code=source)

    ─────────────────── later ───────────────────

Browser navigates to /app/COMP_abc123.html
    │
    │  GET /app/*
    ▼
electron.js → yazz_helper_module.generateAppHtml()
    │  reads level_2_system_code for base_component_id
    │  reads go.html from disk
    │  injects: isStaticHtmlPageApp=true, pipelines, app code, sub-components
    ▼
Returns complete self-contained HTML (no disk write)
    │
    ▼
Browser renders app using baked-in pipeline + component code
```

---

## Authentication

Yazz supports two auth methods:

- **Session-based** (`level_4_sessions`, `level_4_cookies`) — standard cookie sessions via `express-session`.
- **Metamask / Web3** (`level_4_metamask_logins`) — wallet signature verification.
- **Keycloak** — optional enterprise SSO, middleware applied at startup.

Routes can be individually protected using `keycloakProtector(...)` which extracts the `base_component_id` from the request to enforce per-app access control (`level_4_app_allow_co_access`).

---

## Deployment

Yazz runs as a single Node.js process and supports:

- **Local development**: `npm start` on Mac/Linux/Windows.
- **Docker**: `docker run -p 80:80 yazzcom/yazz:march2022`
- **Kubernetes / OpenShift**: standard Deployment with liveness (`/http_get_live_check`) and readiness (`/http_get_readiness_check`) probes.

The only native dependency is `sqlite3` (compiled from source on Apple Silicon). Everything else is pure JavaScript.

User data (the SQLite database and any uploaded files) is stored in the `userData` directory, which defaults to `~/Yazz` on macOS. This path is passed into `yazz_helper_module` as `yz.userData` at startup.
