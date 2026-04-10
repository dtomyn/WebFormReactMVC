# WebFormReactMVC Solution Setup

This solution contains four projects that work together:

- `src/ModernApi`: a .NET 10 Web API that serves a paged, filterable people directory backed by deterministic in-memory seed data.
- `src/LpListSpa`: a React app that renders the shared people directory with server-side paging, filtering, editing, and SignalR presence.
- `src/LegacyHost.WebForms`: a .NET Framework Web Forms app that loads the built React bundle and shows the API data inside the legacy page.
- `src/ModernHost.Mvc`: a .NET 10 ASP.NET Core MVC app that loads the same built React bundle and shows the same API data and SignalR presence in a Razor host.

## Quick Start

From the solution root, run either of these:

```powershell
.\run-all.ps1
```

or:

```cmd
run-all.cmd
```

The launcher does all of the following:

1. builds `WebFormReactMVC.slnx`
2. starts `ModernApi` on `https://localhost:7114`
3. starts `ModernHost.Mvc` on `https://localhost:7184`
4. starts `LegacyHost.WebForms` with IIS Express when available
5. opens the MVC and legacy host pages in your browser

Optional switches:

- `-SkipBuild`: skip the build and reuse the current outputs
- `-SkipLegacyHost`: start only `ModernApi` and `ModernHost.Mvc`
- `-NoBrowser`: do not automatically open browser tabs

Examples:

```powershell
.\run-all.ps1 -SkipBuild
.\run-all.ps1 -SkipLegacyHost
.\run-all.ps1 -SkipBuild -NoBrowser
```

If IIS Express is not installed, the script still starts the API and MVC host and prints the manual fallback for the legacy host.

## How The Integration Works

The React component is not hard-coded into either host project.

Instead, the flow is:

1. `LpListSpa` builds a React bundle.
2. That bundle is copied into `src/LegacyHost.WebForms/ClientApp` and `src/ModernHost.Mvc/wwwroot/people-app`.
3. `src/LegacyHost.WebForms/Default.aspx` and `src/ModernHost.Mvc/Views/Home/Index.cshtml` load that bundle.
4. The bundle calls `ModernApi` at `https://localhost:7114/api/people`.
5. The API returns a paged envelope of people plus metadata.
6. The React component renders the current page of results inside the Web Forms page and the MVC page.

SignalR is also used for live editing presence:

1. Each browser tab connects to `ModernApi` over SignalR.
2. When a user opens an edit dialog, that tab announces which person is being edited.
3. Other open pages immediately see who is currently editing which person.
4. When the dialog is closed or saved, the editing presence is cleared.

## Prerequisites

Install all of the following before trying to run the full end-to-end flow:

1. .NET 10 SDK
2. Node.js and npm
3. Visual Studio 2022 or later with ASP.NET and web development support
4. .NET Framework 4.8 targeting pack

Important:

- `ModernApi` and `ModernHost.Mvc` can be run from the CLI with `dotnet run`.
- `LegacyHost.WebForms` is a classic ASP.NET Web Forms project. The included launcher starts it with IIS Express when IIS Express is installed.
- Both host apps are configured to call `https://localhost:7114` by default.

Before the first run, trust the local .NET development certificate if needed:

```powershell
dotnet dev-certs https --trust
```

## One-Time Setup

From the solution root, run:

```powershell
dotnet build WebFormReactMVC.slnx
```

What this does:

- builds `ModernApi`
- builds `ModernHost.Mvc`
- builds `LpListSpa`
- creates the React bundle in `src/LpListSpa/dist`
- copies the built files into `src/LegacyHost.WebForms/ClientApp`
- copies the built files into `src/ModernHost.Mvc/wwwroot/people-app`

After a successful build, these files should exist:

- `src/LegacyHost.WebForms/ClientApp/people-app.js`
- `src/LegacyHost.WebForms/ClientApp/people-app.css`
- `src/ModernHost.Mvc/wwwroot/people-app/people-app.js`
- `src/ModernHost.Mvc/wwwroot/people-app/people-app.css`

If those files do not exist, the React component cannot appear inside the corresponding host page.

## Manual Run Steps

Use these only if you do not want to use `run-all.ps1`.

### Step 1: Start ModernApi

From the solution root:

```powershell
dotnet run --project src/ModernApi --launch-profile https
```

Wait until you see:

- `Now listening on: https://localhost:7114`

You can verify the API by opening this URL in a browser or calling it from PowerShell:

- `https://localhost:7114/api/people?page=1&pageSize=25`

Expected result: JSON containing a paged envelope with `items`, `page`, `pageSize`, `totalCount`, `totalPages`, `hasPreviousPage`, `hasNextPage`, and `appliedFilters`.

### Step 2A: Run ModernHost.Mvc From The CLI

From the solution root:

```powershell
dotnet run --project src/ModernHost.Mvc/ModernHost.Mvc.csproj --launch-profile https
```

Wait until you see:

- `Now listening on: https://localhost:7184`

Then open:

- `https://localhost:7184`

When the MVC host page opens, it will:

- load `people-app.js` from `/people-app`
- mount the shared React component
- call `ModernApi`
- render a paged, filterable directory backed by `/api/people`

### Step 2B: Open The Solution In Visual Studio For LegacyHost.WebForms

Open:

- `WebFormReactMVC.slnx`

Make sure `LegacyHost.WebForms` is the startup project if you want the browser to open directly into the legacy app.

### Step 3: Run LegacyHost.WebForms

In Visual Studio:

1. Right-click `LegacyHost.WebForms`
2. Choose `Set as Startup Project`
3. Press `F5` or click `IIS Express`

Visual Studio will launch the Web Forms site.

