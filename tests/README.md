# Playwright Tests

This folder contains Playwright TypeScript coverage for the current legacy-hosted page and the backing API:

- `specs/default-page.page.spec.ts` exercises `Default.aspx` and the injected React people directory.
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
- ModernApi: `https://localhost:7114`
- API readiness check: `http://localhost:5135/api/people`

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

`playwright.config.ts` starts both backend processes automatically when they are not already running:

- `ModernApi` via `dotnet run --project src/ModernApi --launch-profile https`
- `LegacyHost.WebForms` via IIS Express and `.vs/CIBC.slnx/config/applicationhost.config`

If local HTTPS certificates are not trusted yet, run:

```powershell
dotnet dev-certs https --trust
```

## Environment Overrides

You can override the built-in local defaults with environment variables:

- `PLAYWRIGHT_BASE_URL`
- `PLAYWRIGHT_API_URL`
- `PLAYWRIGHT_API_READY_URL`
- `PLAYWRIGHT_LEGACY_HOST_COMMAND`
- `PLAYWRIGHT_API_COMMAND`
- `PLAYWRIGHT_IIS_EXPRESS_PATH`
- `PLAYWRIGHT_APPLICATIONHOST_PATH`