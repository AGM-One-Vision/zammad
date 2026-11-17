---
owner: group:default/dto
last_reviewed: 2026-11-17
---

# Deploy & operate

Infrastructure is owned by DevOps (`group:default/devops` in the service
catalog). The Terraform lives in `AGM-One-Vision/agm-infrastructure`:
`environments/prod/zammad.tf` (instance, compose file, ALB, ECR),
`zammad-alerts.tf` (alarms) and `zammad-maintenance.tf` (nightly maintenance).
Runbooks live in `AGM-One-Vision/agm-infra-notes/runbooks/` (`zammad.md`,
`zammad-restore.md`, `zammad-architecture.md`).

## Environments

| Branch | Environment | Image | Host | URL |
|---|---|---|---|---|
| `feature/editor-highlight-textalign` (tip `fedf5211`) | Production | `zammad-agm:7.0.1-editor-extras-v3` in ECR repo `zammad-agm`, built and pushed by hand on 2026-06-30 | EC2 `zammad-prod` (`r6i.large`, eu-central-1, private subnet) | <https://helpdesk.agmegypt.com> |
| `develop` | None. Holds only the DevHub catalog and these docs on top of upstream. | – | – | – |
| – | Development / staging | Not applicable: there is no dev or staging environment. The service catalog lists this as finding Z7 ("development happens on prod"). | – | – |

Production has run on AWS since 2026-06-30. Before that it ran on an
on-premises server that is now decommissioned. Do not use any old notes that
point at that server.

TODO(owner): confirm that the running image is still
`zammad-agm:7.0.1-editor-extras-v3` built from `fedf5211`, and record its ECR
image digest here.

## CI/CD stages

There is **no pipeline** for the AGM image. The service catalog audit
(2026-11-09) scores it 0 of 9 CI/CD stages.

| Stage | Today |
|---|---|
| Build | By hand (`docker build`), on a machine of the engineer's choice. |
| Test | None run for the AGM image. |
| Image scan | None for the AGM image. |
| Push | By hand to ECR repo `zammad-agm`. |
| Deploy | By hand: edit the compose file on the host over SSM, then `docker compose pull` and `up -d`. |
| Verify / rollback | By hand. |

The workflows in `.github/workflows/` (`ci.yaml`, `docker-ci.yaml`,
`docker-release.yaml`, `packager.io.yaml`) come from upstream. They run in
this fork on upstream triggers. They do not build or push the AGM image. The
last scheduled `docker-release` run on `develop` (2026-07-07) failed.

## How to deploy

This is the manual procedure that matches how the current image was built.
Agree a change window first: Zammad is tier Critical, 24/7.

1. **Build the image** from the production branch. Use an immutable tag that
   includes the commit:

    ```sh
    git switch feature/editor-highlight-textalign
    git pull
    SHA=$(git rev-parse HEAD)
    docker build --build-arg COMMIT_SHA=$SHA -t zammad-agm:<version>-agm-${SHA:0:8} .
    ```

    The `Dockerfile` refuses to build without `COMMIT_SHA`, and appends the
    first 8 characters of it to `VERSION`.

2. **Check the frontend assets were built.** Without the AGM Dockerfile fix the
   build "succeeds" but ships almost no frontend files:

    ```sh
    docker run --rm --entrypoint sh zammad-agm:<tag> -c 'find /opt/zammad/public/assets/frontend -type f | wc -l'
    ```

    Expect hundreds of files (about 565 were seen for 7.0.1), not 3.

3. **Push** the image to the ECR repo `zammad-agm`. Get the repository URL
   from the Terraform output `zammad_ecr_url` (in `environments/prod`); the
   registry is the part before the first `/`:

    ```sh
    REPO=<value of terraform output zammad_ecr_url>
    aws ecr get-login-password --region eu-central-1 \
      | docker login --username AWS --password-stdin "${REPO%%/*}"
    docker tag zammad-agm:<tag> "$REPO:<tag>"
    docker push "$REPO:<tag>"
    ```

    TODO(owner): who is allowed to push to `zammad-agm`?
