# Plan: Add a .NET 10 MVC Host for the Shared React People App

## Goal

Add a new ASP.NET Core MVC application under `src/` that:

- hosts the same React people directory currently embedded in `LegacyHost.WebForms`
- reuses `ModernApi` for both REST and SignalR
- keeps `LpListSpa` as the single source of truth for the React UI
- does not duplicate the people component, API contract, or presence logic

This is a hosting change, not a rewrite.

## Current Baseline

- `src/LpListSpa` builds an embeddable Vite bundle named `people-app.js` plus `people-app.css`.
- `src/LegacyHost.WebForms/Default.aspx` renders a mount element with `data-people-app` and `data-api-base-url`.
- `src/LpListSpa/src/peopleApp.jsx` auto-mounts into any `[data-people-app]` container.
- `src/LpListSpa/src/peoplePresence.js` derives both `/api` and `/hubs/peopleEditing` URLs from the provided API base URL.
- `src/ModernApi` already owns the REST endpoints and SignalR hub.

Because the React bundle only needs a DOM mount point and `data-api-base-url`, an MVC host can reuse the existing bundle with little or no React code change.

## Recommended Approach

Create a thin MVC host project, likely `src/ModernHost.Mvc`, and keep responsibilities separated as follows:

- `LpListSpa`: shared React UI and bundling
- `ModernApi`: API and SignalR hub
- `LegacyHost.WebForms`: legacy host that continues to embed the shared bundle
- `ModernHost.Mvc`: new ASP.NET Core MVC host that embeds the same shared bundle

The MVC app should behave like the Web Forms page from the user perspective, but the host page and layout should be native Razor/MVC.

## Proposed Project Shape

Add a new project:

```text
src/
  ModernHost.Mvc/
    Controllers/
      HomeController.cs
    Models/
      HomePageViewModel.cs
    Views/
      Home/
        Index.cshtml
      Shared/
        _Layout.cshtml
    wwwroot/
      people-app/
        people-app.js
        people-app.css
    appsettings.json
    appsettings.Development.json
    Program.cs
    ModernHost.Mvc.csproj
```

Add the new project to `WebFormReactMVC.slnx`.

## Architecture Decisions

### 1. Keep `ModernApi` as the backend and SignalR host

Do not move the REST endpoints or the `/hubs/peopleEditing` hub into the MVC app in the first iteration.

Reasoning:

- it preserves the current working API contract
- it avoids duplicating stateful presence logic
- it keeps the MVC work focused on hosting only
- the React client already expects an external API base URL and works with cross-origin SignalR

### 2. Reuse the current mount contract

The MVC view should render the same mount element contract already used by Web Forms:

```html
<div data-people-app="true" data-api-base-url="...">
</div>
```

That allows the same `autoMountPeopleApps()` entry point to work unchanged.

### 3. Keep the bundle prebuilt and host it as static assets

Per current ASP.NET Core guidance, the MVC app should serve the React bundle from `wwwroot` using `UseStaticFiles()`.

Recommended MVC asset location:

- `wwwroot/people-app/people-app.js`
- `wwwroot/people-app/people-app.css`

This avoids mixing generated assets with MVC code and keeps the host-side include paths explicit.

### 4. Defer same-origin proxying unless it becomes necessary

For the first cut, the MVC app can pass `ModernApiBaseUrl` exactly as Web Forms does today.

That means:

- MVC host origin: `https://localhost:<mvc-port>`
- API and SignalR origin: `https://localhost:7114`

This already fits the current client code and current local CORS policy.

If later you want the MVC host to expose `/api` and `/hubs` on the same origin, add that as a second-step enhancement through a reverse proxy or backend-for-frontend pattern. It should not block the initial MVC host delivery.

## Implementation Plan

### Phase 1: Scaffold the MVC host

Create `src/ModernHost.Mvc` as an ASP.NET Core MVC project targeting `net10.0`.

Minimum host setup:

- `builder.Services.AddControllersWithViews()`
- `app.UseHttpsRedirection()`
- `app.UseStaticFiles()`
- `app.UseRouting()`
- `app.MapControllerRoute(...)`

Notes:

- `UseStaticWebAssets()` is only needed if shared static assets are later moved into a referenced library. It is not required if the bundle is copied directly into this app's `wwwroot`.
- Add `ModernApiBaseUrl` to configuration with a development default of `https://localhost:7114`.

### Phase 2: Add the MVC host page that embeds the shared React bundle

Create a simple `HomeController` and `Index.cshtml` that mirror the Web Forms page responsibilities:

- render host page chrome and explanatory text
- include `people-app.css`
- render the `[data-people-app]` mount element
- include `people-app.js` as a module script
- pass `ModernApiBaseUrl` into the DOM

Recommended view model:

