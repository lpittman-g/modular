# AGENTS.md

## Cursor Cloud specific instructions

`modular` is a monorepo (Yarn 1 classic workspaces) that builds a CLI toolchain
for micro-frontend / monorepo development. The two shipped products are the
`modular-scripts` CLI (`packages/modular-scripts`) and
`create-modular-react-app` (`packages/create-modular-react-app`). Most other
`packages/*` are templates and internal helpers consumed by those two.

The startup update script already runs `yarn install --frozen-lockfile` and
builds the internal prerequisite `@modular-scripts/workspace-resolver`, so
dependencies are ready when a session starts.

### Running the standard commands

All commands are defined in the root `package.json` and are invoked through the
local CLI via `ts-node` (no build step needed to run them):

- Typecheck: `yarn typecheck`
- Lint: `yarn lint` (autofix with `yarn lint:fix`)
- Prettier: `yarn prettier --check .`
- Build (all publishable packages): `yarn build`
- Tests: `yarn test` (see caveats below)

`@modular-scripts/workspace-resolver` must be built before
typecheck/lint/build/test (the update script and CI both do this). If you see
missing-type errors from that package, run
`yarn workspace @modular-scripts/workspace-resolver build`.

### Non-obvious gotchas

- `yarn build` copies the built `remote-view` output into
  `packages/remote-view/dist`, which contains a `package.json` named
  `@modular-scripts/remote-view`. This gitignored directory makes a subsequent
  `yarn install` fail with
  `There are more than one workspace with name "@modular-scripts/remote-view"`
  and makes Jest emit a Haste "naming collision" warning. If you ran
  `yarn build` and then need to reinstall, remove it first:
  `rm -rf packages/remote-view/dist`.
- `yarn test` uses `cross-env`; run it via the `yarn test` script (not by
  prefixing `cross-env` yourself in the shell — it is not on `PATH`). The suite
  runs in-band and is heavy (many tests scaffold, build, and start real apps).
  To iterate quickly, filter with the CLI's `--regex` flag, e.g.
  `yarn modular test --runInBand --env=node --regex="formatPath|getAllFiles"`.
  Positional args to `modular test` are treated as workspace selectors, not Jest
  file patterns.

### Running / demoing the dev server

There is no long-running app inside this repo; the "application" is the CLI plus
the dev server it produces. To exercise the dev server end to end, scaffold an
app and start it:

- Scaffold: `yarn create-modular-react-app /tmp/hello-modular --repo false`
  (this installs the published `modular-scripts` into the new project).
- Start it with the published scripts: `cd /tmp/hello-modular && yarn start`.
- To instead exercise the **local** `modular-scripts` `start` against that app,
  run from the app dir:
  `cd /tmp/hello-modular && TS_NODE_PROJECT=/workspace/tsconfig.json BROWSER=none PORT=3000 node -r /workspace/node_modules/ts-node/register /workspace/packages/modular-scripts/src/cli.ts start app`
  Modular locates the project root from the current working directory, so you
  must `cd` into the target app first. Set `BROWSER=none` so it does not try to
  launch a browser.
