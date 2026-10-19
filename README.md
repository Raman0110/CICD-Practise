# CI/CD Practise

A small Express and TypeScript application for practising a CI/CD workflow with
GitHub Actions. The repository currently contains a complete continuous
integration (CI) pipeline; continuous deployment (CD) can be connected through
a hosting provider such as Render.

## Application overview

The application exposes two routes:

| Route | Purpose |
| --- | --- |
| `GET /` | Displays a simple status page and the current application version. |
| `GET /api/health` | Returns `{"status":"ok"}` with HTTP 200 for tests and deployment health checks. |

The server listens on `0.0.0.0` and uses the port supplied in `PORT`, falling
back to port `3000`. The home page reads `APP_VERSION`, or displays
`development` when the variable is not set.

## CI/CD workflow

```text
Developer change
      |
      v
Push to main or pull request targeting main
      |
      v
GitHub Actions: CI / Verify
      |
      +--> Check out source
      +--> Install Node.js 24 and enable the npm cache
      +--> Install exact locked dependencies with npm ci
      +--> Type-check --> Test --> Build
      |
      v
Verified commit
      |
      v
Optional external deployment (not configured in this repository)
```

### 1. Workflow triggers

The workflow is defined in [`.github/workflows/ci.yml`](.github/workflows/ci.yml).
It runs for:

- every push to `main`;
- every pull request whose target branch is `main`.

Changes pushed to other branches do not start CI unless those changes are part
of a pull request into `main`. GitHub automatically cancels neither older runs
nor duplicate runs because no concurrency policy is currently configured.

### 2. Job environment and permissions

The workflow contains one job, `verify`, displayed in GitHub as **Verify**. It:

- runs on GitHub's latest Ubuntu runner;
- has a 10-minute timeout;
- receives read-only access to repository contents;
- uses Node.js 24, matching the `engines.node` entry in `package.json`;
- enables the npm download cache based on `package-lock.json`.

The npm cache speeds up package downloads; it does not reuse `node_modules`.
Each run still performs a clean dependency installation.

### 3. Dependency installation

`npm ci` installs the exact dependency tree recorded in `package-lock.json`.
Unlike `npm install`, it fails if `package.json` and the lockfile disagree and
does not rewrite the lockfile. This makes CI installations reproducible.

When dependencies change, commit both `package.json` and `package-lock.json`.

### 4. Verification pipeline

The job invokes `npm run verify`, which runs these checks sequentially:

1. `npm run typecheck` runs `tsc --noEmit`. It validates the TypeScript source
   without writing compiled files.
2. `npm test` starts the Express app on an automatically selected local port,
   requests `/api/health`, verifies the HTTP status and JSON response, and then
   closes the test server.
3. `npm run build` runs `tsc`, compiling `src/**/*.ts` into `dist/`.

The commands are joined with `&&`, so the first failure stops the pipeline and
causes the GitHub Actions job to fail. The generated `dist/` directory is used
only to prove that the application builds; it is ignored by Git and is not
uploaded as a workflow artifact.

### 5. Merge protection

A green workflow does not by itself prevent a failing pull request from being
merged. To enforce CI, add a branch protection rule or ruleset for `main` in
GitHub and require the **CI / Verify** status check before merging. Repository
rules are configured in GitHub, so they are not represented in this codebase.

### 6. Continuous deployment

There is currently no deployment job, deployment manifest, or provider
configuration in the repository. A passing push to `main` therefore verifies
the application but does not deploy it by itself.

For a Git-connected Render web service, the equivalent service settings are:

| Setting | Value |
| --- | --- |
| Runtime | Node |
| Branch | `main` |
| Build command | `npm ci && npm run build` |
| Start command | `npm start` |
| Health check path | `/api/health` |

Render supplies `PORT`; the application already reads it. `APP_VERSION` is
optional and can be set to a release identifier or commit SHA. If automatic
deploys are enabled in Render, the end-to-end path becomes:

1. A pull request runs CI before review and merge.
2. The merge creates a push to `main`, which runs CI again.
3. Render detects the new commit and builds the service.
4. Render starts `dist/server.js` through `npm start` and checks
   `/api/health`.

Because provider-side automatic deployment is independent of this GitHub
Actions workflow, configure Render to wait for CI checks before deploying if
that option is available. Otherwise it may begin building while CI is still
running. An alternative is to add an explicit deployment job that depends on
`verify`; that requires provider credentials to be stored as GitHub Actions
secrets.

## Run locally

### Requirements

- Node.js 24
- npm

Install the locked dependencies:

```bash
npm ci
```

Start the development server with automatic restarts:

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Stop the server with
`Ctrl+C`.

To reproduce the complete CI verification locally, run:

```bash
npm run verify
```

Run the production build locally with:

```bash
npm run build
npm start
```

## Scripts

| Command | Description |
| --- | --- |
| `npm run dev` | Runs the TypeScript server in watch mode with `tsx`. |
| `npm run typecheck` | Checks TypeScript without producing build output. |
| `npm test` | Runs the health endpoint integration test. |
| `npm run build` | Compiles TypeScript from `src/` into `dist/`. |
| `npm start` | Runs the compiled `dist/server.js` application. |
| `npm run verify` | Runs type-checking, tests, and the production build in CI order. |

## Environment variables

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `PORT` | No | `3000` | HTTP port. Hosting providers normally supply this value. |
| `APP_VERSION` | No | `development` | Version label displayed on the home page. |

For example, in PowerShell:

```powershell
$env:APP_VERSION = "local-test"
$env:PORT = "4000"
npm run dev
```

## Diagnosing failures

- **`npm ci` fails:** regenerate and commit `package-lock.json` after changing
  dependencies, then retry locally with `npm ci`.
- **Type-checking fails:** run `npm run typecheck` and fix the reported strict
  TypeScript error.
- **The test fails:** run `npm test` and confirm `/api/health` still returns
  HTTP 200 with exactly `{"status":"ok"}`.
- **The build fails:** run `npm run build`; TypeScript will report the source
  file and line that prevented output.
- **A deployment is unhealthy:** check the provider logs, confirm the build
  completed, ensure the start command is `npm start`, and verify the provider
  supplies a valid `PORT`.