- `HomePageViewModel`
  - `string PeopleApiBaseUrl`

Recommended Razor output shape:

- link tag for `~/people-app/people-app.css`
- mount element with `data-people-app` and `data-api-base-url`
- script tag with `type="module"` for `~/people-app/people-app.js`

Use `asp-append-version="true"` on the CSS and JS references so browser caching does not hide bundle updates.

### Phase 3: Extend the shared asset build pipeline

Update `src/LpListSpa/LpListSpa.csproj` so the existing build output is copied to both hosts.

Current behavior:

- build bundle into `src/LpListSpa/dist`
- copy bundle into `src/LegacyHost.WebForms/ClientApp`

Target behavior:

- build bundle into `src/LpListSpa/dist`
- copy bundle into `src/LegacyHost.WebForms/ClientApp`
- copy bundle into `src/ModernHost.Mvc/wwwroot/people-app`

Recommended change:

- replace the single-host copy target with a multi-destination target
- keep `LpListSpa` as the only place that knows how the React bundle is produced
- continue treating host-side generated assets as build artifacts and never edit them directly

This preserves a single React build path for both host applications.

### Phase 4: Wire configuration cleanly

Match the existing Web Forms behavior by making the API base URL configurable in the MVC host.

Recommended configuration keys:

- `ModernApiBaseUrl` in `appsettings.json`
- overrideable via `appsettings.Development.json` or environment variables

Recommended rule:

- MVC should never hard-code API or hub URLs in the view
- the view should only emit the configured value into `data-api-base-url`

### Phase 5: Validate SignalR and CORS behavior

The current client uses SignalR with `withCredentials: false`, and the current API CORS policy allows localhost origins.

For local development, this should continue to work as-is once the MVC host runs on a localhost origin.

Validation checklist:

- MVC page loads the shared React component
- `/api/people` data appears in the new host
- editing presence appears in the MVC host
- MVC and Web Forms pages can both observe the same presence state at the same time
- saving in one host reflects in the other host after reload or refetch

For anything beyond local development, tighten CORS to explicit configured origins instead of broad localhost matching.

### Phase 6: Add test coverage for the new host

Extend Playwright rather than creating a separate test stack.

Recommended test updates:

- add a new Playwright web server entry for `ModernHost.Mvc`
- add `PLAYWRIGHT_MVC_BASE_URL`
- add a new page spec for the MVC host
- refactor shared assertions so Web Forms and MVC can reuse the same behavior checks

Suggested test scenarios:

- page renders the people directory in MVC
- edit dialog opens and remains accessible
- SignalR presence updates between two MVC pages
- SignalR presence updates between MVC and Web Forms pages

The cross-host presence test is especially important because it proves both hosts are truly using the same backend and hub.

## Deliverables

The first implementation pass should produce:

- a new `src/ModernHost.Mvc` project
- MVC page hosting the shared React bundle
- `WebFormReactMVC.slnx` updated to include the new host
- shared build pipeline copying bundle assets into both hosts
- Playwright coverage for the MVC host path
- run instructions updated in `README.md`

## Non-Goals for the First Pass

Do not include these in the initial MVC delivery unless a separate requirement appears:

- moving API endpoints into the MVC project
- moving the SignalR hub into the MVC project
- rewriting the React app for MVC-specific rendering
- replacing Web Forms
- introducing server-side rendering for the React component
- introducing authentication or authorization changes unrelated to hosting

## Risks and Watchouts

### Generated asset drift

If the asset copy step is only updated for one host, the two hosts will drift and show different UI behavior.

Mitigation:

- keep a single SPA build and multi-copy target in `LpListSpa.csproj`

### Cache confusion during development

Fixed output names such as `people-app.js` can be cached aggressively by browsers.

Mitigation:

- use `asp-append-version`
- rebuild the SPA as part of host builds where practical

### Cross-origin assumptions hidden in tests

Current tests focus on Web Forms. MVC could appear to work visually while SignalR or API calls fail due to origin mismatches.

Mitigation:

- add explicit MVC and mixed-host Playwright coverage

## Recommended Delivery Order

1. Create the MVC project and get a plain Razor page running.
2. Copy the built React assets into MVC `wwwroot/people-app`.
3. Render the same mount contract and confirm the React component loads.
4. Verify API calls and SignalR presence from MVC.
5. Update Playwright to cover MVC-only and mixed-host scenarios.
6. Update `README.md` with run instructions for all three hosts.

## Summary

The lowest-risk plan is to treat the MVC application as a second host for the existing shared React bundle, not as a new frontend implementation. The React app stays in `LpListSpa`, the backend and SignalR hub stay in `ModernApi`, and the new MVC project only needs to provide Razor layout, static asset hosting, configuration, and test coverage.