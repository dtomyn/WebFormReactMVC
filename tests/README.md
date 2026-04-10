# Playwright Tests

This folder contains Playwright TypeScript coverage for both shared-bundle hosts and the backing API:

- `specs/default-page.page.spec.ts` exercises `Default.aspx` and the injected React people directory.
- `specs/modern-host.page.spec.ts` exercises `ModernHost.Mvc` and the shared React people directory.
- `specs/mixed-host.page.spec.ts` opens both hosts together and verifies cross-host behavior.
- `specs/modern-api.api.spec.ts` exercises `ModernApi` at `/api/people`.

## Install

From this `tests` folder:

```powershell
npm install
npm run install:browsers
```

## Run

The default configuration uses the local ports already defined in this repo:

- Legacy host: `http://localhost:63755`
- ModernHost.Mvc: `https://localhost:7184`
- ModernApi: `https://localhost:7114`
- API readiness check: `https://localhost:7114/api/people`

Run the full suite with a fresh solution build:

```powershell
npm test
```

Useful targeted commands:

```powershell
npm run capture:side-by-side
npm run test:page
npm run test:api
npm run test:headed
npm run test:ui
```

`npm run test:page` now covers:

- legacy-host rendering and accessibility
- MVC-host rendering and host labeling
- mixed-host SignalR presence behavior

`playwright.config.ts` starts all required backend processes automatically when they are not already running:

- `ModernApi` via `dotnet run --project src/ModernApi --launch-profile https`
- `LegacyHost.WebForms` via IIS Express and `.vs/WebFormReactMVC.slnx/config/applicationhost.config`
- `ModernHost.Mvc` via `dotnet run --project src/ModernHost.Mvc/ModernHost.Mvc.csproj --launch-profile https`

## Side-By-Side Screenshots

Run this from `tests` when you want a single browser page that shows both hosts side by side and captures the live editing presence state:

```powershell
npm run capture:side-by-side
```

The screenshot flow:

- starts `ModernApi`, `LegacyHost.WebForms`, and `ModernHost.Mvc`
- loads both hosts into side-by-side iframes on a single Playwright page
- opens the edit dialog in the legacy host
- confirms the modern host shows the active editor presence before editing the same person
- opens the same person in the modern host and confirms each dialog shows the other editor
- writes PNG screenshots to `tests/test-results/captures/`
- writes the animated GIF to `docs/mixed-host-side-by-side-presence.gif`

Expected files:

- `mixed-host-side-by-side-loaded.png`
- `mixed-host-side-by-side-left-editing.png`
- `mixed-host-side-by-side-dual-editing.png`
- `../docs/mixed-host-side-by-side-presence.gif`

If local HTTPS certificates are not trusted yet, run:

```powershell
dotnet dev-certs https --trust
```

## Environment Overrides

You can override the built-in local defaults with environment variables:

- `PLAYWRIGHT_BASE_URL`
- `PLAYWRIGHT_MVC_BASE_URL`
- `PLAYWRIGHT_API_URL`
- `PLAYWRIGHT_API_READY_URL`
- `PLAYWRIGHT_LEGACY_HOST_COMMAND`
- `PLAYWRIGHT_API_COMMAND`
- `PLAYWRIGHT_IIS_EXPRESS_PATH`
- `PLAYWRIGHT_APPLICATIONHOST_PATH`