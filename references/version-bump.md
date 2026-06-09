# Version Bump Reference

## Magento Edition Metapackages

| Edition | Composer package |
|---|---|
| Community (open-source) | `magento/product-community-edition` |
| Enterprise / Adobe Commerce | `magento/product-enterprise-edition` |
| Adobe Commerce Cloud | managed via `magento/magento-cloud-metapackage` |

## Standard Version Bump

Edit `composer.json` — update the edition constraint:

```json
{
  "require": {
    "magento/product-community-edition": "2.4.8",
    "php": "~8.2.0||~8.3.0"
  }
}
```

For patch upgrades, use exact version (not `^` or `~`) to avoid accidental drift.

## Third-Party Module Constraint Widening

When upgrading a major version, third-party modules may need their `require` constraints widened:

```bash
# Find modules pinned to old Magento version
grep -r '"magento/framework"' vendor/ --include=composer.json | grep "2\.4\.[0-5]"
```

For modules you own/control:
- Update their `composer.json` to widen the `magento/framework` constraint

For Marketplace modules:
- Check if a compatible version exists and update the version constraint in root `composer.json`

## minimum-stability

For major upgrades Magento sometimes releases RC candidates. If needed:

```json
{
  "minimum-stability": "RC",
  "prefer-stable": true
}
```

Reset to `stable` after upgrade is complete.

## Downgrading (version rollback)

`composer update` will not downgrade — even with `-W`, it returns "Nothing to modify in lock file" if the lock already satisfies the constraint. Use `composer require` with an explicit version instead:

```bash
# Correct approach for a downgrade
docker compose exec <service> composer require \
  magento/product-community-edition:<target_version> \
  --no-interaction -W -d /var/www/html
```

Note: `composer require` on the metapackage prints a warning ("does not function properly … use 'require-commerce' instead") but then runs `composer update` internally and resolves correctly. The warning is safe to ignore for open-source CE.

After downgrading, the top-level `"version"` field in `composer.json` is **not** updated by `composer require` (it only touches the `require` section). Update it manually to match.

**After the composer step, run the full post-composer build sequence** — the DB schema, compiled DI, and caches still reflect the higher version until these complete:

```bash
docker compose exec <service> bin/magento setup:upgrade
docker compose exec <service> bin/magento setup:di:compile
docker compose exec <service> bin/magento setup:static-content:deploy -f en_US
docker compose exec <service> bin/magento cache:clean
docker compose exec <service> bin/magento cache:flush
```

Also clear `generated/` before `setup:di:compile` if downgrading across a major version — stale interceptors targeting the old version's class signatures will cause fatal errors:

```bash
docker compose exec <service> rm -rf /var/www/html/generated/code /var/www/html/generated/metadata
```

## Verifying After Composer Update

```bash
# Confirm installed version
bin/magento --version

# Confirm no dependency conflicts remain
composer check-platform-reqs
```
