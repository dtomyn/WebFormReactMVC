# Copilot Instructions

## Repository Overview

This is a **legacy modernization showcase**: a .NET Framework 4.8 Web Forms app (`LegacyHost.WebForms`) that embeds a React SPA (`LpListSpa`) which in turn consumes a .NET 10 Web API (`ModernApi`). The three projects are orchestrated by `WebFormReactMVC.slnx`.

```
LegacyHost.WebForms (.NET 4.8)
  └── embeds → people-app.js (React bundle, built from LpListSpa)
                  └── calls → ModernApi (.NET 10) — REST + SignalR
```

The React bundle is **generated at build time** by MSBuild targets in `LpListSpa.csproj` and copied into `src/LegacyHost.WebForms/ClientApp/`. Never edit files in `ClientApp/` directly.

---

## Build

```powershell
# Full solution (builds API, bundles React, copies to Web Forms)
dotnet build WebFormReactMVC.slnx

# React bundle only
dotnet build src/LpListSpa/LpListSpa.csproj

# API only
dotnet build src/ModernApi/ModernApi.csproj
```

---

## Running Locally

```powershell
# Start the API (required before opening the legacy host)
dotnet run --project src/ModernApi --launch-profile https
# Listens on https://localhost:7114 and http://localhost:5135
```

- **React SPA standalone** (dev only, with HMR): `cd src/LpListSpa && npm run dev` → http://localhost:5173
- **LegacyHost.WebForms**: Must be launched via Visual Studio with IIS Express (F5). CLI execution is not supported for Web Forms projects.

---

## Testing

Tests live in `tests/` and use **Playwright (TypeScript)**. Playwright auto-starts both backend services when running tests.

```powershell
cd tests
npm install
npm run install:browsers    # first time only

npm test                    # build solution + run all tests
npm run test:api            # API tests only (specs/*.api.spec.ts)
npm run test:page           # UI tests only (specs/*.page.spec.ts)
npm run test:headed         # UI tests with visible browser
npm run test:ui             # interactive Playwright UI mode

# Run a single test by name
npx playwright test --grep "your test name here"

# Run a specific test file
npx playwright test specs/modern-api.api.spec.ts

npm run typecheck           # TypeScript type-check without running tests
```

Override service URLs via environment variables: `PLAYWRIGHT_BASE_URL`, `PLAYWRIGHT_API_URL`, `PLAYWRIGHT_API_READY_URL`.

---

## Linting

```powershell
# Lint the React SPA
cd src/LpListSpa
npm run lint
```

C# projects use nullable reference types (`<Nullable>enable</Nullable>`) and implicit usings. No `.editorconfig` or StyleCop configuration is present.

---

## Architecture & Key Patterns

### ModernApi (`src/ModernApi/`)

- Uses **Minimal APIs** in `Program.cs` — all endpoints defined there (no controllers).
- **SignalR hub** at `/hubs/peopleEditing` (`PeopleEditing/PeopleEditingHub.cs`) for real-time presence tracking.
- `PeopleEditingTracker` is a **singleton** service backed by `ConcurrentDictionary` — it is the only stateful component.
- Data is **in-memory only** (a seeded `List<Person>` with object-level locking for mutation). There is no database.
- Domain types are **C# records**: `Person`, `UpdatePersonRequest`, `PersonEditingPresence`.
- OpenAPI is mapped only in the Development environment (`app.MapOpenApi()`).
- CORS allows `localhost` origins only (configured in `Program.cs`).

### LpListSpa (`src/LpListSpa/`)

- Built with **Vite** as a **library** (not a standard SPA): the entry point is `src/widget.jsx`, output is a single `people-app.js` file.
- `widget.jsx` exports `mountPeopleApp(container, options)` and `autoMountPeopleApps(selector)` — the Web Forms page calls `autoMountPeopleApps('[data-people-app]')` on load.
- `peoplePresence.js` contains the SignalR connection factory (`createPeoplePresenceConnection`).
- The API base URL is passed in at **mount time** from Web Forms (`Default.aspx.cs` reads it from `Web.config`).
- `src/main.jsx` is only used for the standalone Vite dev server.

### LegacyHost.WebForms (`src/LegacyHost.WebForms/`)

- `Default.aspx.cs` reads `ModernApiBaseUrl` from `Web.config` `<appSettings>` and writes it into the page as a `data-api-base-url` attribute.
- The React bundle (`ClientApp/people-app.js`) is referenced via a `<script>` tag in the master page.
- To change the API URL in development, edit `Web.config`: `<add key="ModernApiBaseUrl" value="..." />`.

---

## Conventions

- **Test file naming**: `*.page.spec.ts` for UI tests, `*.api.spec.ts` for API tests.
- **CSS**: kebab-case with BEM notation (e.g., `.presence-panel__header`).
- **C# naming**: PascalCase for types/records, standard .NET conventions throughout.
- **React naming**: camelCase functions, PascalCase components, JSX (not TSX).
- All SignalR WebSocket traffic proxied through `/hubs` in Vite dev mode (see `vite.config.js`).
