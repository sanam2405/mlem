# Hot Reloading vs Restarting the Dev Server

A practical guide to understanding when code changes are picked up automatically and when you need to restart, specific to the Jurisphere monorepo and the frameworks it uses.

---

## Table of Contents

1. [The Two Mechanisms](#1-the-two-mechanisms)
2. [Service-by-Service Breakdown](#2-service-by-service-breakdown)
3. [Next.js with Turbopack — Webapp & Console](#3-nextjs-with-turbopack--webapp--console)
4. [FastAPI with Uvicorn — API](#4-fastapi-with-uvicorn--api)
5. [SQLAlchemy & the Connection Pool](#5-sqlalchemy--the-connection-pool)
6. [Temporal Workers with watchmedo](#6-temporal-workers-with-watchmedo)
7. [Node.js --watch — Notifications Service](#7-nodejs---watch--notifications-service)
8. [Webpack Dev Server — Word & Outlook Add-ins](#8-webpack-dev-server--word--outlook-add-ins)
9. [Design System — The Monorepo Bottleneck](#9-design-system--the-monorepo-bottleneck)
10. [React Query Cache & Stale Data](#10-react-query-cache--stale-data)
11. [Environment Variables (.env)](#11-environment-variables-env)
12. [Dependencies & Lock Files](#12-dependencies--lock-files)
13. [Database Migrations](#13-database-migrations)
14. [Quick Reference Decision Table](#14-quick-reference-decision-table)
15. [Debugging: "I Changed the Code but Nothing Happened"](#15-debugging-i-changed-the-code-but-nothing-happened)

---

## 1. The Two Mechanisms

Every dev tool that "picks up changes" does one of two things:

### Hot Module Replacement (HMR)

The running process stays alive. Only the changed module is swapped in-place. Application state (React component state, in-memory caches, WebSocket connections) **survives**.

```
File saved → Bundler detects change → Recompiles only that module
→ Pushes update over WebSocket → Runtime swaps module in-place
→ React re-renders affected components → State preserved
```

**Used by**: Next.js (via Turbopack), Webpack Dev Server (Word/Outlook add-ins).

### Full Process Restart

The running process is killed and re-launched from scratch. All in-memory state is **lost**. Every module is re-imported. Every connection is re-established.

```
File saved → Watcher detects change → Kills process (SIGTERM)
→ Re-executes the start command → Full cold start
→ All imports re-evaluated → New connections established
```

**Used by**: Uvicorn `--reload` (API), `watchmedo auto-restart` (Temporal workers), Node.js `--watch` (notifications).

### Why This Matters

HMR is faster and preserves state, but it can only swap code — it cannot re-read environment variables, re-establish database connections, or pick up new dependencies. Full restart is slower but gives you a clean slate.

The confusion happens when you expect HMR behavior (instant, stateful) but the tool does a full restart (slow, stateless), or when you expect a restart but the tool does nothing at all.

---

## 2. Service-by-Service Breakdown

Here is how every service in the Jurisphere dev environment is started, what watches for changes, and what kind of reload it performs:

| Service | Port | Start Command | Watch Tool | Reload Type |
|---|---|---|---|---|
| Webapp (Next.js) | 3000 | `next dev --turbopack -H 0.0.0.0` | Turbopack (Rust) | HMR (Fast Refresh) |
| Console (Next.js) | 3141 | `next dev --turbopack` | Turbopack (Rust) | HMR (Fast Refresh) |
| API (FastAPI) | 8000 | `uvicorn main:app --reload` | watchfiles (Rust) | Full process restart |
| Worker — Interactive | — | `watchmedo auto-restart --pattern="*.py" --recursive --interval=1` | watchdog (Python) | Full process restart |
| Worker — Ingest | — | (same as above) | watchdog (Python) | Full process restart |
| Worker — Background | — | (same as above) | watchdog (Python) | Full process restart |
| Notifications | 3002 | `node --watch src/index.js` | Node.js built-in | Full process restart |
| Word Add-in | 3005 | `webpack-dev-server` | Webpack | HMR |
| Outlook Add-in | 3006 | `webpack-dev-server` | Webpack | HMR |
| Design System | — | `tsup --watch` (manual) | tsup/esbuild | Rebuild only (no reload) |
| Temporal Server | 7233 | `temporal server start-dev` | None | Manual restart |
| Redis | 6379 | `redis-server` | None | Manual restart |

---

## 3. Next.js with Turbopack — Webapp & Console

The webapp and console are both Next.js apps running with Turbopack. They use **Fast Refresh**, which is React's HMR implementation — it swaps React components in-place while preserving `useState`, `useRef`, and other hook state.

### What Fast Refresh Handles (No Restart Needed)

| Change | Behavior |
|---|---|
| Edit a React component file (`.tsx`) | **HMR** — component re-renders, state preserved |
| Edit a Server Component | **Server-side HMR** — Turbopack re-evaluates only the changed server module (stable since Next.js 16.2) |
| Edit `layout.tsx` | **HMR** — the entire subtree under that layout re-renders |
| Edit `middleware.ts` | **Server recompilation** — triggers a full page reload in the browser, but the dev server stays running |
| Edit CSS / Tailwind classes | **HMR** — styles update instantly |
| Add/remove `'use client'` directive | **Full page reload** — fundamentally changes the rendering boundary |
| Syntax error in a file | **Error overlay** — fix the error and state recovers without reload |
| Edit a utility file imported by components | **HMR** — re-runs that file AND all files importing it. State may reset if the import chain crosses a non-component boundary |

### What Requires Restarting `next dev`

| Change | Why |
|---|---|
| `next.config.mjs` / `next.config.ts` | Config is read once at server startup. Not watched. |
| `tailwind.config.ts` | Tailwind's JIT compiler is initialized once. Changes to `content` paths, plugins, or theme extensions require restart. (Changes to classes in existing files are fine — those are picked up by HMR.) |
| New dependency installed (`pnpm add`) | The module resolver caches the `node_modules` tree at startup. New packages are invisible until restart. |
| Turbopack cache corruption | Rare, but if HMR stops working for no apparent reason: `rm -rf .next && restart`. The webapp task already does `rm -rf .next` on start. |
| Design system rebuild | See [Section 9](#9-design-system--the-monorepo-bottleneck). |

### The `serverComponentsHmrCache` Gotcha

Next.js caches `fetch()` responses across HMR refreshes by default, including `cache: 'no-store'` fetches. This means:

```
1. Server Component fetches data from API → renders
2. You change the API endpoint's logic → uvicorn reloads
3. You edit the Server Component → HMR triggers
4. The component re-renders but uses the CACHED fetch response
5. You see stale data and think your API change didn't work
```

The webapp has this disabled (`turbopackFileSystemCacheForDev: false`), but the console has it **enabled** (`turbopackFileSystemCacheForDev: true`). If you see stale data in the console after changing API logic, do a full page reload (Cmd+Shift+R) to bypass the cache.

### Anonymous Exports Kill Fast Refresh

```tsx
// BAD — Fast Refresh cannot track anonymous components. Falls back to full reload.
export default () => <div>Hello</div>;

// GOOD — named export. Fast Refresh preserves state.
export default function Greeting() {
  return <div>Hello</div>;
}
```

If you notice that every save causes a full page reload instead of a fast refresh, check for anonymous default exports.

---

## 4. FastAPI with Uvicorn — API

The API runs as:

```bash
uvicorn main:app --reload
```

This is **not** hot module replacement. Uvicorn spawns a supervisor process that watches the filesystem. When a change is detected, it **kills the worker process entirely** and re-executes it from scratch. Every Python module is re-imported. Every object is re-created.

### How the Watcher Works

With `uvicorn[standard]` installed (which Jurisphere uses), uvicorn uses **watchfiles** — a Rust library that hooks into OS-level filesystem events (FSEvents on macOS, inotify on Linux). This is efficient and near-instant.

Without `uvicorn[standard]`, it falls back to polling file modification times — slower and CPU-heavy.

### What Triggers a Reload

| Change | Detected? | Notes |
|---|---|---|
| Any `.py` file modified | **Yes** | Default watch pattern is `*.py` |
| New `.py` file created | **Yes** | watchfiles detects new file creation |
| `.py` file deleted | **Yes** | Triggers reload |
| `.env` / `.env.local` changed | **No** | Excluded by the default `.*` glob pattern. `--reload-include=.env` does NOT override this because the exclude takes precedence. |
| `.yaml` / `.json` config file changed | **No** | Only `*.py` is watched by default. Would need `--reload-include '*.yaml'`. |
| Alembic migration file changed | **Yes** (it's `.py`) | But the migration doesn't run — see [Section 13](#13-database-migrations). |

### Reload Latency

- **Detection**: Near-instant (OS-level events via watchfiles).
- **Debounce**: 0.25 seconds (uvicorn's `--reload-delay` default). Multiple rapid saves within this window are batched into one restart.
- **Restart**: Depends on import time. A large app with heavy imports (SQLAlchemy, Temporal SDK, ML libraries) can take 2-5 seconds to fully restart.
- **Total**: Typically 1-3 seconds from save to ready.

### What Happens During Restart

```
1. Supervisor sends SIGTERM to worker process
2. Worker process exits (all in-memory state lost)
3. OS reclaims all file descriptors (DB connections, sockets)
4. Database server detects broken connections, rolls back pending transactions
5. New worker process starts, re-imports all modules
6. New SQLAlchemy engine + connection pool created
7. New connections established on first request
```

Any request that hits the API during this ~2 second window will fail with a connection error. The frontend will show a network error toast.

---

## 5. SQLAlchemy & the Connection Pool

### On Uvicorn Reload

When uvicorn kills the worker process, database connections are **not gracefully closed** — the process simply exits and the OS reclaims the sockets. PostgreSQL detects the broken connections via TCP keepalive and cleans them up server-side.

The new worker process creates a fresh `Engine` with a new connection pool. Connections are established lazily on first use.

```
Before reload:
  Worker Process → Connection Pool (5 connections) → PostgreSQL

During reload (process killed):
  [Process dead] → [5 orphaned connections] → PostgreSQL
  PostgreSQL: "TCP keepalive failed, cleaning up..."

After reload (new process):
  New Worker → New Pool (0 connections) → PostgreSQL
  First request → Pool creates connection #1
```

### Why This Matters for Development

1. **Pending transactions are rolled back.** If you save a `.py` file while a long-running request is in-flight, that request's transaction is aborted.

2. **Connection limits.** Each reload creates new connections without gracefully closing old ones. PostgreSQL has a connection limit (typically 100). Rapid-fire saves (e.g., auto-save every keystroke) can temporarily exhaust the limit until PostgreSQL cleans up orphaned connections. In practice this is rare — PostgreSQL's keepalive timeout is usually 2 minutes.

3. **`pool_pre_ping=True`** is recommended for development. It issues a `SELECT 1` before reusing a pooled connection, catching stale connections that survived from a previous process incarnation. Jurisphere uses this in its database configuration.

### Schema Changes

SQLAlchemy models are Python code — changing a model file triggers uvicorn reload, and the new model definition is imported. But **the database schema doesn't change** until you run a migration. See [Section 13](#13-database-migrations).

---

## 6. Temporal Workers with watchmedo

The three Temporal workers (interactive, ingest, background) all use:

```bash
watchmedo auto-restart --pattern="*.py" --recursive --interval=1 \
  -- uv run python -m app.job_queue.worker
```

### How It Differs from Uvicorn's Reload

| Aspect | uvicorn `--reload` | watchmedo `auto-restart` |
|---|---|---|
| Library | watchfiles (Rust, OS-native events) | watchdog (Python, OS-native events) |
| Detection | Event-driven + 0.25s debounce | Polling at `--interval=1` (1 second) |
| Scope | Watches the uvicorn working directory | `--recursive` watches from the CWD down |
| Pattern | `*.py` only by default | `--pattern="*.py"` (configurable) |
| Restart | Kills worker, re-execs within supervisor | Kills subprocess via SIGTERM, re-execs command |

### What This Means for Temporal Workflows

- **In-flight activities are interrupted.** Temporal will retry them based on the retry policy, but the current execution is lost.
- **Workflow state is safe.** Temporal stores workflow state server-side. The worker restart doesn't affect workflow history — the worker simply picks up where it left off.
- **Long-running activities (e.g., document processing) will be retried from scratch.** If you're debugging an activity that processes a 500-page PDF, saving a `.py` file will restart the worker and the activity will re-run from the beginning.

### Practical Advice

If you're debugging a long-running workflow and don't want the worker to restart on every save, temporarily start the worker without watchmedo:

```bash
cd api && dotenvx run -f .env.local -f .env -- uv run python -m app.job_queue.worker
```

Then restart manually when you're ready to test your changes.

---

## 7. Node.js `--watch` — Notifications Service

The notifications service runs as:

```bash
node --watch src/index.js
```

### How `--watch` Works

Node.js tracks every file that is `require()`'d or `import()`'d starting from the entry point. When any of these files change, it **kills and re-executes the entire process**.

This is a full process restart, not HMR. All in-memory state is lost — including:
- Active SSE (Server-Sent Events) connections to browsers
- Redis subscription state
- Any in-memory notification buffers

### What Is and Isn't Watched

| File | Watched? | Why |
|---|---|---|
| `src/index.js` | **Yes** | Entry point |
| Files imported by `src/index.js` | **Yes** | Part of the dependency graph |
| `package.json` | **No** | Not imported by the app |
| New files not yet imported | **No** | Not part of the dependency graph until imported |
| `.env` files | **No** | Read via `process.env`, not `require()` |

### Impact on Development

When the notifications service restarts, all SSE connections drop. The webapp's notification listener will reconnect automatically (EventSource has built-in reconnection), but you'll see a brief gap in real-time notifications during the restart.

---

## 8. Webpack Dev Server — Word & Outlook Add-ins

The Word add-in (port 3005) and Outlook add-in (port 3006) use webpack-dev-server with `hot: true`.

### How Webpack HMR Works

```
File saved → Webpack recompiles changed module(s)
→ Dev server pushes update manifest via WebSocket
→ HMR runtime in browser receives manifest
→ If module (or ancestor) has module.hot.accept() → swap in-place
→ If no module accepts → fall back to full page reload
```

### What Triggers HMR vs Full Reload

| Change | Behavior |
|---|---|
| React component edit | **HMR** — react-refresh-webpack-plugin handles the swap |
| CSS/SCSS edit | **HMR** — style-loader implements `module.hot.accept()` |
| Non-component JS/TS edit without HMR handler | **Full page reload** |
| `webpack.config.js` | **Requires manual restart** — config is read once at startup |
| `manifest.xml` | **Requires re-sideloading** the add-in in Office |

### Office Add-in Specifics

Office WebView caches aggressively. If you change code but the add-in doesn't update:

1. Check if webpack-dev-server is actually recompiling (look at terminal output).
2. Clear the Office WebView cache (in Word: File → Options → Trust Center → Clear Web Cache).
3. Close and re-open the task pane.

---

## 9. Design System — The Monorepo Bottleneck

The design system (`packages/design-system`) is the **biggest friction point** in the dev environment. It is a shared package consumed by the webapp and console, but it is **not automatically watched**.

### How It Works Today

```
task start
  └── task build-design-system     ← builds once
  └── overmind start
       ├── webapp (imports from design-system/dist/)
       ├── console (imports from design-system/dist/)
       └── ... other services
```

The design system is built **once** at startup via the `build-design-system` dependency (and the webapp's `predev` script: `pnpm --filter @jurisphere/design-system build`). After that, no one watches it.

### The Problem

```
1. You edit packages/design-system/src/components/Button.tsx
2. Nothing happens. The webapp still uses the OLD built output in dist/.
3. You wonder why your change didn't take effect.
4. You restart the entire dev environment (task start).
5. The design system rebuilds. The webapp picks up the new output.
6. 30 seconds wasted.
```

### The Fix: Run the Design System in Watch Mode

In a separate terminal:

```bash
cd packages/design-system && pnpm dev
# This runs: pnpm generate && tsup --watch
```

Now tsup watches for changes in the design system source and rebuilds `dist/` incrementally. The webapp's Turbopack detects the changed files in `dist/` and triggers HMR.

```
Edit design-system source → tsup rebuilds dist/ (~200ms)
→ Turbopack detects dist/ change → HMR in webapp → Component updates
```

**Total latency**: ~1-2 seconds, no restart needed.

### Why It's Not Automatic

The `task start` command uses Overmind, which runs the processes defined in `Procfile.dev`. The design system is not in the Procfile — it's a build dependency, not a long-running service. Adding a `design-system: cd packages/design-system && pnpm dev` line to the Procfile would fix this, but it hasn't been done yet.

---

## 10. React Query Cache & Stale Data

React Query (TanStack Query) maintains an in-memory cache of all fetched data. This cache interacts with HMR in confusing ways.

### The Problem

```
1. Webapp fetches GET /api/matters → React Query caches the response
2. You change API logic (e.g., add a new field to the response)
3. Uvicorn restarts with new logic
4. You edit a component in the webapp → HMR triggers
5. React Query says "I already have fresh data for this query" → does NOT refetch
6. You see the OLD response without the new field
7. You think your API change didn't work
```

### When Cache Survives HMR

- The `QueryClient` instance is typically created at module scope (e.g., in a provider file). If that module is not re-evaluated during HMR, the same `QueryClient` with all its cached data survives.
- `staleTime` controls how long data is considered "fresh." If set to a high value (minutes or `Infinity`), queries won't refetch on component remount.
- `gcTime` (garbage collection time, default 5 minutes) controls how long unmounted query data persists. Stale data from a component you navigated away from is still in memory for 5 minutes.

### When Cache Is Cleared

- Full page reload (Cmd+Shift+R) always clears the cache.
- If the module exporting `QueryClient` is re-evaluated during HMR (rare — usually only if you edit that specific file), a new empty `QueryClient` is created.
- Navigating away and back triggers a refetch if `staleTime` has elapsed.

### How to Force Fresh Data During Development

**Option 1: Hard reload the page**

Cmd+Shift+R clears the React Query cache (and all other in-memory state).

**Option 2: Use React Query Devtools**

The devtools panel (bottom of the screen in dev mode) lets you inspect and manually invalidate individual queries. Click a query → "Invalidate" → it refetches.

**Option 3: Change the query key**

If you're debugging a specific query, temporarily add a dummy param to force a cache miss:

```tsx
// Temporarily force refetch by changing the key
useQuery({ queryKey: ["matters", matterId, "v2"], ... });
//                                          ^^^^ remove after debugging
```

### The Rule of Thumb

**If you changed backend logic and the frontend still shows old data, do a hard reload (Cmd+Shift+R) before assuming your backend change didn't work.**

This is what happened with the A4 page size fix — the code was correct, but the dev server needed a restart because the change was in a utility consumed at build time, and the old output was cached.

---

## 11. Environment Variables (.env)

Environment variables are the most common "restart required" scenario, and every framework handles them differently.

### Next.js (Webapp & Console)

| Variable Type | Hot Reload? | Notes |
|---|---|---|
| `NEXT_PUBLIC_*` in `.env` | **No** — restart required | These are inlined into the JS bundle at compile time. The compiled code literally contains the string value, not a reference to `process.env`. Changing the value requires recompilation. |
| Server-side env vars in `.env` | **Partial** | Next.js detects `.env` file changes and triggers a reload since v12.3. But vars used inside `next.config.mjs` still require a manual restart (config is read once). |
| `NEXT_PUBLIC_*` in `.env.local` | **No** — restart required | Same as above — build-time inlining. |
| Vars accessed via `process.env` in API routes | **Yes** | Server-side code re-evaluates `process.env` on each request. But the dev server must re-read the `.env` file first — which happens on Next.js restart, not on every request. |

**In practice**: Always restart `next dev` after changing any `.env` file. The "partial" detection is unreliable enough that it's not worth debugging.

### FastAPI (API)

| Scenario | Hot Reload? | Why |
|---|---|---|
| `.env` or `.env.local` changed | **No** — restart required | Uvicorn's watcher explicitly excludes dotfiles (`.*` pattern). The `--reload-include=.env` flag does NOT override this because the exclude takes precedence. This is a known uvicorn issue. |
| Env var changed in shell, then save a `.py` file | **Yes** | The `.py` save triggers a process restart, which re-reads the environment. But env vars from `.env` files are injected by `dotenvx`, which runs once at process startup. |
| Env var used in `Settings` / Pydantic BaseSettings | **No** — restart required | Pydantic reads env vars once during class instantiation. The `Settings` object is typically a singleton. |

**In practice**: After changing any `.env` file for the API, you need to restart the entire process (not just trigger a `.py` reload). Kill the API process in Overmind (`overmind restart api`) or restart `task start`.

### Temporal Workers

Same as API — `dotenvx` injects env vars at process startup. Changing `.env` requires restarting the worker process. Saving a `.py` file triggers `watchmedo` restart, but `dotenvx` won't re-read the `.env` file because it's the parent process wrapper, not the restarted child.

**In practice**: Restart via Overmind: `overmind restart worker-interactive` (etc.).

### Notifications Service

Same pattern — env vars are hardcoded in the root `Taskfile.yaml`:

```yaml
notifications:
  env:
    PORT: "3002"
    REDIS_URL: "redis://localhost:6379"
    NOTIFICATION_STREAM_TOKEN_SECRET: "jurisphere-notifications-stream-token-secret"
    WEBAPP_URL: "http://localhost:3000"
```

These are set once when the task starts. Changing them requires restarting the notifications process.

### Summary

**Changing a `.env` file always requires restarting the affected service.** No framework in this repo reliably hot-reloads environment variables.

---

## 12. Dependencies & Lock Files

### Installing a New npm Package (Webapp/Console)

```bash
cd webapp && pnpm add some-package
```

After installing:

| What | Hot Reload? | Notes |
|---|---|---|
| Import the new package in a `.tsx` file | **No** — restart `next dev` | The module resolver caches the `node_modules` tree at startup. New packages are invisible until restart. |
| Update an existing package version | **No** — restart `next dev` | Same reason — cached module resolution. |
| `pnpm remove` a package | **No** — restart `next dev` | Removing a package while the dev server uses it causes runtime errors. |

### Installing a New Python Package (API)

```bash
cd api && uv add some-package
```

After installing:

| What | Hot Reload? | Notes |
|---|---|---|
| Import the new package in a `.py` file | **No** — restart uvicorn | New packages are installed into `site-packages`, which is outside uvicorn's watched directories. The supervisor process doesn't detect changes there. |
| Update an existing package version | **No** — restart uvicorn | Same reason. Additionally, the old version may be cached in `sys.modules`. |

**In practice**: After any `pnpm add`, `pnpm remove`, `uv add`, or `uv remove`, restart the affected service.

---

## 13. Database Migrations

### Alembic Migrations (API)

```bash
cd api && task run -- alembic upgrade head
```

Running a migration modifies the database schema directly (DDL statements). The running API process does **not** detect this.

| Scenario | What Happens |
|---|---|
| Add a new column + update model + save | Uvicorn restarts (`.py` change), new model is imported. But if the migration hasn't run, the column doesn't exist in the DB → runtime error on first query touching that column. |
| Run migration + don't save any `.py` file | The DB schema changes, but the API is still running with the OLD model code. It might work (if the migration only adds a nullable column) or might break (if it renames/removes a column). |
| Run migration + save a `.py` file | **Correct workflow.** Migration updates DB, `.py` save triggers uvicorn restart, new model code matches new schema. |

### The Correct Order

```
1. Write the migration          (alembic revision --autogenerate -m "...")
2. Write the model changes      (edit the SQLAlchemy model)
3. Run the migration            (alembic upgrade head)
4. Save a .py file / restart    (uvicorn picks up the new model)
```

Step 3 and 4 can happen in either order for additive changes (new nullable column), but for destructive changes (column rename, column removal), the model code must match the schema when the API restarts.

---

## 14. Quick Reference Decision Table

### "I just changed X — do I need to restart?"

| What You Changed | Webapp | API | Workers | Notifications |
|---|---|---|---|---|
| `.tsx` / `.ts` component file | HMR ✅ | — | — | — |
| `.py` source file | — | Auto-restart ✅ | Auto-restart ✅ | — |
| `.js` source file (notifications) | — | — | — | Auto-restart ✅ |
| `.env` / `.env.local` | ❌ Restart | ❌ Restart | ❌ Restart | ❌ Restart |
| `next.config.mjs` | ❌ Restart | — | — | — |
| `tailwind.config.ts` | ❌ Restart | — | — | — |
| `webpack.config.js` | — | — | — | — |
| `package.json` (deps changed) | ❌ Restart | — | — | ❌ Restart |
| `pyproject.toml` (deps changed) | — | ❌ Restart | ❌ Restart | — |
| Alembic migration (run) | — | ❌ Restart | ❌ Restart | — |
| Design system source | ❌ Rebuild DS | — | — | — |
| `Taskfile.yaml` | ❌ Restart | ❌ Restart | ❌ Restart | ❌ Restart |
| Procfile changes | ❌ Restart Overmind | ❌ Restart Overmind | ❌ Restart Overmind | ❌ Restart Overmind |

Legend: ✅ = automatic, ❌ = manual action required, — = not applicable.

### Restart Commands

```bash
# Restart a single service via Overmind
overmind restart api
overmind restart webapp
overmind restart worker-interactive
overmind restart notifications

# Restart everything
overmind quit    # or Ctrl+C
task start       # re-runs build-design-system + all services

# Nuclear option (clears all caches)
cd webapp && rm -rf .next    # clear Turbopack cache
task start                   # full restart
```

---

## 15. Debugging: "I Changed the Code but Nothing Happened"

Work through this checklist in order:

### Step 1: Did the watcher detect the change?

Look at the terminal where the service is running.

- **Uvicorn**: You should see `WARNING: WatchFiles detected changes in '...'`. If you don't, the file extension isn't watched (e.g., `.env`, `.yaml`).
- **Next.js**: You should see `○ Compiling /page ...` in the terminal. If you don't, the file might be outside the watched directory or not imported by any page.
- **watchmedo**: You should see the worker process restarting (new PID logged). If you don't, the file doesn't match `--pattern="*.py"`.

### Step 2: Is the watcher even running?

In Overmind, check if the service is alive:

```bash
overmind ps
```

If a service crashed and didn't restart, you'll see it as stopped. Restart it:

```bash
overmind restart <service-name>
```

### Step 3: Is it a caching issue?

| Symptom | Likely Cause | Fix |
|---|---|---|
| API logic changed but frontend shows old data | React Query cache | Hard reload: Cmd+Shift+R |
| Component changed but page doesn't update | Turbopack cache | `rm -rf webapp/.next && restart` |
| Design system changed but webapp looks the same | Design system not rebuilt | `cd packages/design-system && pnpm build` |
| Env var changed but app uses old value | Process env cache | Restart the service |
| New npm package can't be imported | Module resolver cache | Restart `next dev` |
| API returns old schema after migration | Uvicorn using old model | Save a `.py` file to trigger restart |

### Step 4: Is it a stale browser cache?

If nothing else works:

1. Hard reload: **Cmd+Shift+R** (bypasses browser cache).
2. Open DevTools → Network tab → check **"Disable cache"**.
3. Clear site data: DevTools → Application → Storage → "Clear site data".

### Step 5: Is it a stale build artifact?

Nuclear option — clear everything and restart:

```bash
# Kill all dev processes
overmind quit

# Clear build caches
rm -rf webapp/.next
rm -rf console/.next
rm -rf packages/design-system/dist

# Rebuild and restart
task start
```

---

## Appendix: How Each Watcher Works Internally

For the curious — here's what's happening under the hood.

### watchfiles (Uvicorn)

Rust library. Uses OS-native filesystem APIs:
- **macOS**: FSEvents (kernel-level file event stream)
- **Linux**: inotify (kernel-level filesystem notification)
- **Windows**: ReadDirectoryChangesW

Near-zero CPU usage when idle. Detection is event-driven, not polling. Events are debounced by uvicorn's `--reload-delay` (default 0.25s).

### watchdog (Temporal Workers via watchmedo)

Python library. Also uses OS-native APIs (FSEvents, inotify), but the `--interval=1` flag adds a 1-second polling interval on top. This means changes may take up to 1 second to be detected, compared to watchfiles' near-instant detection.

### Turbopack (Next.js)

Rust-based bundler. Maintains a **module graph** in memory. When a file changes, it traces the dependency graph to find all affected modules and recompiles only those. The compiled update is pushed to the browser over WebSocket, where the React Fast Refresh runtime swaps the modules in-place.

Key difference from uvicorn: Turbopack does **incremental recompilation** (only changed modules), while uvicorn does **full process restart** (everything re-imported from scratch).

### Node.js `--watch`

Built into Node.js. Tracks the module dependency graph (every file loaded via `require()` or `import`). When a tracked file changes, the entire Node.js process exits and re-executes. Simpler than the others — no incremental compilation, no module swapping, just a clean restart.
