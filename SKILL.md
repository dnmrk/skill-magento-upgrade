---
name: magento-upgrade
description: Use when upgrading a Magento 2 / Adobe Commerce installation to a newer version — patch, minor, or major. Triggers: target version specified, composer.json constraint needs updating, extension or module compatibility unknown, maintenance window required, UCT warnings need interpretation, deprecated or removed APIs found, version bump blocked by constraint conflicts, or DB migration required after version change.
allowed-tools:
  - Read
  - Glob
  - Grep
---

# Magento 2 / Adobe Commerce Upgrade

## Overview

Step-by-step Magento 2 version upgrade. Track every phase with the host agent's visible task tool when available (Claude Code: `TaskCreate`). Detect Docker first — every command runs inside the container if found.

## Hard Constraints

- **Never skip Phase 5 (UCT / compatibility check)** — modules break silently without it.
- **Every command runs inside the container** — detect Docker at Step 0 before touching anything else.
- **Never skip Phase 13 (setup:upgrade)** — this is the single most critical post-composer command; without it the DB schema doesn't match the codebase.
- **Never skip Phase 14 (setup:di:compile)** — missing DI compilation produces fatal errors on first page load.
- **Disable cron before maintenance on (Phase 7)** — a cron firing mid-migration can corrupt the DB state.
- **Never proceed to Phase 19 (deployment checklist) until Phase 18 (smoke test) passes** — HTTP 200 alone is not sufficient.
- **Never skip Phase 20 (lessons)** — every upgrade produces learnings; capture them before closing the branch.
- **"Flex recipes" does not exist in Magento** — the equivalent is root file conflict resolution (Phase 11).

## Signal → Reference Routing

| Task signal | Always read | Also read if |
|---|---|---|
| UCT, upgrade:check, module warnings, "Not compatible" | `references/compatibility-check.md` | `references/version-bump.md` if major upgrade |
| composer.json, version constraint, metapackage, constraint conflict | `references/version-bump.md` | `references/compatibility-check.md` if warnings ambiguous |
| setup:upgrade, setup:di:compile, static-content:deploy, generated/ | `references/build.md` | `references/version-bump.md` if major upgrade broke APIs |
| root files, magento2-base, .htaccess, nginx.conf, app/etc conflicts | `references/root-file-sync.md` | — |
| smoke test, 5xx, blank page, missing assets, admin login | `references/smoke-test.md` | `references/deployment-checklist.md` once passing |
| deployment checklist, QA notes | `references/deployment-checklist.md` | — |

## Mode-Specific Behavior

| Mode | Do |
|---|---|
| **Planning / scoping** | Detect version from `composer.json`, classify upgrade type (patch/minor/major), run infrastructure gate if major, surface unknowns before starting phases |
| **Executing upgrade** | Follow phases in order, read the phase reference on entry, mark plan/task entries complete as you go, stop at blockers rather than pushing through |
| **Post-upgrade review** | Pin baseline version, walk full Phase 18 smoke test, tailor QA Notes to what actually changed |

## Cross-Cutting Invariants

- Read the current version from `composer.json` and `bin/magento --version` — **never assume it**.
- Major upgrades require the **infrastructure gate** (Phase 8) before the version bump.
- UCT evaluates against your target version — run it with `--coming-version` set explicitly.
- Third-party modules from Marketplace **always require manual compat check** — UCT only scans your custom code.
- `setup:static-content:deploy` must specify locales and themes explicitly in production to avoid deploying unnecessary assets.
- HTTP 200 on `/admin` is **not a passing smoke test** — confirm admin login and check for JS errors.

---

## Step 0: Environment Detection

```bash
ls docker-compose.yml docker-compose.yaml compose.yml compose.yaml 2>/dev/null
```

If any file exists, determine the PHP service name (`app`, `php`, `web`, `magento`) and prefix **every** subsequent command:

```bash
docker compose exec <service> <command>
```

Also detect the Magento root path (often `/var/www/html` or `/app`).

---

## Upgrade Phases

Create visible progress tracking before starting. In Claude Code, use `TaskCreate`. Mark each phase complete as you finish it.

**Required phases — create a task for every one, no exceptions:**
Phase 1 (Backup), Phase 2 (Branch), Phase 3 (Release notes review), Phase 4 (Vendor sync), Phase 5 (UCT / compatibility check), Phase 6 (Rollback plan), Phase 7 (Cron disable + Maintenance on), Phase 9 (Version bump), Phase 10 (Composer update), Phase 11 (Root file sync), Phase 12 (Commit), Phase 13 (setup:upgrade), Phase 14 (DI compile), Phase 15 (Static content deploy), Phase 16 (Cache clean), Phase 17 (Reindex + Cron re-enable + Maintenance off), Phase 18 (Smoke test), Phase 19 (Deployment checklist), Phase 20 (Lessons).