4. **Change the tag in the compose file on the host** (the six `zammad-*`
   services). Also record the new tag in Terraform
   (`environments/prod/zammad.tf`, the compose block inside
   `aws_instance.zammad` `user_data`) through a DevOps pull request, so a
   rebuilt instance gets the same image. That resource ignores `user_data`
   changes (`lifecycle { ignore_changes = [ami, user_data] }`), so a
   `terraform apply` does **not** change the running host. Do not remove that
   setting without DevOps: the database lives on the instance's root volume.
5. **Roll out** over SSM Session Manager on the instance. The compose file is
   `/opt/zammad/docker-compose.yml`. Run `docker compose ls` first and check
   that the project `zammad` points there (the audit records Terraform drift,
   Z8):

    ```sh
    docker compose ls
    cd /opt/zammad
    docker compose pull
    docker compose up -d
    ```

    `zammad-init` runs database migrations on start.
6. **Re-apply the 4 GB memory caps.** `docker compose up -d` recreates the
   containers without the per-container limits that the nightly maintenance
   window sets. Re-apply them at once (or run the maintenance window's task on
   demand):

    ```sh
    docker update --memory 4g --memory-swap 4g \
      zammad-zammad-railsserver-1 zammad-zammad-scheduler-1 zammad-zammad-websocket-1
    ```

7. **Verify:** log in at <https://helpdesk.agmegypt.com>. Check that the
   editor buttons are there and that a Reporting download works. Check that
   the ALB target group is healthy and that no alarm fires.

The compose path comes from Terraform, which writes
`/opt/zammad/docker-compose.yml`; the container names `zammad-zammad-*-1`
used by the maintenance window match a compose project in that directory. One
comment in `zammad-maintenance.tf` names
`/home/ubuntu/zammad-docker-compose/`, so always confirm with
`docker compose ls`.

## Rollback

- **Image only (no data change):** set the previous image tag back in
  `/opt/zammad/docker-compose.yml` (confirm with `docker compose ls`), run
  `docker compose up -d`, then re-apply the memory caps:

    ```sh
    docker update --memory 4g --memory-swap 4g \
      zammad-zammad-railsserver-1 zammad-zammad-scheduler-1 zammad-zammad-websocket-1
    ```

    Keep the previous image in ECR until the new one has run cleanly. This is
    safe only if the new version did not run database migrations that the old
    code cannot read.
- **After migrations, or data damage:** restore from backup.
    - AWS Backup takes a daily AMI of the instance (plan
      `agm-connect-prod-zammad-daily`, kept 30 days). The accepted data-loss
      window (RPO) is 24 hours. Follow
      `agm-infra-notes/runbooks/zammad-restore.md` (about 30 to 45 minutes).
    - The `zammad-backup` container also writes a daily `pg_dump` at 03:00
      Cairo time into the `zammad-backup` volume, kept 10 days.
- Before a risky change, start an on-demand AWS Backup job so the restore point
  is fresh.

## When an alarm fires

All alarms go to SNS `zammad-prod-alerts`, then through the shared
`agm-cloudwatch-teams` forwarder to Microsoft Teams. The CloudWatch dashboard
is `Zammad-Health`. Each alarm's description links a section in
`agm-infra-notes/runbooks/zammad.md`.

The table lists the 9 alarms defined in Terraform (`zammad-alerts.tf`). The
2026-11-09 audit counts 14 host alarms: some exist outside Terraform
(drift, see the internal audit, Z10).

| Alarm | Means | First action |
|---|---|---|
| `zammad-prod-alb-unhealthy` | The ALB has no healthy Zammad target. The helpdesk is down. | SSM into the instance, run `docker ps` and check `zammad-nginx` and `zammad-railsserver`. Restart failed containers. |
| `zammad-prod-ec2-impaired` | The instance status check failed (OS hang, out of memory). | The alarm reboots the instance automatically. Check that it came back, then look for the cause (memory). |
| `zammad-prod-mem-warn` (80%, 15 min) / `zammad-prod-mem-critical` (90%) | Memory pressure. A hang can follow within minutes. | Run `docker stats` to find the container using the most memory and restart it. At critical, reboot before SSM stops responding. |
| `zammad-prod-disk-warn` (85%) / `zammad-prod-disk-critical` (92%) | Root disk filling up, often Docker JSON logs or dumps. | Truncate the largest `/var/lib/docker/containers/*/*-json.log` files. Check the `zammad-backup` volume size. |
| `zammad-prod-alb-5xx-rate` | Users get server errors. | Read the Rails and nginx logs. Often database connection saturation or an Elasticsearch timeout. |
| `zammad-prod-alb-latency-p95` (> 2 s, 15 min) | Users feel slowness. | Correlate with the memory alarms. Check Elasticsearch and slow database queries. |
| `zammad-prod-ebs-io-saturation` | Disk I/O is backing up (gp3 queue length). | Check `docker system df` and `iostat`. Consider more IOPS. |

