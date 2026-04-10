# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Legacy modernization showcase: a shared React 19 people-directory SPA is hosted by both a .NET Framework 4.8 Web Forms app and an ASP.NET Core MVC app, consuming a .NET 10 Web API with SignalR real-time editing presence.

```
LegacyHost.WebForms (.NET 4.8)   ──┐
                                    ├── embed → people-app.js (React bundle from LpListSpa)
ModernHost.Mvc (.NET 10)          ──┘           └── calls → ModernApi (.NET 10) — REST + SignalR
```

The React bundle (`people-app.js` + `people-app.css`) is generated at build time by MSBuild targets in `LpListSpa.csproj` and copied into both `src/LegacyHost.WebForms/ClientApp/` and `src/ModernHost.Mvc/wwwroot/people-app/`. Never edit files in those output directories directly.

## Build Commands

```bash
# Full solution (builds API, MVC host, bundles React, copies to both hosts)
dotnet build WebFormReactMVC.slnx

# React bundle only
dotnet build src/LpListSpa/LpListSpa.csproj

# API only
dotnet build src/ModernApi/ModernApi.csproj

# MVC host only
dotnet build src/ModernHost.Mvc/ModernHost.Mvc.csproj
```

## Running Locally

The easiest way to start everything is `run-all.ps1`:

```powershell
# Start all services (builds solution, launches API, MVC host, and Legacy host)
./run-all.ps1

# Useful switches
./run-all.ps1 -SkipBuild          # reuse current build
./run-all.ps1 -SkipLegacyHost     # start only API + MVC host
./run-all.ps1 -NoBrowser          # don't open browser tabs
```

Or start services individually:

```bash
# Start the API (required before anything else)
dotnet run --project src/ModernApi --launch-profile https
# https://localhost:7114  http://localhost:5135

# Start the MVC host
dotnet run --project src/ModernHost.Mvc --launch-profile https
# https://localhost:7184

# React standalone dev (HMR, proxies /api and /hubs to API)
cd src/LpListSpa && npm run dev
# http://localhost:5173

# LegacyHost.WebForms — Visual Studio + IIS Express only (no CLI)
# http://localhost:63755
```

## Testing (Playwright, TypeScript)

Tests are in `tests/`. Playwright auto-starts all three backend services (ModernApi, LegacyHost.WebForms, ModernHost.Mvc).

```bash
cd tests
npm install                # first time
npm run install:browsers   # first time

npm test                   # build solution + run all tests
npm run test:api           # API tests only (*.api.spec.ts)
npm run test:page          # UI tests only (*.page.spec.ts)
npm run test:headed        # UI tests with visible browser
npm run test:ui            # interactive Playwright UI mode

# Single test by name
npx playwright test --grep "test name"

# Single test file
npx playwright test specs/modern-api.api.spec.ts

npm run typecheck          # TS type-check without running tests

# Capture side-by-side presence demo GIF
npm run capture:side-by-side
```

Override URLs: `PLAYWRIGHT_BASE_URL`, `PLAYWRIGHT_MVC_BASE_URL`, `PLAYWRIGHT_API_URL`, `PLAYWRIGHT_API_READY_URL`.

Playwright projects: `legacy-host-chromium` (page tests) and `modern-api` (API tests).

## Linting

```bash
cd src/LpListSpa && npm run lint
```

No `.editorconfig` or StyleCop for C#. C# projects use `<Nullable>enable</Nullable>` and implicit usings.

## Architecture

