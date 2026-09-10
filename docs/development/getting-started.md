# Getting Started (developers)

> **Status: design phase.** There is no code, no `package.json`, and no build
> yet. This document describes the **intended** developer workflow so it can be
> reviewed now and is ready when the skeleton lands. Steps that depend on
> not-yet-existing tooling are marked _(planned)_.

## What you can do today

- Read the [architecture](../architecture.md) and [protocol](../protocol.md)
  documents and the [ADRs](../adr/0000-adr-process.md), and open issues or a
  [Discussion][discussions] where they're unclear, incomplete, or wrong.
- Propose a design change via the [ADR process](../adr/0000-adr-process.md).
- Improve the documentation directly with a PR.

Everything below is what the loop will look like once there is an
implementation.

## Prerequisites _(planned)_

| Tool | Version | Notes |
|---|---|---|
| Node.js | current LTS (pinned in `.nvmrc`) | use `nvm`/`fnm`/`asdf` to match |
| pnpm | as pinned in `package.json` `packageManager` | `corepack enable` will provide it |
| Podman | recent | for the local PostgreSQL and the end-to-end pod |
| `psql` client | any recent | convenience for inspecting the dev database |
| Git | any recent | — |

The [environment guide](environment.md) has the details and a **devcontainer**
that provides all of this preconfigured (VS Code / GitHub Codespaces).

## First-time setup _(planned)_

```sh
# 1. Fork the repo on GitHub, then clone your fork
git clone git@github.com:<you>/InterruptingCow.git
cd InterruptingCow
git remote add upstream https://github.com/OWNER/InterruptingCow.git

# 2. Match the Node version and install dependencies
nvm use            # reads .nvmrc
corepack enable
pnpm install       # installs the whole workspace

# 3. Configure
cp .env.example .env
#   The defaults are meant to work out of the box against the dev database
#   below. Every key is documented in .env.example.

# 4. Start a throwaway PostgreSQL for development
pnpm db:up         # starts a Podman postgres with a host-less volume
pnpm db:migrate    # applies migrations
```

## The daily loop _(planned)_

```sh
pnpm dev           # runs the daemon and the web client with reload
pnpm verify        # lint + typecheck + unit + integration tests — run before every push
```

Other scripts (the single task interface — nothing requires memorised
incantations):

| Script | Does |
|---|---|
| `pnpm dev` | daemon + web client, watch mode |
| `pnpm build` | production build of all packages |
| `pnpm verify` | `format:check` + `lint` + `typecheck` + `test` + `test:integration` |
| `pnpm test` | unit tests (fast) |
| `pnpm test:integration` | integration tests (needs the dev PostgreSQL) |
| `pnpm test:e2e` | end-to-end suite (builds a pod, runs Playwright) |
| `pnpm db:up` / `pnpm db:down` | start / stop the dev PostgreSQL |
| `pnpm db:migrate` / `pnpm db:reset` | apply migrations / drop and re-create |
| `pnpm fake-mush` | run the fake MU\* server fixture standalone |
| `pnpm format` | apply formatting |
| `pnpm docs:serve` | preview the docs site _(once a generator is added)_ |

## Making a change _(planned)_

1. Branch from an up-to-date `main`:
   ```sh
   git fetch upstream && git switch -c feat/my-change upstream/main
   ```
2. Write the change **and its tests**. See
   [coding standards](coding-standards.md) and [testing](testing.md).
   - New behavior → unit tests, and integration tests if it touches the database
     or the socket layer.
   - Bug fix → a test that fails before your fix and passes after.
   - Schema change → a migration **with a tested rollback**.
   - Behavior/config/protocol change → update the docs in the same PR.
3. `pnpm verify` until green.
4. Commit with a [Conventional Commit](../../CONTRIBUTING.md#commit-messages)
   message and a **DCO sign-off**:
   ```sh
   git commit -s -m "feat(session): describe the change"
   ```
5. Push to your fork and open a PR against `upstream/main`. Fill in the PR
   template; link the issue (`Closes #123`); link the ADR if relevant.
6. Address review. A maintainer squash-merges once checks pass and it has an
   approving review.

## Where things live

See [Repository layout](../architecture.md#repository-layout). In short:

- `packages/protocol` — the wire contract (zod schemas + types + codec).
- `packages/daemon` — the service. The core is `src/session/`.
- `packages/web` — the Angular PWA.
- `e2e/` — end-to-end tests and the fake MU\* server.
- `deploy/` — the Podman pod and proxy config.
- `docs/` — you are here.

## Getting help

- [GitHub Discussions][discussions] for questions.
- Not the issue tracker for questions — see [`SUPPORT.md`](../../SUPPORT.md).

[discussions]: https://github.com/OWNER/InterruptingCow/discussions
