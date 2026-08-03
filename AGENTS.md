# AGENTS.md

## Cursor Cloud specific instructions

This repo is **Modular** (`modular-scripts`), a Yarn v1 **workspaces monorepo**
that ships a developer CLI (build/test/lint/start/analyze) plus supporting
libraries and templates. There is **no long-running application server and no
database** — the "app" you can run is a scaffolded micro-frontend served by the
`modular start` dev server. Standard commands live in `package.json`
(`scripts`), `CONTRIBUTING.md`, and `.github/workflows/{test,integration}.yml`;
prefer those as the source of truth. Notes below are the non-obvious gotchas.

### Environment

- The update script runs `yarn install` (installs deps, applies `patch-package`
  patches, installs husky hooks). Puppeteer's Chromium is downloaded during
  install and is used by browser tests.
- Node: the VM default (Node 22) works for install/build/lint/test/dev server.
  CI targets Node 16/18/20; `engines` requires `>=20` among those. Chromium runs
  headless fine here.

### Build (required before running tests)

- Tests import the compiled `@modular-scripts/workspace-resolver`, so build that
  prerequisite first: `yarn workspace @modular-scripts/workspace-resolver build`
  (this is exactly what CI does before `yarn test`). The update script does NOT
  build it, so run it once per fresh pod.
- `yarn build` (the full build) additionally compiles CMRA + modular-scripts
  and, as a publish-prep step, copies remote-view output into
  `packages/remote-view/dist`.

### GOTCHA: full `yarn build` creates a duplicate workspace

- `yarn build`'s `prepare:remote-view:copy` step writes
  `packages/remote-view/dist/package.json`, which the `packages/**` workspace
  glob then picks up as a **second** workspace named
  `@modular-scripts/remote-view`. This breaks subsequent Yarn/Modular commands
  with
  `There are more than one workspace with name "@modular-scripts/remote-view"`
  and produces a `jest-haste-map: Haste module naming collision` warning.
- Fix: `rm -rf packages/remote-view/dist` before running `yarn install`,
  `modular add`, `modular start`, or a full test run. The dir is gitignored, so
  removing it is safe. If you only need to run tests, prefer building just the
  workspace-resolver prereq (above) instead of the full `yarn build`.

### Running tests

- Tests MUST run serially — the `test` script already uses `--runInBand`
  (parallelization is not supported). Full suite (as CI): `yarn test`. Note
  `yarn test` uses `cross-env`; invoke it via a yarn script (or `npx cross-env`)
  rather than calling `cross-env` directly.
- `modular test` is **selective** (runs only workspaces changed vs the default
  branch), so a bare `yarn modular test <pkg>` often reports
  `No workspaces found in selection`. `--regex` matches test _names_, not paths.
- To run a specific file/dir, bypass selective logic and pass the path straight
  to Jest:
  `yarn modular test --bypass --runInBand --env=node <path/to/test-or-dir>`
  (e.g. `packages/remote-view/src/__tests__/`).
- The heavy `packages/modular-scripts/src/__tests__/*` integration tests spawn
  real builds and dev servers on fixed ports (3000/4000); they are slow and, if
  interrupted, can leave orphaned servers that cause "Something is already
  running on port …" on the next run.

### Running the app (dev server)

- Scaffold a micro-frontend, then start it:
  `yarn modular add <name> --unstable-type app --template app` then
  `BROWSER=none yarn modular start <name>` (serves on `http://localhost:3000`;
  set `BROWSER=none` so it doesn't try to open a desktop browser).
  `modular start` is a yarn/workspace operation, so clear the
  duplicate-workspace gotcha above first.

### Integration / E2E (Verdaccio) — optional, heavy

- The true publish→install→CLI-runthrough E2E lives in
  `integration-test-scripts/` and needs a local Verdaccio registry on port 4873
  (`setupVerdaccio.sh` installs `verdaccio` + `forever` globally). Not needed
  for normal development or for the checks above.
