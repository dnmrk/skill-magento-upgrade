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

## Verifying After Composer Update

```bash
# Confirm installed version
bin/magento --version

# Confirm no dependency conflicts remain
composer check-platform-reqs
```
