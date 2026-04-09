# CIBC Solution Setup

This solution contains three projects that work together:

- `ModernApi`: a .NET 10 Web API that returns an in-memory list of people.
- `LpListSpa`: a React app that renders the people list.
- `LegacyHost.WebForms`: a .NET Framework Web Forms app that loads the built React bundle and shows the API data inside the legacy page.

## How The Integration Works

The React component is not hard-coded into the Web Forms project.

Instead, the flow is:

1. `LpListSpa` builds a React bundle.
2. That bundle is copied into `LegacyHost.WebForms/ClientApp`.
3. `LegacyHost.WebForms/Default.aspx` loads that bundle.
4. The bundle calls `ModernApi` at `https://localhost:7114/api/people`.
5. The API returns the in-memory people list.
6. The React component renders that list inside the Web Forms page.

## Prerequisites

Install all of the following before trying to run the full end-to-end flow:

1. .NET 10 SDK
2. Node.js and npm
3. Visual Studio 2022 or later with ASP.NET and web development support
4. .NET Framework 4.8 targeting pack

Important:

- `ModernApi` can be run from the CLI with `dotnet run`.
- `LegacyHost.WebForms` is a classic ASP.NET Web Forms project. The practical way to run it is from Visual Studio with IIS Express or local IIS.
- The Web Forms app is configured to call `https://localhost:7114` by default.

## One-Time Setup

From the solution root, run:

```powershell
dotnet build CIBC.slnx
```

What this does:

- builds `ModernApi`
- builds `LpListSpa`
- creates the React bundle in `LpListSpa/dist`
- copies the built files into `LegacyHost.WebForms/ClientApp`

After a successful build, these files should exist:

- `LegacyHost.WebForms/ClientApp/people-app.js`
- `LegacyHost.WebForms/ClientApp/people-app.css`

If those files do not exist, the React component cannot appear inside the Web Forms page.

## Run The Full Scenario

To see the React component inside the legacy Web Forms page with live API data, use this order.

### Step 1: Start ModernApi

From the solution root:

```powershell
dotnet run --project ModernApi --launch-profile https
```

Wait until you see:

- `Now listening on: https://localhost:7114`

You can verify the API by opening this URL in a browser or calling it from PowerShell:

- `https://localhost:7114/api/people`

Expected result: JSON containing five people.

### Step 2: Open The Solution In Visual Studio

Open:

- `CIBC.slnx`

Make sure `LegacyHost.WebForms` is the startup project if you want the browser to open directly into the legacy app.

### Step 3: Run LegacyHost.WebForms

In Visual Studio:

1. Right-click `LegacyHost.WebForms`
2. Choose `Set as Startup Project`
3. Press `F5` or click `IIS Express`

Visual Studio will launch the Web Forms site.

When `Default.aspx` opens, the page will:

- load `people-app.js`
- mount the React component into the legacy page
- call `ModernApi`
- render the list returned by `/api/people`

If everything is working, you will see the people directory rendered inside the Web Forms page.

## Fastest Reliable Run Sequence

If you want the shortest dependable path, use exactly this:

```powershell
dotnet build CIBC.slnx
dotnet run --project ModernApi --launch-profile https
```

Then, in Visual Studio:

1. Open `CIBC.slnx`
2. Run `LegacyHost.WebForms` with IIS Express

## When You Change The React App

If you edit files under `LpListSpa/src`, you must rebuild so the updated bundle is copied into the Web Forms project.

Use either of these:

```powershell
dotnet build LpListSpa/LpListSpa.csproj
```

or:

```powershell
dotnet build LegacyHost.WebForms/LegacyHost.WebForms.csproj
```

Either option rebuilds the SPA bundle and refreshes `LegacyHost.WebForms/ClientApp`.

After that, refresh the browser page running the legacy app.

## Optional: Run The React App By Itself

If you want to work on the React UI outside the Web Forms host:

```powershell
cd LpListSpa
npm run dev
```

Then open the Vite URL shown in the terminal, usually:

- `http://localhost:5173`

In this mode, Vite proxies `/api` requests to `https://localhost:7114`, so `ModernApi` must still be running.

This is useful for frontend iteration, but it is not the legacy-hosted scenario.

## Config That Controls The Legacy-to-API Connection

The Web Forms host reads the API base URL from:

- `LegacyHost.WebForms/Web.config`

Current setting:

```xml
<add key="ModernApiBaseUrl" value="https://localhost:7114" />
```

That must match the API URL you are actually running.

The API launch URL is defined in:

- `ModernApi/Properties/launchSettings.json`

Current HTTPS URL:

- `https://localhost:7114`

## Troubleshooting

### The Web Forms page opens but the React component does not appear

Check all of the following:

1. `LegacyHost.WebForms/ClientApp/people-app.js` exists
2. `LegacyHost.WebForms/ClientApp/people-app.css` exists
3. You rebuilt after changing the React app
4. Browser dev tools do not show a 404 for `/ClientApp/people-app.js`

### The React component appears but no data is shown

Check all of the following:

1. `ModernApi` is still running
2. `https://localhost:7114/api/people` returns JSON
3. `LegacyHost.WebForms/Web.config` still points to `https://localhost:7114`
4. The browser does not show a certificate warning or blocked HTTPS request

### The API call fails because of certificate trust

Run:

```powershell
dotnet dev-certs https --trust
```

Then restart the API and reload the page.

### The legacy project will not run from the CLI

That is expected for classic Web Forms projects. Run `LegacyHost.WebForms` from Visual Studio using IIS Express.

## Verified Baseline

This repository was verified with:

- `dotnet build CIBC.slnx`
- `dotnet run --project ModernApi --launch-profile https`
- `https://localhost:7114/api/people`

The API returned the expected in-memory list of five people, and the build successfully produced the React assets consumed by the legacy Web Forms host.