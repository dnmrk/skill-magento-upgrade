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

## Build

| Mistake | Consequence | Fix |
|---|---|---|
| Skipping `setup:upgrade` | DB schema diverges from codebase | Always run after `composer update` |
| Skipping `setup:di:compile` | Fatal errors on first page load | Required after any code or config change |
| Not clearing `generated/` before `setup:di:compile` | Stale compiled proxies cause class-not-found errors | `rm -rf generated/code generated/metadata` before compile |
| Deploying static content without specifying locales | Deploys all locales — very slow, fills disk | Specify `--theme` and locales explicitly in production |
| Forgetting to flush Redis/Varnish after `cache:flush` | Old cached pages / config served after upgrade | Flush infrastructure caches separately from Magento cache |

## Post-Upgrade

| Mistake | Consequence | Fix |
|---|---|---|
| Declaring upgrade done after HTTP 200 | Admin broken, checkout broken, JS errors present | Run full smoke test per `references/smoke-test.md` |
| Not re-enabling cron | Scheduled jobs (order emails, indexing, price rules) stop running | `bin/magento cron:install` after maintenance off |
| Leaving indexers in "Reindex required" state | Search, category pages, prices return stale/missing data | `bin/magento indexer:reindex` |
| Not capturing lessons | Same mistakes repeat on next upgrade | Always write to `tasks/lessons.md` before closing branch |