Phase 8 (Infrastructure Gate) is conditional — required for major upgrades only.

---

### Phase 1 — Backup

```bash
# Database
docker compose exec <db-service> mysqldump -u<user> -p<pass> <db> | gzip > tasks/<target_version>/<current_version>_dump.sql.gz

# Verify the dump is non-empty
ls -lh tasks/<target_version>/
```

Also confirm file backups are in place (media, var, pub — or that these are excluded per deployment strategy).

---

### Phase 2 — Branch

Check for an existing upgrade branch first — the work may already exist:

```bash
git branch -a | grep -i <ticket-or-version>
```

If a matching branch exists: surface it and ask whether to continue or create new. Do **not** silently create a duplicate.

```bash
git checkout -b upgrade/<target_version>
```

---

### Phase 3 — Review Release Notes

Before touching any code, read the official upgrade notes for the target version:

- Adobe Commerce / Magento 2 release notes: https://experienceleague.adobe.com/en/docs/commerce-operations/release/notes/overview
- Upgrade guide for the target version: https://experienceleague.adobe.com/en/docs/commerce-operations/upgrade-guide/overview

Flag any breaking changes relevant to the project (removed classes, renamed config keys, changed DB schema, new system requirements).

---

### Phase 4 — Vendor Sync

```bash
composer install
```

---

### Phase 5 — UCT / Compatibility Check

```bash
# Run Adobe's Upgrade Compatibility Tool against the target version
docker compose exec <service> vendor/bin/upgrade-compatibility-tool upgrade:check \
  --coming-version=<target_version> /app

# If UCT is not installed:
composer require magento/upgrade-compatibility-tool --dev
```

→ **Read `references/compatibility-check.md`** for output interpretation, severity levels, and third-party module handling before acting on any results.

Also manually audit:
- Third-party Marketplace modules against their compatibility matrix
- Custom modules for deprecated API usage

---

### Phase 6 — Rollback Plan

Document the rollback procedure **before** any code or DB changes are made. If something goes wrong post-deployment, this must be ready to execute without thinking.

Capture in `tasks/<target_version>/rollback-plan.md`:
- DB restore command with backup file path
- Git branch to revert to (`git checkout <previous-branch>`)
- Full command sequence to restore: `composer install` → `setup:upgrade` → `setup:di:compile` → `setup:static-content:deploy`
- Who to notify and how
- Estimated rollback time

---

### Phase 7 — Cron Disable + Maintenance On

**Disable cron first** — a cron job firing mid-migration can leave the DB in a partial state:

```bash
# Disable Magento cron
docker compose exec <service> bin/magento cron:remove

# Enable maintenance mode
docker compose exec <service> bin/magento maintenance:enable
```

---

### Phase 8 — Infrastructure Gate (major upgrades only)

If upgrading a **major** version, verify manually against the release notes:

- PHP version requirement for the target release (`php -v`)
- MySQL / MariaDB version requirement
- Elasticsearch / OpenSearch version requirement
- Redis version requirement
- Composer version (`composer --version`)

This is a **manual decision gate** — confirm with the user before proceeding if infrastructure changes are needed.

---

### Phase 9 — Version Bump

Edit `composer.json`: update `magento/product-community-edition` or `magento/product-enterprise-edition` (and any pinned `magento/*` packages) to the target constraint.

→ **Read `references/version-bump.md`** for full details: metapackage patterns, third-party constraint widening for major upgrades, and handling `minimum-stability`.

---

### Phase 10 — Composer Update

```bash
docker compose exec <service> composer update --no-interaction
```

For major upgrades, run with `-W` to allow transitive dependency updates:

```bash
docker compose exec <service> composer update --no-interaction -W
```

---

### Phase 11 — Root File Sync (conflict resolution)

After `composer update`, `magento/magento2-base` may have updated root-level files. Review and resolve any conflicts:

```bash
git status --short | grep -E "\.(htaccess|conf|xml|json|lock)$"
```

→ **Read `references/root-file-sync.md`** for the files `magento2-base` manages, conflict resolution strategy, and which changes to keep vs. accept from upstream.

---

### Phase 12 — Commit

Commit `composer.json`, `composer.lock`, and any resolved root file changes before running the build commands. The build outputs (`generated/`, `pub/static/`) are gitignored and should not be committed.

