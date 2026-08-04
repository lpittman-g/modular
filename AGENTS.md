# AGENTS.md

## Cursor Cloud specific instructions

### What this repo is

`modular` is JP Morgan's monorepo toolchain for micro-frontends (a CLI plus a
set of publishable npm packages under `packages/**`). There is **no backend
service or database** — the "application" is the `modular` CLI and the dev
server it launches (`modular start`, default port `3000`). Standard commands
live in `package.json` scripts and `CONTRIBUTING.md`; don't duplicate them, just
use them (`yarn build`, `yarn lint`, `yarn typecheck`, `yarn test`).

### Toolchain (Node 20 + Yarn Classic)

- Package manager is **Yarn Classic (v1)**; `yarn.lock` is authoritative. Do not
  introduce npm/pnpm lockfiles.
- The repo targets Node `16 || 18 || 20` (CI uses those). The base VM image's
  default `node` on `PATH` (`/exec-daemon/node`) is **Node 22**, which is _not_
  a CI-validated version. Interactive shells are configured (via `~/.bashrc`) to
  run `nvm use 20`, so `node -v` should already report v20 and `yarn` resolves
  to the Node 20 Yarn Classic. If you ever see Node 22, run `nvm use 20` before
  building or testing.

### Running tests — non-obvious gotcha (Browserslist)

The pinned `caniuse-lite` is old relative to the current date and prints a
`caniuse-lite is outdated` **warning to stderr**. Tests that spin up a dev
server (e.g. `packages/modular-scripts/src/__tests__/start.test.ts`) use a
harness that treats **any** dev-server stderr output as a fatal error, so they
fail with that warning alone. Always export `BROWSERSLIST_IGNORE_OLD_DATA=true`
when running tests/builds to suppress it. Do **not** run
`npx update-browserslist-db` to "fix" it — that mutates a pinned dependency.

Other test notes:

- Tests must run single-process; `yarn test` already passes `--runInBand`. The
  full suite is heavy (launches Puppeteer/Chromium). Scope it with
  `--regex <path>` or package names, e.g.
  `yarn test --regex "packages/remote-view/"`.
- `modular test` is **selective** by default (it only runs workspaces changed
  vs the default branch), so a bare `yarn modular test <pkg>` can print
  `No workspaces found in selection`. A `--regex <path>` value overrides this
  (it is passed to Jest as a positional `testPathPattern`, i.e. a path match).
  To pass arguments straight through to Jest for a specific file/dir, use
  `--bypass`, e.g.
  `yarn modular test --bypass --runInBand --env=node <path/to/test-or-dir>`.
- The heavy `packages/modular-scripts/src/__tests__/*` integration tests spawn
  real builds and dev servers on fixed ports (3000/4000); if interrupted they
  can leave orphaned servers that cause "Something is already running on port …"
  on the next run.
- Skip network/update checks with `CI=true SKIP_MODULAR_STARTUP_CHECK=true`.
- Puppeteer ships its own Chromium (downloaded at install) and launches headless
  with `--no-sandbox` in this VM.

Example full test invocation:

```
CI=true SKIP_MODULAR_STARTUP_CHECK=true BROWSERSLIST_IGNORE_OLD_DATA=true yarn test --regex "<path-or-package>"
```

### Build artifact collides with `yarn install`

`yarn build` copies `dist/modular-scripts-remote-view` into
`packages/remote-view/dist`, whose `package.json` is also named
`@modular-scripts/remote-view`. Because workspaces glob `packages/**`, a later
`yarn install` then fails with _"There are more than one workspace with name
@modular-scripts/remote-view"_ (and Jest prints a harmless `jest-haste-map`
naming collision warning). If you need to re-install after building, first
`rm -rf packages/remote-view/dist`. The startup update script already does this
before `yarn install`.

### Verifying the environment end to end

The canonical "hello world" is the README Getting Started flow: scaffold a
project with the local CLI and start its dev server, e.g.

```
yarn create-modular-react-app /tmp/hello-modular --prefer-offline
cd /tmp/hello-modular
BROWSERSLIST_IGNORE_OLD_DATA=true SKIP_MODULAR_STARTUP_CHECK=true yarn start app
```

then open http://localhost:3000. Editing `packages/app/src/App.tsx` hot-reloads.
