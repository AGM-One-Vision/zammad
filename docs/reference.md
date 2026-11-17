---
owner: group:default/dto
last_reviewed: 2026-11-17
---

# Reference

## Configuration

Zammad reads its runtime settings from environment variables of the
containers. Non-secret settings are set in the compose file on the host,
`/opt/zammad/docker-compose.yml` (verify with `docker compose ls` first, see
[Deploy & operate](deploy-operate.md#how-to-deploy)). Secrets are held by
DevOps. This page lists names only, never values. The DevHub card's
**Config** tab is the standard place for configuration names; until it lists
Zammad's settings, the list lives here.

| Setting | Where it lives | Purpose |
|---|---|---|
| `POSTGRESQL_DB`, `POSTGRESQL_HOST`, `POSTGRESQL_PORT`, `POSTGRESQL_USER`, `POSTGRESQL_OPTIONS` | compose `environment` of every `zammad-*` service | Database connection. |
| `POSTGRESQL_PASS` | secret, held by DevOps (see internal audit) | Database password. Never paste it into docs, tickets or chat. |
| `REDIS_URL` | compose `environment` | Redis connection. |
| `MEMCACHE_SERVERS` | compose `environment` | Memcached connection. |
| `ELASTICSEARCH_HOST`, `ELASTICSEARCH_PORT`, `ELASTICSEARCH_SCHEMA`, `ELASTICSEARCH_NAMESPACE` | compose `environment` | Search index connection and index name prefix. |
| `ZAMMAD_FQDN`, `ZAMMAD_HTTP_TYPE`, `NGINX_SERVER_SCHEME` | compose `environment` | Public host name and scheme. `NGINX_SERVER_SCHEME` must be `https` behind the ALB, or secure session cookies break. |
| `RAILS_TRUSTED_PROXIES` | compose `environment` | Which proxy addresses Rails trusts for the client IP. |
| `TZ` | compose `environment` | Container time zone (Africa/Cairo). |
| `BACKUP_DIR`, `BACKUP_TIME`, `HOLD_DAYS` | compose `environment` of `zammad-backup` (and `zammad-scheduler`) | Daily `pg_dump` location, time and retention. |
| `COMMIT_SHA` | Docker build argument (`Dockerfile`) | Required at build time; added to `VERSION`. |
| Bridge settings (`Bridge`, `3CX`, `Admin`, `Zammad` sections) | See the bridge's own docs | Configuration of `3cx-zammad-bridge`. |
| 3CX credentials and the Zammad API token used by the bridge | secret, held by DevOps (see internal audit) | Bridge access to 3CX and to Zammad. |
| Alert routing | SNS topic `zammad-prod-alerts` (Terraform `zammad-alerts.tf`) | Forwarded to Microsoft Teams by the shared `agm-cloudwatch-teams` forwarder. |
| Application settings: `fqdn`, `http_type`, `es_url`, `es_index`, `locale_default`, `timezone_default`, `0100_notification`, email channels, triggers, Core Workflows, schedulers, overviews | Zammad database, changed in the admin UI (**Settings**, **Channels**, **Manage**) or with the REST API `/api/v1/settings` | Not in this repo. See [Changes outside the repo](#changes-outside-the-repo). |

## API

Zammad serves its own vendor APIs. AGM adds none.

- REST API under `/api/v1/` at <https://helpdesk.agmegypt.com>. Reference:
  <https://docs.zammad.org/en/latest/api/intro.html>.
- GraphQL API at `/graphql`, used by the new desktop and mobile views.
- CTI endpoint `/api/v1/cti/<token>`, used by 3cx-zammad-bridge for call
  events.

The API is registered in DevHub as `zammad-api` (a `kind: API` entity defined
in `agm-service-catalog/services/zammad.yaml`). This component lists it in
`providesApis`. There is no AGM OpenAPI file: the API is the upstream product
API, documented by Zammad.

Known callers: 3cx-zammad-bridge (CTI endpoint and REST API with an API token).
Other callers: TODO(owner) (see [Overview](index.md#who-uses-it)).

## Data

| Store | Where | Notes |
|---|---|---|
| PostgreSQL 17, database `zammad_production` | Docker volume `postgresql-data` on the instance root EBS volume (gp3, encrypted with KMS) | All tickets, users and configuration. Contains customer personal data (PII). Background jobs (`delayed_job`) are also stored here. |
| Elasticsearch 8.18 index (namespace `zammad`) | Docker volume `elasticsearch-data` | Can be rebuilt from PostgreSQL with `rake zammad:searchindex:rebuild`. |
| Redis | Docker volume `redis-data` | Key-value data used by the Rails services. |
| File storage | Docker volume `zammad-storage` (`/opt/zammad/storage`) | Used when attachments are stored on the filesystem. TODO(owner): confirm the storage backend set in the admin UI. |
| Daily `pg_dump` | Docker volume `zammad-backup` | 10 days kept. The audit proposes moving dumps to S3 (decision Z9). |
| AMI backups | AWS Backup plan `agm-connect-prod-zammad-daily` | Daily, 30 days kept. |

Terraform defines no S3 bucket and no message queue for Zammad (as of
2026-11-17). Background jobs are queued in PostgreSQL.

## AGM changes

Only commits written by AGM are listed. Branch compares against `develop` also
show upstream commits (for example `fix/report-download-query-separator` shows
24 commits, but 23 are upstream commits and one is AGM's). See
`DOCS-EVIDENCE.md` for the commands.

| Change | Why | Branch / PR / commit | Files touched | In production? | Upstream status |
|---|---|---|---|---|---|
| Desktop view editor: Highlight (multicolour) button and TextAlign menu (left, centre, right, justify) | The London Cab customer-service team asked for highlight and alignment in ticket replies. | `feature/editor-highlight-textalign`, commit `e841ae0333`; upstream PR zammad/zammad#6106 | `package.json`, `pnpm-lock.yaml`, `app/frontend/shared/components/Form/fields/FieldEditor/extensions.ts`, `app/frontend/apps/desktop/components/Form/fields/FieldEditor/useEditorActions.ts` | Yes. Production image is built from this branch, tip `fedf5211` (Terraform image tag, service catalog audit, version `7.1.x-fedf5211.docker` recorded 2026-08-23). | Declined: #6106 closed without merge on 2026-05-06 (a new upstream editor is being built). |
| Classic view editor bubble menu: text colour, highlight and alignment actions, drop-up colour palettes, icons that exist in the sprite, icons that follow the theme colour | Same request, for agents on the classic view. | `feature/editor-highlight-textalign`, commits `21c24f9c31`, `bbba735826`, `3b5edb8b80`, `a886c1f03f` (included in PR #6106) | `app/assets/javascripts/app/lib/base/jquery.contenteditable.js`, `app/assets/javascripts/app/lib/mixins/richtext_bubble_menu.coffee`, `app/assets/stylesheets/zammad.scss` | Yes (same evidence). | Declined (part of #6106). |
| Docker build: call `/opt/zammad/bin/rake assets:precompile` instead of `rake` | Without it the image is built with almost no frontend assets ("rake is empty"). | `feature/editor-highlight-textalign`, commit `05ebf2ea21` (included in PR #6106) | `Dockerfile` | Yes, at build time: the production image was built with it. | Declined (part of #6106). Not sent on its own. |
| Reporting "Download" URL uses `&` instead of `;` between query parameters | Rack 2.2+ no longer splits on `;`, so the download failed with HTTP 422. | `feature/editor-highlight-textalign`, commit `fedf5211b9`; `fix/report-download-query-separator`, commit `984f97ddab`; upstream PR zammad/zammad#6110 | `app/assets/javascripts/app/controllers/report.coffee` | Yes (`fedf5211b9` is the production branch tip). | Accepted: merged upstream as `ea91c4e58a` (2026-05-04), in 7.1.0 and later, not in 7.0.x. Drop the AGM commit when moving to 7.1.0 or later. |
| `SqlHelper` fix: values containing `?` in multi-select conditions no longer break triggers, overviews and Core Workflows (HTTP 422) | A contact-reason value with `?` blocked trigger and Core Workflow saves. | `fix/sql-helper-question-mark-bind`, commits `4aabdfb510`, `ccec8e601e`; upstream PR zammad/zammad#6090, issue zammad/zammad#6091 | `lib/sql_helper.rb`, `spec/lib/sql_helper_spec.rb` | No. Not on the production branch. Upstream's own fix `9b6bf63c66` (2026-04-22) is already in the production base. | Declined: #6090 closed without merge; upstream fixed #6091 its own way in `9b6bf63c66` (in 7.1.0 and later). The AGM branch can be deleted. |
| DevHub catalog entry and TechDocs | Register Zammad in DevHub. | `develop`, commit `bba7f22e46`, merged by AGM PR #1 (`71f782a24d`) | `catalog-info.yaml`, `mkdocs.yml`, `docs/` | No (documentation only, not in the image). | Not sent (AGM-only). |

**Branches that are not AGM changes.** The fork copied all upstream branches
when it was created on 2026-04-19:

- `copilot/add-ntfy-webhook-support`, `copilot/fix-bug-in-code-base`,
  `copilot/fix-saml-single-logout-service`, `copilot/fix-whatsapp-tab-title`.
  Their commits are by the GitHub Copilot bot, dated January to March 2026
  (before the fork existed). `copilot/add-ntfy-webhook-support` has the same
  commit as the upstream branch of that name.
- `interface`, `interface-v1`, `interface-v2`, `stable`, `stable-1.0` to
  `stable-6.5`, `stable-backport-*`. They hold no commits by AGM authors.
  Some differ from upstream only because upstream moved on.

These branches are not deployed. TODO(owner): delete the inherited branches
from the AGM fork so only AGM branches remain.

## Known gaps

### Changes outside the repo

These live in the production database or on the server, not in Git. They were
recorded in AGM's ops notes and could not be checked against the live system
for these docs (no live access was used).

| Change | Where | Status |
|---|---|---|
| Per-group required and hidden ticket fields: Core Workflow #13 (CD group) and #16 to #25 (one per London Cab group), from the "Zammad Workflows" requirements sheet | Admin UI → **Manage → Core Workflow** (applied with a local script over the REST API) | Applied 2026-04-19. TODO(owner): export and keep them in version control. |
| Custom ticket fields (for example `booking_ref`, `ride_type`, `cs_contact_reason`, C&D and driver fields). `booking_ref` and `ride_type` are required on every screen, so API edits fail with HTTP 422 if they are empty. | Admin UI → **Object Manager** | TODO(owner): document the field list and decide if the two required-everywhere fields should be required only per group. |
| Workaround: one `cs_contact_reason` tree value was renamed to drop a `?` | Object Manager attribute | Kept. Upstream fixed the bug in 7.1.0 and later, so the rename is no longer needed on the current build. |
| Triggers for routing and closing: #26 "Complaint Notifier", #27 "Bad Rating Routing", #31 "SIXT Auto Emails Closure", #38 "Email Automatic Assignment to LC Email Group", #45 (routes all inbound calls to the LC Phone group) | Admin UI → **Manage → Trigger** | #38 was changed on 2026-07-08 so #27 wins. A fix for #31 (auto-close only new tickets) was proposed and was on hold. TODO(owner): confirm the current state of #31 and #45. |
| Scheduler #3 "Remove Unresponsive Agent (2h)" removes the owner from every open ticket, every hour, with no time condition. Scheduler #2 "Auto-close Chat Tickets (1h)" closes open chat tickets with no age condition. | Admin UI → **Manage → Scheduler** | Found 2026-09-15. TODO(owner): has #3 been fixed or switched off? Fix it before any backlog clean-up. |
| **Agent notifications switched off on purpose**: setting `0100_notification` (id 234) emptied. The notification email channel (ch1) was disabled and replaced by a `sendmail` channel (ch2) that does not deliver. | Admin UI / `/api/v1/settings/234`, **Channels → Email** | Temporary since 2026-09-16, to unblock the job queue (the old SMTP host no longer exists). **Must be reverted** once notifications have a working Microsoft 365 sender. TODO(owner): current status, and move the revert steps (today in local ops notes) into `agm-infra-notes/runbooks/`. |
| Email channels: several inbound mailboxes were failing (Entra ID account disabled), and two old cPanel channels are disabled | **Channels → Email** | TODO(owner): current status of each channel. |
| Overviews #14 "LC Phone (Open & New)" and #15 "LC Emails (Open & New)" changed (sort order; a temporary filter on #14, removed on 2026-10-04) | **Manage → Overviews** | Done. |
| Old server only (historical): in-place edits of the compiled `application-*.js` inside the running container (report URL separators, editor icon names), an `.env` `IMAGE_REPO`/`VERSION` override and a compose override with `pull_policy: never` | The decommissioned on-premises server | Superseded: the same fixes are commits on the production branch, built into the image. Do not repeat in-place edits; they are lost when a container is recreated. |

### Other known gaps

- TODO(owner): confirm the production image tag and digest, and that it was
  built from `fedf5211` (see [Environments](deploy-operate.md#environments)).
- TODO(owner): confirm whether `zammad.agmegypt.com` is still used. Terraform
  and the service catalog use `helpdesk.agmegypt.com`, but the old name is
  still linked in `devhub-catalog/systems/zammad.yaml`,
  `agm-infra-notes/catalogs/zammad/catalog-info.yaml`, and on the bridge's
  default branch (`main`) in `3cx-zammad-bridge/catalog-info.yaml` and
  `3cx-zammad-bridge/docs/index.md` (the bridge's `docs/devhub` branch fixes
  its own copies). Update these when the answer is known.
- The compose file is at `/opt/zammad/docker-compose.yml` (Terraform writes it
  there, and the container names `zammad-zammad-*-1` match that directory).
  One Terraform comment names another path and the audit records drift (Z8),
  so check with `docker compose ls` before editing.
- Open security findings are tracked in the internal service catalog audit
  (`agm-service-catalog/docs/audits/zammad.md`), not here, because this
  repository is public. TODO(owner): close the open security findings listed
  there.
- TODO(owner): there is no build pipeline for the AGM image (Z5) and no dev
  environment (Z7).
- TODO(owner): Z1 (scheduler OOM) and Z2 (`TransactionJob` `NameError`) are
  open P1 findings with no alarm (Z4). Is Z2 related to the AGM build or to
  data?
- TODO(owner): the DevHub component `zammad` is also defined in
  `agm-service-catalog/services/zammad.yaml`. Two definitions of
  `component:default/zammad` will conflict in Backstage. Keep one.