```bash
git add composer.json composer.lock
git add .htaccess pub/.htaccess app/etc/config.php  # add any resolved root file conflicts
git commit -m "upgrade/<target_version>: bump Magento to <target_version>"
```

---

### Phase 13 — setup:upgrade

**Critical — do not skip.** Runs all pending DB schema and data patch scripts:

```bash
docker compose exec <service> bin/magento setup:upgrade
```

Watch for errors. Any failure here must be resolved before proceeding.

---

### Phase 14 — DI Compile

```bash
docker compose exec <service> bin/magento setup:di:compile
```

Compilation errors here reveal type mismatches, missing plugins, and removed classes in custom/third-party modules. Resolve before proceeding.

---

### Phase 15 — Static Content Deploy

```bash
# Deploy for all configured locales and themes
docker compose exec <service> bin/magento setup:static-content:deploy -f \
  en_US <other-locales>
```

For production, omit `-f` and specify only needed locales and themes to avoid deploying unused assets:

```bash
docker compose exec <service> bin/magento setup:static-content:deploy \
  --theme Magento/luma --theme <custom-theme> en_US <locale>
```

---

### Phase 16 — Cache Clean

```bash
docker compose exec <service> bin/magento cache:clean
docker compose exec <service> bin/magento cache:flush
```

Also flush infrastructure caches if present:
- Redis: `redis-cli FLUSHALL`
- Varnish: `varnishadm ban req.url ~ .`
- CDN: purge at the CDN level if applicable

---

### Phase 17 — Reindex + Cron Re-enable + Maintenance Off

```bash
# Reindex all indexers
docker compose exec <service> bin/magento indexer:reindex

# Re-enable cron
docker compose exec <service> bin/magento cron:install

# Disable maintenance mode
docker compose exec <service> bin/magento maintenance:disable
```

---

### Phase 18 — Smoke Test

Verify the site is operational. **Do not proceed to Phase 19 until this passes.**

→ **Read `references/smoke-test.md`** for the full verification checklist: storefront load, admin login, checkout funnel, payment method display, search, JS console errors, log triage.

---

### Phase 19 — Deployment Checklist

→ **Read `references/deployment-checklist.md`** for the full template. Write output to `tasks/<target_version>/deployment-checklist.md`. Tailor QA Notes to what actually changed.

---

### Phase 20 — Compile Lessons

After every upgrade, close the loop. This phase must always run — even if the upgrade was routine.

Capture to `tasks/lessons.md`:
- Any modules that broke and how they were fixed
- Any unexpected schema changes
- Infrastructure surprises
- Time estimates vs. actual
- Anything worth flagging for the next upgrade

---

## Reference Map

- [`references/compatibility-check.md`](references/compatibility-check.md) — UCT output interpretation, severity levels, third-party module handling
- [`references/version-bump.md`](references/version-bump.md) — metapackage patterns, constraint widening, minimum-stability
- [`references/build.md`](references/build.md) — setup:upgrade failure handling, DI compile errors, static content deploy flags
- [`references/root-file-sync.md`](references/root-file-sync.md) — magento2-base managed files, conflict resolution strategy
- [`references/smoke-test.md`](references/smoke-test.md) — storefront/admin verification, JS errors, log triage
- [`references/deployment-checklist.md`](references/deployment-checklist.md) — full checklist template, QA Notes rules
- [`references/common-mistakes.md`](references/common-mistakes.md) — full common mistakes reference

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Running commands on host when Docker is present | Check for compose file at Step 0 — every command runs inside the container |
| Skipping `setup:upgrade` after `composer update` | Without it the DB schema diverges from the codebase — always run it |
| Skipping `setup:di:compile` | Produces fatal errors on first page load — required after any code change |
| Skipping cron disable before maintenance | Cron firing mid-migration can corrupt partial DB state |
| Treating UCT output as definitive for Marketplace modules | UCT only scans custom code paths — Marketplace modules need manual compat check |
| Deploying static content without specifying locales in production | Deploys ALL locales — slow and fills disk; always specify locales explicitly in prod |
| Forgetting to flush Redis/Varnish after `cache:flush` | Magento's cache:flush only clears Magento's registered cache backends — infrastructure caches need separate flushing |
| Not verifying the DB backup before starting | An unverified backup is no backup — check file size and run a test restore on a spare DB |
| Calling it "Flex recipes" | Magento doesn't use Symfony Flex; the step is root file conflict resolution from `magento2-base` updates |
