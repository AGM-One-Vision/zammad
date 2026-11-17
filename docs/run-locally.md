---
owner: group:default/dto
last_reviewed: 2026-11-17
---

# Run locally

Zammad ships its own developer guide. This page tells you which AGM branch to
use and links the upstream steps. Do not copy production data to your machine:
the production database holds customer personal data.

## Prerequisites

- Git and the GitHub CLI, with read access to `AGM-One-Vision/zammad`.
- **Recommended path (devcontainer):** Docker Desktop or Docker CLI with at
  least 8 GB of memory for Docker, and VS Code with the Dev Containers
  extension. Upstream guide:
  [Devcontainer Setup](https://github.com/zammad/zammad/blob/develop/doc/developer_manual/development_environment/devcontainer-setup.md).
- **Manual path:** the versions this code expects:
    - Ruby `3.4.9` (`.ruby-version`, `Gemfile`), Rails `~> 8.0.0`;
    - Node.js `>= 24` and pnpm `>= 10` (`package.json` → `engines`,
      `packageManager: pnpm@10.33.2`);
    - PostgreSQL, Redis and (for search) Elasticsearch.

    Upstream guide:
  [Manual Setup](https://github.com/zammad/zammad/blob/develop/doc/developer_manual/development_environment/manual-setup.md).

## Steps

1. Clone the AGM fork and switch to the branch production is built from:

    ```sh
    git clone https://github.com/AGM-One-Vision/zammad.git
    cd zammad
    git switch feature/editor-highlight-textalign
    ```

    Use `develop` only to work on the DevHub docs. `develop` does not contain
    the AGM code changes.

2. Start the environment.
    - Devcontainer: open the folder in VS Code and click **Reopen in
      Container**. The container starts PostgreSQL, Redis and Elasticsearch
      and seeds the database.
    - Manual: follow the upstream manual setup guide.

3. Start Zammad:
    - devcontainer: run `dev` in a terminal inside the container;
    - manual setup: run `bin/dev`. It starts the processes in `Procfile.dev`
      (Vite, Rails web server, websocket server, background worker, Sass
      watcher).

4. Open <http://localhost:3000> (the devcontainer serves
   <https://localhost:3000>). Log in with the default development account from
   the upstream
   [Getting Started](https://github.com/zammad/zammad/blob/develop/doc/developer_manual/development_environment/getting-started.md)
   guide (step 5, "Login").

5. Check the AGM changes:
    - open a ticket and reply. The new desktop view editor toolbar has a
      highlight button and a text-align menu. The classic view's text bubble
      menu has colour, highlight and alignment actions;
    - in **Reporting**, the **Download** link should return an `.xlsx` file
      (HTTP 200), not HTTP 422.

To build the production image locally, see
[Deploy & operate → How to deploy](deploy-operate.md#how-to-deploy).

## Test data

- The devcontainer seeds a development database on first start.
- To reset it, run `rails zammad:bootstrap:reset`. See the upstream
  [Development Workflow](https://github.com/zammad/zammad/blob/develop/doc/developer_manual/development_environment/development-workflow.md)
  guide.
- There is no anonymised copy of production data. Never restore a production
  dump on a laptop: it contains customer personal data (PII).

TODO(owner): should AGM provide a small anonymised sample (groups, custom ticket
fields, Core Workflows) so developers can test AGM-specific configuration?

## Common problems

| Problem | Cause and fix |
|---|---|
| Blank browser window on first start | The first run compiles assets. Wait a few minutes. |
| The editor shows no highlight or alignment buttons | You are on `develop`. Switch to `feature/editor-highlight-textalign` and run `pnpm install` again; the branch adds two tiptap packages. |
| Reporting **Download** fails with HTTP 422 "Required parameter 'profile' is missing" | You are on code without the report URL fix (`develop`, or any upstream 7.0.x). The fix is on the AGM branch and in upstream 7.1.0 and later. |
| `docker build` produces an image with an almost empty `public/assets/frontend` ("rake is empty") | The `assets:precompile` step used the gem's rake. The AGM branch's `Dockerfile` calls `/opt/zammad/bin/rake`; keep that change. |
| `docker build` fails with "the required build argument $COMMIT_SHA is missing" | Pass `--build-arg COMMIT_SHA=$(git rev-parse HEAD)`. |
| A clone made with `--depth 1` cannot compare branches | Fetch more history for the branches you need, for example `git fetch --depth 50 origin feature/editor-highlight-textalign develop`. |
