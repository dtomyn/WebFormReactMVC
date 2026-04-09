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