### ModernApi (`src/ModernApi/`)
- **Minimal APIs** — endpoint routing in `Program.cs`, no controllers.
- **GET `/api/people`** — server-side paged directory. Query params: `page`, `pageSize` (10/25/50/100), `search`, `location`, `role`, `sortBy`, `sortDirection`. Returns `PeoplePageResponse` with items, pagination metadata, and `appliedFilters`.
- **PUT `/api/people/{id:int}`** — update a person record with validation.
- **SignalR hub** at `/hubs/peopleEditing` (`PeopleEditing/PeopleEditingHub.cs`) for live editing presence.
- `PeopleEditingTracker` is a singleton backed by `ConcurrentDictionary` — the only stateful component.
- Data is **in-memory** (2,500 deterministically seeded people via `PeopleSeedData.cs`). No database.
- Domain types in `People/` folder: `Person`, `UpdatePersonRequest`, `PeopleQueryRequest`, `PeoplePageResponse`, `PeopleDirectoryService`, `PeopleSeedData`.
- `PeopleDirectoryService` handles paging, filtering, and sorting with object-level locking.
- OpenAPI enabled in development (`/openapi/v1.json`).
- CORS allows `localhost` origins only.

### LpListSpa (`src/LpListSpa/`)
- Vite **library mode** — entry point is `src/widget.jsx`, output is `people-app.js` + `people-app.css`.
- Exports `mountPeopleApp(container, options)` and `autoMountPeopleApps(selector)`.
- UI includes server-side paging, search, location/role filters, sort controls, and page-size selector.
- `peoplePresence.js` contains the SignalR connection factory.
- API base URL passed at mount time from each host.
- `src/main.jsx` is only used for standalone Vite dev server.

### LegacyHost.WebForms (`src/LegacyHost.WebForms/`)
- `Default.aspx.cs` reads `ModernApiBaseUrl` from `Web.config` `<appSettings>`.
- React bundle loaded via `<script>` in `Site.Master`.
- API URL config: `<add key="ModernApiBaseUrl" value="https://localhost:7114" />` in `Web.config`.

### ModernHost.Mvc (`src/ModernHost.Mvc/`)
- ASP.NET Core MVC app that hosts the same shared React bundle.
- `HomeController` serves the landing page; `HomePageViewModel` exposes the configured API base URL.
- React bundle loaded from `wwwroot/people-app/` in `_Layout.cshtml`.
- API URL config: `ModernApiBaseUrl` in `appsettings.json` / `appsettings.Development.json`.
- Runs on `https://localhost:7184`.

### Build Orchestration
- `LpListSpa.csproj` MSBuild targets: `EnsureNodeModules` → `BuildSpa` → `SyncSpaToLegacyHost` (also copies to `ModernHost.Mvc/wwwroot/people-app/`).
- `LegacyHost.WebForms.csproj` has `BuildEmbeddedPeopleApp` target that builds LpListSpa first.
- After React changes, rebuild with `dotnet build src/LpListSpa/LpListSpa.csproj` then refresh browser.

## Test Files

- `specs/default-page.page.spec.ts` — Legacy Web Forms host: paging, filtering, editing presence.
- `specs/modern-host.page.spec.ts` — MVC host: rendering, paging, query params, filtering/sorting.
- `specs/mixed-host.page.spec.ts` — Cross-host: shared bundle contract, shared SignalR presence.
- `specs/side-by-side-presence.page.spec.ts` — Visual capture: screenshots both hosts, generates demo GIF.
- `specs/modern-api.api.spec.ts` — API contract: paging, filtering, sorting, validation, error handling.
- `specs/people-directory-test-helpers.ts` — Shared helpers: `PeoplePagePayload`, `waitForPeopleResponse()`, `readPeoplePayload()`, `expectSharedDirectoryChrome()`.

## Conventions

- **Test files**: `*.page.spec.ts` for UI tests, `*.api.spec.ts` for API tests.
- **CSS**: kebab-case with BEM notation (`.presence-panel__header`).
- **C#**: PascalCase, sealed records for data types, nullable reference types enabled.
- **React**: camelCase functions, PascalCase components, JSX (not TSX).
- SignalR traffic proxied through `/hubs` in Vite dev mode (see `vite.config.js`).

## Prerequisites

- .NET 10 SDK
- Node.js and npm
- Visual Studio 2022+ with ASP.NET/web development (for Web Forms only)
- .NET Framework 4.8 targeting pack
