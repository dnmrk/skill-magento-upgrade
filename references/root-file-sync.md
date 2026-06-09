# Root File Sync (magento2-base Conflict Resolution)

## What is magento2-base?

`magento/magento2-base` is a Magento metapackage that manages root-level files. After `composer update`, it may overwrite or conflict with your customised root files.

## Files Managed by magento2-base

Common files that may conflict after upgrade:

```
.htaccess
nginx.conf.sample
pub/.htaccess
pub/static/.htaccess
pub/media/.htaccess
app/etc/config.php        (module enable/disable list — review carefully)
app/design/               (base theme assets)
lib/                      (JS libraries)
setup/                    (install wizard)
```

## Conflict Resolution Strategy

```bash
# See all changed/conflicting files after composer update
git status --short
git diff HEAD -- .htaccess nginx.conf.sample
```

**Decision per file:**

| File | Default action |
|---|---|
| `.htaccess` / `nginx.conf.sample` | Accept upstream (Magento's version), re-apply local server config changes manually |
| `app/etc/config.php` | **Review carefully** — new modules may have been added; don't blindly reset |
| `pub/.htaccess` | Accept upstream unless you have specific media/static serving customisations |
| `composer.json` (root) | Keep your version — you set the version constraints |

## app/etc/config.php Special Handling

This file tracks enabled/disabled modules. After upgrade it may:
- Add new Magento modules (newly introduced in the target version)
- Change the enabled state of existing modules

```bash
# Review the diff carefully
git diff HEAD -- app/etc/config.php

# After resolving, always run setup:upgrade to sync with DB
bin/magento setup:upgrade
```

## Committing Root File Changes

Commit root file resolutions separately from composer.lock changes for clean history:

```bash
git add .htaccess pub/.htaccess app/etc/config.php
git commit -m "upgrade/<version>: resolve magento2-base root file conflicts"
```
