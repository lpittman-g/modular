# AGENTS

`modular` (published as `modular-scripts`) is a Yarn-workspaces monorepo
containing the `modular` CLI toolchain plus supporting libraries and project
templates. There is no server/database — the "product" is the CLI/library set
under `packages/`.

## Cursor Cloud specific instructions

Dependencies are already installed by the startup update script
(`yarn install`). Standard commands live in `package.json` (`scripts`),
`CONTRIBUTING.md`, and `.github/workflows/`.

### Toolchain

- Package manager is Yarn Classic (`yarn` 1.x); do not switch to npm/pnpm/Yarn
  Berry.
- CI runs on Node 16/18/20 (`.github/workflows/test.yml`). The VM default Node
  22 satisfies the repo's `engines` (`>=16.10.0`) and works for
  install/build/lint/test/`modular start`. A fresh login shell already exposes a
  working `node` + `yarn` (via nvm default), so no version pinning is needed.

### Build / lint / test / run

- Build all publishable packages: `yarn build` (see `package.json`).
- Lint: `yarn lint` (a jest-based ESLint runner; not a plain eslint invocation).
- Tests must be run single-process. Use the `yarn test` wrapper (it sets
  `NODE_OPTIONS=--max_old_space_size=5120` and `--runInBand --env=node`). The
  full suite is large and slow (includes Puppeteer browser tests, and an
  optional Verdaccio integration flow).
  - Test prerequisite: build the internal `workspace-resolver` first, exactly as
    CI does: `yarn workspace @modular-scripts/workspace-resolver build` (its
    `dist`/`dist-cjs` is git-ignored and absent on a fresh checkout).
    `yarn build` also covers this.
  - To scope tests, select by path regex:
    `yarn modular test --regex="packages/<name>"`. Passing a bare package name
    relies on changed-file detection and often selects nothing ("No workspaces
    found in selection") on a clean tree.
- Run an app in dev mode: `yarn modular start <appName>` (default port 3000). In
  a headless VM set `BROWSER=none` so it does not try to open a desktop browser.

### Gotcha: `yarn build` creates a duplicate workspace

`yarn build` runs `prepare:remote-view` which copies a publish artifact to
`packages/remote-view/dist/`. Because workspaces glob is `packages/**`, that
`dist/package.json` registers a SECOND workspace named
`@modular-scripts/remote-view`. This breaks any command that runs a workspace
install (`yarn install`, `modular add`, `modular start <new pkg>`) with
`error There are more than one workspace with name "@modular-scripts/remote-view"`,
and causes jest haste-map collision warnings. Fix by removing the git-ignored
artifact before install-dependent commands: `rm -rf packages/remote-view/dist`
(regenerate later with `yarn prepare:remote-view`).

### Notes

- `create-modular-react-app` scaffolds a project that installs `modular-scripts`
  from the public npm registry (network + published version). To exercise THIS
  repo's source instead, drive the local CLI directly via `yarn modular ...`
  inside the repo.
- Full end-to-end integration against a local registry is optional and scripted
  under `integration-test-scripts/` (Verdaccio on `http://localhost:4873` +
  `forever`).
