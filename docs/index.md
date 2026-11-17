---
owner: group:default/dto
last_reviewed: 2026-11-17
---

# Zammad Helpdesk (AGM)

## What it does

Zammad is AGM's customer helpdesk. Customer requests that arrive by email, by
phone and through the web become tickets that agents can track, assign and
answer.

This repository is AGM's fork of the open-source
[zammad/zammad](https://github.com/zammad/zammad) project. **It is not a vanilla
mirror.** AGM carries its own changes on top of upstream:

- extra formatting actions (highlight, text alignment, text colour) in both
  ticket editors;
- a fix to the Docker build so the frontend assets are compiled;
- a fix to the "Download" link in the Reporting view.

Production runs a hand-built image (`zammad-agm`) built from the AGM branch
`feature/editor-highlight-textalign`. That branch is an upstream `develop`
snapshot of 2026-04-29 (version `7.1.x`) plus the AGM commits. The default
branch `develop` is **not** what runs in production, and it is not kept in sync
with upstream (as of 2026-11-17 it is about 1,860 commits behind upstream
`develop`).

The full list of changes, with branches, commits and upstream status, is in
[Reference → AGM changes](reference.md#agm-changes).

Phone tickets come from a separate AGM service,
[3cx-zammad-bridge](https://github.com/AGM-One-Vision/3cx-zammad-bridge), which
runs on the same server. See [Architecture](architecture.md).

## Who uses it

- Customer-service agents across AGM. London Cab is the main user.
- Customers, indirectly: by email and phone, which become tickets.
- The 3cx-zammad-bridge service, which calls the Zammad API to create phone
  tickets.

TODO(owner): list any other systems that call the Zammad API (the service
catalog audit could not see other callers).

## Business owner

London Cab (`group:default/london-cab`), as recorded in the AGM service catalog
(`agm-service-catalog/services/zammad.yaml`). The catalog flags this as open:
Zammad serves the whole company, but London Cab is the main user.

TODO(owner): confirm whether the business owner is London Cab or a company-wide
owner.

## PM

None. The service catalog records "PM: none".

## Links

| What | Link |
|---|---|
| Production (only environment) | <https://helpdesk.agmegypt.com> |
| Development / staging | Not applicable: there is no dev or staging environment (see [Deploy & operate](deploy-operate.md#environments)). |
| Source (AGM fork) | <https://github.com/AGM-One-Vision/zammad> |
| Branch that production is built from | [`feature/editor-highlight-textalign`](https://github.com/AGM-One-Vision/zammad/tree/feature/editor-highlight-textalign) |
| Upstream project | <https://github.com/zammad/zammad> |
| DevHub component | <https://devhub.agmegypt.com/catalog/default/component/zammad> |
| DevHub system | <https://devhub.agmegypt.com/catalog/default/system/zammad> |
| Service catalog audit (2026-11-09) | <https://github.com/AGM-One-Vision/agm-service-catalog/blob/main/docs/audits/zammad.md> |
| Phone bridge | <https://github.com/AGM-One-Vision/3cx-zammad-bridge> |
| Upstream admin and user manual | <https://docs.zammad.org> |
| Upstream REST API reference | <https://docs.zammad.org/en/latest/api/intro.html> |

The earlier version of `catalog-info.yaml` linked `https://zammad.agmegypt.com`.
Terraform (the load balancer host rule and `ZAMMAD_FQDN`) and the service
catalog use `helpdesk.agmegypt.com`. The old name is still linked in
`devhub-catalog/systems/zammad.yaml`,
`agm-infra-notes/catalogs/zammad/catalog-info.yaml`, and in
`3cx-zammad-bridge` (`catalog-info.yaml` and `docs/index.md` on `main`).

TODO(owner): is `zammad.agmegypt.com` an alias of the helpdesk, or a stale name
that can be removed from those files?