**Nightly maintenance:** the SSM maintenance window `zammad-prod-ruby-recycle`
runs at 03:00 Africa/Cairo every day. It sets a 4 GB memory limit on the
`railsserver`, `scheduler` and `websocket` containers, then restarts them to
free Ruby memory. Expect under 30 seconds of slowness. Its logs go to the
CloudWatch log group `/ssm/zammad-prod/maintenance`.

**Failures no alarm sees** (service catalog audit, 2026-11-09):

- Z1: the scheduler container was killed for using too much memory (OOM) 61
  times in 17 days, at its 4 GB cap.
- Z2: `TransactionJob` fails permanently about 85 times an hour
  (`NameError: wrong constant name`), so triggers and notifications are
  skipped.
- Z4: there is no alarm on container OOM or on failed jobs.

If agents report that "tickets do not trigger" or "nobody got notified", check
these first. Also check [Known gaps](reference.md#known-gaps): agent
notifications are switched off on purpose at the moment.

## Upgrading

Production is built from an upstream `develop` snapshot plus AGM commits. It
is not built from an upstream release. To take a new upstream version without
losing the AGM changes:

1. **Pick a target.** Prefer an upstream release tag (for example `7.2.3`) over
   `develop`. Read `BREAKING_CHANGES.md` and `CHANGELOG.md` for the range.
2. **Create a new branch from the tag** and keep the old branch for rollback:

    ```sh
    git fetch https://github.com/zammad/zammad.git tag <tag> --no-tags
    git switch -c agm/<tag>-editor-extras <tag>
    ```

3. **Re-apply only the AGM commits that are still needed.** Cherry-pick them in
   this order:
    - `e841ae0333`: desktop editor (Highlight and TextAlign; `package.json`,
      `pnpm-lock.yaml`, `extensions.ts`, `useEditorActions.ts`);
    - `21c24f9c31`, `bbba735826`, `3b5edb8b80`, `a886c1f03f`: classic editor
      bubble menu;
    - `05ebf2ea21`: Dockerfile rake binstub, if the target still uses
      `bundle exec rake assets:precompile`.

    **Do not** re-apply `fedf5211b9` (report URL). Upstream merged the same fix
    as `ea91c4e58a`, which is in 7.1.0 and later.
4. **Known traps:**
    - `pnpm-lock.yaml` will conflict. Re-add the two tiptap packages and run
      `pnpm install` to regenerate the lockfile; do not hand-merge it.
    - AGM pins `@tiptap/extension-highlight` and `@tiptap/extension-text-align`
      at `3.19.0`, while the other tiptap packages are at `3.22.5`. Align
      them to the target's tiptap version.
    - Upstream declined the editor change (PR #6106) because a new editor is
      coming. When upstream replaces the editor, these commits will not apply.
      Ask the business whether the new editor covers the need.
    - Moving from the current `7.1.x` develop snapshot to a release tag can
      run database migrations. Take an on-demand backup first.
5. **Build, check and deploy** as in [How to deploy](#how-to-deploy), with a
   new immutable tag.
6. **Test** login, reply with the new toolbar buttons (desktop and classic
   views), Reporting download, email fetching, a phone call through the 3CX
   bridge and search.
7. **Update these docs** and the AGM changes table.

TODO(owner): decide whether `develop` in this fork should be synced with
upstream, or whether the production branch should become the default branch so
DevHub shows the code that runs.
