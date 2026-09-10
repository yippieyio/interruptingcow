# Development Environment

> **Status: design phase.** No toolchain files exist yet (`package.json`,
> `.nvmrc`, `.devcontainer/`, `.env.example`). This describes the **intended**
> setup. When the skeleton lands, this document is the reference and the
> devcontainer makes most of it automatic.

## Toolchain

| Tool | How it's pinned | Why |
|---|---|---|
| **Node.js** | `.nvmrc` / `.node-version` — the current LTS at the time | the daemon runtime ([ADR 0001](../adr/0001-all-typescript.md)); LTS for multi-month uptime |
| **pnpm** | `package.json` `packageManager` field; provided by Corepack | workspace management; deterministic installs |
| **Podman** | not pinned; a recent version | local PostgreSQL, the end-to-end pod, the release image ([ADR 0005](../adr/0005-podman-pod-external-state.md)) |
| **`psql`** | not pinned | inspecting the dev database (optional but handy) |
| **Playwright browsers** | installed via `pnpm exec playwright install` | end-to-end tests |

Match Node with `nvm use` (or `fnm`, `asdf`) and enable pnpm with
`corepack enable`.

## The devcontainer _(planned)_

`.devcontainer/` will provide a ready environment for VS Code Dev Containers and
GitHub Codespaces:

- The pinned Node LTS, pnpm via Corepack, Podman, the `psql` client, and the
  Playwright browsers, all preinstalled.
- The workspace opened with the recommended extensions.
- `pnpm install` and `pnpm db:up && pnpm db:migrate` run (or one command away) so
  a fresh container is ready to `pnpm dev`.

Using the devcontainer is the fastest path and the one we support first. The
manual path below is kept working for people who don't want a container.

## Manual setup _(planned)_

```sh
git clone git@github.com:<you>/InterruptingCow.git
cd InterruptingCow
git remote add upstream https://github.com/OWNER/InterruptingCow.git

nvm use
corepack enable
pnpm install

cp .env.example .env
pnpm db:up          # Podman postgres for development
pnpm db:migrate
pnpm dev
```

## Configuration model

- All configuration comes from **environment variables** (a `.env` file in
  development, real environment in production).
- The daemon **validates the whole configuration against a zod schema at
  startup** and refuses to start on anything invalid, with a message pointing at
  the offending key. "It started" therefore means "the config is valid".
- **`.env.example` is the canonical list.** Every key appears there with a
  comment and either a safe default or an obvious placeholder. Adding a config
  key without adding it to `.env.example` is an incomplete change.

Indicative keys (the real list lives in `.env.example` once it exists):

| Key | Purpose |
|---|---|
| `ICOW_HTTP_PORT` | port the daemon listens on (behind the proxy) |
| `ICOW_DATABASE_URL` | PostgreSQL connection string |
| `ICOW_JWT_SECRET` | signing secret for access tokens |
| `ICOW_REFRESH_TTL` / `ICOW_ACCESS_TTL` | token lifetimes |
| `ICOW_WORLD_CRED_KEY` | key material for encrypting stored game credentials (from a host secret, not the image) |
| `ICOW_RECOVERY_GRACE_SECONDS` | the our-fault recovery grace window ([ADR 0006](../adr/0006-connection-recovery-semantics.md)) |
| `ICOW_LOG_LEVEL` | `error` / `warn` / `info` / `debug` |

Secrets never go in `.env.example`, in the repository, in logs, or in the
container image.

## The development database

- A **Podman PostgreSQL** container, started with `pnpm db:up`, using a
  **container-local** volume (dev data is disposable — unlike production, see
  [ADR 0005](../adr/0005-podman-pod-external-state.md)).
- `pnpm db:migrate` applies migrations; `pnpm db:reset` drops and recreates for a
  clean slate.
- Integration tests spin up their own throwaway database (or schema) so they
  don't depend on your dev data being in a particular state.

## The fake MU\* server

`pnpm fake-mush` runs the fixture from `e2e/fixtures/` standalone — a small TCP
server that speaks enough telnet to negotiate options and can be scripted to
send canned output or drop the connection. Point a dev world at it to work on
the client or the session layer without touching a real game. CI uses the same
fixture; **CI never connects to a real MU\* server.**

## Editor setup

- `.editorconfig` covers basic whitespace rules for any editor.
- A `.vscode/extensions.json` _(planned)_ recommends the ESLint, Prettier, and
  relevant language extensions.
- Formatting is done by Prettier on save or via `pnpm format`; don't fight it in
  review.

## Operating-system notes

- **macOS / Linux:** first-class. Podman runs natively on Linux; on macOS it uses
  a managed VM (`podman machine`).
- **Windows:** use WSL2 and treat it as Linux. Native Windows is not a supported
  development target.
