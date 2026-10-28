# The just-agents fork

Upstream [Twenty](https://github.com/twentyhq/twenty) CRM, run as its own
Dokploy project on UTEG. just-agents' outreach agent works the CRM through
Twenty's built-in MCP server. The fork pins the release we run, carries any
patches we need, and publishes our own image. Today it adds deploy files only.

## What the fork adds

| Path | Purpose |
| --- | --- |
| `.github/workflows/ja-publish.yml` | Image build on `ja/*` tags |
| `packages/twenty-docker/just-agents/docker-compose.yml` | The Dokploy compose app: server, worker, Postgres, Redis, daily backups |
| `packages/twenty-docker/just-agents/.env.example` | Its environment |

## Branches

`just-agents` is an upstream release tag plus our commits, nothing else. Today
that is `twenty/v2.52.0`. To move to a new release, rebase onto its tag:

```bash
git fetch upstream --tags
git rebase --onto twenty/vX.Y.Z twenty/vOLD just-agents
git push --force-with-lease origin just-agents
```

Never `git push --tags` to the fork: upstream's `twenty/v*` and
`npm-release/*` tags trigger their deploy and npm release workflows.

## Releasing

Tag `just-agents` as `ja/<upstream version>-ja.<n>` and push the tag:

```bash
git tag ja/v2.52.0-ja.1 just-agents && git push origin ja/v2.52.0-ja.1
```

`ja-publish` builds the `twenty` target of `packages/twenty-docker/twenty/Dockerfile`
(server and frontend, linux/amd64) and pushes `ghcr.io/justanarthur/twenty:v2.52.0-ja.1`.
A new GHCR package starts private: make it public, or add GHCR as a registry
in Dokploy.

## Deploying on UTEG

1. DNS: an A record for the CRM domain (e.g. `crm.example.com`) to ryde.sk's IP.
2. Dokploy (dokploy.justadomainname.dev): **Create Project** `twenty` →
   **Create Service → Compose**. Source: GitHub `justAnArthur/twenty`, branch
   `just-agents`, compose path `./packages/twenty-docker/just-agents/docker-compose.yml`.
3. **Environment**: the variables in `.env.example`. Generate the encryption
   key with `openssl rand -base64 32` and keep a copy outside the server:
   losing it loses every secret stored in the database. The two passwords are
   `openssl rand -hex 32` (they go into connection URLs as is).
4. Leave Dokploy's **Domains** empty: the server's Traefik labels route
   `TWENTY_DOMAIN` with a Let's Encrypt certificate. **Deploy.**

The server runs the database migrations on start; the first boot takes a few
minutes. `https://<domain>/healthz` answers once it is up.

## First sign-up

The instance runs one workspace (`IS_MULTIWORKSPACE_ENABLED=false`). The
first account to sign up creates it, becomes admin, and sign-up closes. Sign
up yourself right after the first deploy, then check from a private window
that a second sign-up is refused. Add people later by invitation
(Settings → Members).

## The outreach agent's access

1. Settings → Members → Roles → **+ Create Role** `Outreach agent`.
   - All objects: See, Edit, Delete and Destroy **off**.
   - Object rules, **See + Edit** (no Delete, no Destroy): People, Companies,
     Opportunities, Notes, Tasks, Lists, List Members, and the note and task
     targets if the page lists them.
   - Settings and actions: none. Leave Send Email, Workflows, campaigns and
     API key settings off.
2. Settings → Developers → API → **+ Create key** `outreach-agent`. Copy it
   now; it is shown once.
3. Back on the role: **Assignment → Assign to API key** → `outreach-agent`. A
   key without a role gets default permissions, so do not skip this.

The agent connects to `https://<domain>/mcp` with
`Authorization: Bearer <key>`. No mailbox is connected, so nothing in Twenty
can send email.

## Backups and restore

The `backup` service writes `pg_dump -Fc` dumps to the `backups` volume as
`twenty-<UTC timestamp>.dump`: one when it starts, then one a day, keeping
the newest 14. Copy them off the server now and then (`docker cp`, or the
volume's path under `/var/lib/docker/volumes`).

Restore (container names are `<dokploy app name>-<service>-1`):

```bash
docker stop <app>-server-1 <app>-worker-1
docker exec <app>-backup-1 ls -1 /backups
docker exec <app>-backup-1 pg_restore --clean --if-exists --no-owner -d default /backups/twenty-<ts>.dump
docker start <app>-server-1 <app>-worker-1
```

A dump on demand: `docker restart <app>-backup-1`.

## Upgrading

1. Rebase `just-agents` onto the new upstream tag and release `ja/vX.Y.Z-ja.1`
   (above). Read upstream's upgrade guide for that version first.
2. Take a fresh dump: `docker restart <app>-backup-1`.
3. Set `TWENTY_TAG=vX.Y.Z-ja.1` in the Dokploy environment and **Deploy**. The
   server runs the upgrade migrations on start;
   `docker exec <app>-server-1 yarn command:prod upgrade:status` shows where
   they stand.

## Upstream workflows

GitHub Actions is not turned on for the fork yet (no workflows are
registered), so `ja-publish` does not run either. Turn it on from the
repository's **Actions** tab, then disable every upstream workflow, which
leaves `ja-publish`:

```bash
gh workflow list -R justAnArthur/twenty --all --json id,path \
  -q '.[] | select(.path != ".github/workflows/ja-publish.yml") | .id' |
  xargs -n1 gh workflow disable -R justAnArthur/twenty
```

None of them fire on pushes to `just-agents` or on `ja/*` tags. The ones that
would otherwise run:

- **Schedules on `main`**: `i18n-pull`, `docs-i18n-pull` and `website-i18n-pull`
  (every two hours: Crowdin pull, push a branch, open a PR),
  `ci-ai-catalog-sync` (daily) and `ci-dpa-subprocessors-sync` (weekly), both
  opening PRs, and `ci-server-integration-test-timings` (weekly).
- **PRs into `just-agents`**: about forty `pull_request` CI workflows, plus
  `docs-i18n-pull` (pushes translations to the PR branch), `ci-e2e-main` (QA
  scout with a Claude token), `app-prod-parity-e2e-dispatch`,
  `website-preview-dispatch`, and the `pull_request_target` ones:
  `preview-env-dispatch`, `pr-review-dispatch`,
  `external-contributor-pr-auto-draft`, `ci-utils` (Danger, with the repo
  token) and `ci-sensitive-paths-approval`.
- **Comments and reviews**: `claude`, `ci-blocked-contributors`,
  `pr-review-relay`, and through it `post-ci-comments` and
  `visual-regression-dispatch`.
- **Upstream tags**: `cd-deploy-tag` (`twenty/v*`), `publish-npm-packages`
  (`npm-release/*`), `cd-storybook-ui` (`sdk/v*`, `ui/v*`).

Without upstream's secrets most of them fail, but they spend Actions minutes,
and the ones that only need the repo token can open PRs on the fork.
