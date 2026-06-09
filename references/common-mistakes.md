# Common Mistakes

## Pre-Upgrade

| Mistake | Consequence | Fix |
|---|---|---|
| Not verifying the DB backup before starting | Backup is corrupt; can't roll back | Test restore to a spare DB before upgrading |
| Skipping release notes review | Miss breaking changes specific to target version | Always read Adobe's upgrade notes for the exact target version |
| Trusting UCT as the only compat check | Marketplace modules not scanned by UCT | Manually check each third-party module's compatibility matrix |
| Not disabling cron before upgrade | Cron job fires mid-migration, corrupts DB state | Always `cron:remove` before `maintenance:enable` |

## Composer / Version Bump

| Mistake | Consequence | Fix |
|---|---|---|
| Using `^` constraint for Magento edition | Accidentally upgrades past target version | Use exact version string `"2.4.8"` not `"^2.4"` |
| Forgetting to widen third-party constraints for major upgrade | Composer refuses to resolve | Check module `composer.json` and widen `magento/framework` constraint |
| Running `composer update` without `-W` on major upgrade | Transitive deps not updated; resolver fails | Use `composer update -W` for major upgrades |
| Using `composer update <package> -W` for a **downgrade** | "Nothing to modify in lock file" — Composer won't downgrade via update | Use `composer require magento/product-community-edition:<target> -W` instead; require forces version resolution |

## Build

| Mistake | Consequence | Fix |
|---|---|---|
| Skipping `setup:upgrade` | DB schema diverges from codebase | Always run after `composer update` |
| Skipping `setup:di:compile` | Fatal errors on first page load | Required after any code or config change |
| Not clearing `generated/` before `setup:di:compile` | Stale compiled proxies cause class-not-found errors | `rm -rf generated/code generated/metadata` before compile |
| Deploying static content without specifying locales | Deploys all locales — very slow, fills disk | Specify `--theme` and locales explicitly in production |
| Forgetting to flush Redis/Varnish after `cache:flush` | Old cached pages / config served after upgrade | Flush infrastructure caches separately from Magento cache |
| Running `setup:install --cleanup-database` with existing `env.php` | "WebsiteRepository: default website isn't defined" — maintenance:enable reads website config from partially wiped DB | Drop+recreate the DB **and** delete `app/etc/env.php` + `app/etc/config.php` before running `setup:install` |
| Relying on `--skip-db-validation` to bypass `SqlVersionProvider` | `--skip-db-validation` only skips the pre-flight check; `SqlVersionProvider` fires again during "Installing database schema" and still throws | Add unsupported version pattern to `app/etc/di.xml` via DI array merge (see build.md) |

## Docker / markshust Setup

| Mistake | Consequence | Fix |
|---|---|---|
| Running `mysqldump` on MariaDB 11.8+ container | "executable file not found in $PATH" | Use `mariadb-dump` and `mariadb` instead of `mysqldump` / `mysql` |
| Editing image tags manually in `compose.yaml` for an infra change | PHP/DB versions drift from the tested matrix | Run `bin/detect-versions <edition> <version>` — it regenerates `compose.versions.yaml` from `lib/versions.tsv` |
| Assuming `src/` is a bind mount into the container | Composer/setup changes inside the container don't appear in `src/` | `appdata:/var/www/html` is a **named Docker volume**; sync with `docker cp container:/var/www/html/file ./src/file` |
| Assuming versions.tsv PHP mapping matches official Magento system requirements | e.g. markshust maps 2.4.8 → PHP 8.4, though Magento officially lists 8.1–8.3 | Trust `lib/versions.tsv` and `bin/detect-versions` for the validated pairing; don't infer PHP from official docs alone |
| Running `composer require` from the host machine project root | "composer.json has been created" error — host has no Magento packages | Always run composer inside the container: `docker compose exec phpfpm composer require … -d /var/www/html` |

## Post-Upgrade

| Mistake | Consequence | Fix |
|---|---|---|
| Declaring upgrade done after HTTP 200 | Admin broken, checkout broken, JS errors present | Run full smoke test per `references/smoke-test.md` |
| Not re-enabling cron | Scheduled jobs (order emails, indexing, price rules) stop running | `bin/magento cron:install` after maintenance off |
| Leaving indexers in "Reindex required" state | Search, category pages, prices return stale/missing data | `bin/magento indexer:reindex` |
| Not capturing lessons | Same mistakes repeat on next upgrade | Always write to `tasks/lessons.md` before closing branch |
