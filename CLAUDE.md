# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Legacy modernization showcase: a .NET Framework 4.8 Web Forms app embeds a React 19 SPA that consumes a .NET 10 Web API with SignalR real-time presence.

```
LegacyHost.WebForms (.NET 4.8)
  └── embeds → people-app.js (React bundle from LpListSpa)
                  └── calls → ModernApi (.NET 10) — REST + SignalR
```

The React bundle is generated at build time by MSBuild targets in `LpListSpa.csproj` and copied into `src/LegacyHost.WebForms/ClientApp/`. Never edit files in `ClientApp/` directly.

## Build Commands

```bash
# Full solution (builds API, bundles React, copies to Web Forms)
dotnet build CIBC.slnx

# React bundle only
dotnet build src/LpListSpa/LpListSpa.csproj

# API only
dotnet build src/ModernApi/ModernApi.csproj
```

## Running Locally

```bash
# Start the API (required before anything else)
dotnet run --project src/ModernApi --launch-profile https
# https://localhost:7114  http://localhost:5135

# React standalone dev (HMR, proxies /api and /hubs to API)
cd src/LpListSpa && npm run dev
# http://localhost:5173

# LegacyHost.WebForms — Visual Studio + IIS Express only (no CLI)
```

## Testing (Playwright, TypeScript)

Tests are in `tests/`. Playwright auto-starts both backend services.

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
```

Override URLs: `PLAYWRIGHT_BASE_URL`, `PLAYWRIGHT_API_URL`, `PLAYWRIGHT_API_READY_URL`.

## Linting

```bash
cd src/LpListSpa && npm run lint
```

No `.editorconfig` or StyleCop for C#. C# projects use `<Nullable>enable</Nullable>` and implicit usings.

## Architecture

### ModernApi (`src/ModernApi/`)
- **Minimal APIs** — all endpoints in `Program.cs`, no controllers.
- SignalR hub at `/hubs/peopleEditing` (`PeopleEditing/PeopleEditingHub.cs`) for live editing presence.
- `PeopleEditingTracker` is a singleton backed by `ConcurrentDictionary` — the only stateful component.
- Data is **in-memory** (seeded `List<Person>` with object-level locking). No database.
- Domain types are sealed C# records: `Person`, `UpdatePersonRequest`, `PersonEditingPresence`.
- CORS allows `localhost` origins only.

### LpListSpa (`src/LpListSpa/`)
- Vite **library mode** — entry point is `src/widget.jsx`, output is single `people-app.js`.
- Exports `mountPeopleApp(container, options)` and `autoMountPeopleApps(selector)`.
- `peoplePresence.js` contains the SignalR connection factory.
- API base URL passed at mount time from Web Forms (read from `Web.config` by `Default.aspx.cs`).
- `src/main.jsx` is only used for standalone Vite dev server.

### LegacyHost.WebForms (`src/LegacyHost.WebForms/`)
- `Default.aspx.cs` reads `ModernApiBaseUrl` from `Web.config` `<appSettings>`.
- React bundle loaded via `<script>` in `Site.Master`.
- API URL config: `<add key="ModernApiBaseUrl" value="https://localhost:7114" />` in `Web.config`.

### Build Orchestration
- `LpListSpa.csproj` MSBuild targets: `EnsureNodeModules` → `BuildSpa` → `SyncSpaToLegacyHost`.
- `LegacyHost.WebForms.csproj` has `BuildEmbeddedPeopleApp` target that builds LpListSpa first.
- After React changes, rebuild with `dotnet build src/LpListSpa/LpListSpa.csproj` then refresh browser.

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