When `Default.aspx` opens, the page will:

- load `people-app.js`
- mount the shared React component into the legacy page
- call `ModernApi`
- render a paged, filterable directory backed by `/api/people`

If everything is working, you will see the people directory rendered inside the Web Forms page.

### Optional: Run Both Hosts Side By Side

You can run both hosts at the same time to compare behavior and verify shared SignalR presence across them:

- `https://localhost:7184`
- `http://localhost:63755/Default.aspx`

## Fastest Reliable Run Sequence

The fastest dependable path is now:

```powershell
.\run-all.ps1
```

If you want to skip the legacy host and only run the API plus MVC host:

```powershell
.\run-all.ps1 -SkipLegacyHost
```

## When You Change The React App

If you edit files under `src/LpListSpa/src`, you must rebuild so the updated bundle is copied into both host projects.

Use either of these:

```powershell
dotnet build src/LpListSpa/LpListSpa.csproj
```

or:

```powershell
dotnet build WebFormReactMVC.slnx
```

Either option rebuilds the SPA bundle and refreshes:

- `src/LegacyHost.WebForms/ClientApp`
- `src/ModernHost.Mvc/wwwroot/people-app`

After that, refresh whichever host pages are currently running.

## Optional: Run The React App By Itself

If you want to work on the React UI outside either host:

```powershell
cd src/LpListSpa
npm run dev
```

Then open the Vite URL shown in the terminal, usually:

- `http://localhost:5173`

In this mode, Vite proxies `/api` and `/hubs` requests to `https://localhost:7114`, so `ModernApi` must still be running.

This is useful for frontend iteration, but it is not the host-integrated scenario.

## Run Automated Tests

Playwright coverage now includes:

- `LegacyHost.WebForms`
- `ModernHost.Mvc`
- paged API contract coverage for `ModernApi`
- a mixed-host scenario that verifies shared SignalR editing presence across both hosts

From the `tests` folder:

```powershell
npm install
npm run install:browsers
npm test
```

For targeted commands and environment overrides, see `tests/README.md`.

## Test Live Editing Presence

To verify that other users can see who is editing which person across either host:

1. Start `ModernApi`
2. Start `ModernHost.Mvc`, `LegacyHost.WebForms`, or both
3. Open any two pages or tabs from this set:
   - `https://localhost:7184`
   - `http://localhost:63755/Default.aspx`
4. Use the search box or pager if needed to locate a person, then click `Edit`
5. In the other page, look at the `Who is editing right now` section and the person card badge

Expected behavior:

- each page gets its own default display name
- opening the dialog shows that user as currently editing the selected person
- the presence appears in the same host and in the other host if both are running
- closing or saving the dialog removes that presence from the other page

The display name is stored per browser tab session, so two tabs can appear as different users during local testing.

## Config That Controls The Host-to-API Connections

The Web Forms host reads the API base URL from:

- `src/LegacyHost.WebForms/Web.config`

Current setting:

```xml
<add key="ModernApiBaseUrl" value="https://localhost:7114" />
```

The MVC host reads the API base URL from:

- `src/ModernHost.Mvc/appsettings.Development.json`
- `src/ModernHost.Mvc/appsettings.json`

Current development setting:

```json
"ModernApiBaseUrl": "https://localhost:7114"
```

Both hosts should point to the same `ModernApi` URL when you run them side by side.

The API launch URL is defined in:

- `src/ModernApi/Properties/launchSettings.json`

Current HTTPS URL:

- `https://localhost:7114`

The MVC host launch URLs are defined in:

- `src/ModernHost.Mvc/Properties/launchSettings.json`

Current HTTPS URL:

- `https://localhost:7184`

## Troubleshooting

### A host page opens but the React component does not appear

Check all of the following:

1. If you are testing Web Forms, `src/LegacyHost.WebForms/ClientApp/people-app.js` and `people-app.css` exist
2. If you are testing MVC, `src/ModernHost.Mvc/wwwroot/people-app/people-app.js` and `people-app.css` exist
3. You rebuilt after changing the React app
4. Browser dev tools do not show a 404 for `/ClientApp/people-app.js` or `/people-app/people-app.js`

### The React component appears but no data is shown

Check all of the following:

1. `ModernApi` is still running
2. `https://localhost:7114/api/people?page=1&pageSize=25` returns JSON
3. The active host configuration still points to `https://localhost:7114`
   - `src/LegacyHost.WebForms/Web.config` for Web Forms
   - `src/ModernHost.Mvc/appsettings.Development.json` for MVC
4. The browser does not show a certificate warning or blocked HTTPS request

### The API call fails because of certificate trust

Run:

```powershell
dotnet dev-certs https --trust
```

Then restart the API and reload the page.

### The launcher could not start the legacy host

Check all of the following:

1. IIS Express exists at `C:\Program Files\IIS Express\iisexpress.exe` or set `PLAYWRIGHT_IIS_EXPRESS_PATH`
2. If you already opened the solution in Visual Studio, the `.vs\<solution>\config\applicationhost.config` file still exists
3. If IIS Express is available but the launcher still cannot start the legacy host, open `WebFormReactMVC.slnx` in Visual Studio and run `LegacyHost.WebForms` with IIS Express once

## Verified Baseline

This repository was verified with:

- `dotnet build WebFormReactMVC.slnx`
- `dotnet run --project src/ModernApi --launch-profile https`
- `dotnet run --project src/ModernHost.Mvc/ModernHost.Mvc.csproj --launch-profile https`
- `cd tests && npm run typecheck`

The API returned the expected in-memory list of five people, the build successfully produced the React assets consumed by both hosts, and the Playwright page coverage now includes legacy-host, MVC-host, and mixed-host scenarios.