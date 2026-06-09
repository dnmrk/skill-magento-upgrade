# Deployment Checklist

Write completed checklist to `tasks/<target_version>/deployment-checklist.md`.

## Template

```markdown
# Deployment Checklist — Magento <current> → <target>

**Date:** <date>
**Performed by:** <name>
**Environment:** <staging / production>

## Pre-Upgrade
- [ ] Database backup taken and verified (size: ___, location: ___)
- [ ] File backup confirmed (media, var, or noted as excluded)
- [ ] Upgrade branch created: `<branch-name>`
- [ ] Release notes reviewed for <target-version>
- [ ] UCT run — critical issues: ___ / errors: ___ / warnings: ___
- [ ] Third-party module compatibility confirmed: <list modules checked>
- [ ] Rollback plan documented: <link or summary>
- [ ] Infrastructure requirements met (PHP: ___, MySQL: ___, ES/OS: ___)

## Upgrade Execution
- [ ] Cron disabled
- [ ] Maintenance mode enabled
- [ ] `composer update` completed without errors
- [ ] Root file conflicts resolved
- [ ] `setup:upgrade` completed without errors
- [ ] `setup:di:compile` completed without errors
- [ ] `setup:static-content:deploy` completed
- [ ] `cache:clean` + `cache:flush` run
- [ ] Redis / Varnish / CDN cache flushed
- [ ] Indexers reindexed — all show Ready
- [ ] Cron re-enabled
- [ ] Maintenance mode disabled

## Smoke Test Results
- [ ] Storefront loads (HTTP ___)
- [ ] Category page loads with products
- [ ] Add to cart works
- [ ] Checkout page loads
- [ ] Admin login successful
- [ ] No critical errors in exception.log
- [ ] Search returning results
- [ ] Cron status healthy

## QA Notes

<Tailor to what changed in this upgrade: modules bumped, APIs fixed, config changes, known issues found and resolved>

## Rollback Plan

<If something goes wrong post-deployment:>
1. Enable maintenance: `bin/magento maintenance:enable`
2. Restore DB: `mysql -u<user> -p<pass> <db> < <backup>.sql`
3. Checkout previous branch: `git checkout <previous-branch>`
4. Run `composer install` to restore vendor
5. Run `setup:upgrade`, `setup:di:compile`, `setup:static-content:deploy`
6. Disable maintenance: `bin/magento maintenance:disable`
```
