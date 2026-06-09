# Build Reference

## Post-Composer Command Sequence

Always run in this exact order — sequence matters:

```bash
# 1. Run DB schema and data patches
bin/magento setup:upgrade

# 2. Compile DI configuration
bin/magento setup:di:compile

# 3. Deploy static content
bin/magento setup:static-content:deploy -f en_US <other-locales>

# 4. Clean caches
bin/magento cache:clean
bin/magento cache:flush
```

## setup:upgrade

Runs all pending `Schema\*` and `Data\*` patch classes plus legacy InstallSchema/UpgradeSchema scripts.

**Common failure causes:**
- Missing DB table from a custom module's schema — check the migration class
- Foreign key constraint violation — data patch running before schema patch
- Duplicate entry error — data patch not idempotent; check `patch_list` table

> **Post-upgrade "Page Cache invalidated" banner:** After `setup:upgrade` completes, the admin shows a yellow banner flagging `full_page` as invalidated. This is expected — `setup:upgrade` marks it stale internally even after a prior `cache:flush`. Run `bin/magento cache:clean full_page` in Phase 16 to clear it. Not an indicator of a failed upgrade.

```bash
# Check which patches have run
SELECT patch_name, is_applied FROM patch_list ORDER BY patch_name;
```

## setup:di:compile

Compiles interceptors, factories, proxies, and the DI configuration.

**Common failure causes:**
- Class not found — removed upstream class used in a plugin/preference; needs removal or replacement
- Type mismatch in constructor — interface type hint removed or changed in target version; update the constructor
- Missing `generated/` cleanup — stale generated classes conflicting with new code

```bash
# Clean generated code before compile if hitting stale class errors
rm -rf generated/code generated/metadata
bin/magento setup:di:compile
```

## setup:static-content:deploy

**Production (specify explicitly):**
```bash
bin/magento setup:static-content:deploy \
  --theme Magento/blank \
  --theme Magento/luma \
  --theme <Vendor>/<custom-theme> \
  en_US en_GB <other-locales>
```

**Development (force deploy all):**
```bash
bin/magento setup:static-content:deploy -f
```

**Common failure causes:**
- Missing locale pack — install via `bin/magento i18n:pack`
- Theme compilation error — usually a LESS/SCSS variable removed in the new version; check theme inheritance chain
- Symlink mode not matching deploy mode — run `bin/magento deploy:mode:show` to confirm

## setup:install (clean reinstall / downgrade)

For a clean install — or any time you need to reinstall rather than upgrade — the DB and generated config files must both be clean **before** running `setup:install`. If `app/etc/env.php` exists when `setup:install` starts, Magento reads it, connects to the (partially wiped) DB, and fails with:

```
The default website isn't defined. Set the website and try again.
```

**Pre-install checklist:**
```bash
# 1. Drop and recreate the database (not just --cleanup-database)
docker compose exec db mariadb -uroot -p<root_pass> \
  -e "DROP DATABASE IF EXISTS magento; CREATE DATABASE magento CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 2. Delete generated config files
docker compose exec <phpfpm> rm -f \
  /var/www/html/app/etc/env.php \
  /var/www/html/app/etc/config.php

# 3. Clear compiled classes and static assets
docker compose exec <phpfpm> rm -rf \
  /var/www/html/generated/code \
  /var/www/html/generated/metadata \
  /var/www/html/pub/static/frontend \
  /var/www/html/pub/static/adminhtml
```

Then run `setup:install` without `--cleanup-database` (the DB is already empty).

### Unsupported DB version (SqlVersionProvider)

`--skip-db-validation` only bypasses the initial pre-flight check. `SqlVersionProvider` runs again during "Installing database schema" and throws even with that flag set. The correct fix is a DI array merge — place it in a **custom module's** `etc/di.xml` (e.g. `app/code/YourVendor/MagentoFixes/etc/di.xml`), not in `app/etc/di.xml`. The `app/etc/di.xml` file is owned by `magento/magento2-base` and can be overwritten during a future upgrade's root-file sync (Phase 11), silently removing the patch.

```xml
<type name="Magento\Framework\DB\Adapter\SqlVersionProvider">
    <arguments>
        <argument name="supportedVersionPatterns" xsi:type="array">
            <!-- Covers MariaDB 11.5 through 11.x (all minor versions) -->
            <item name="MariaDB-11.5-plus" xsi:type="string">^11\.\d+\.</item>
        </argument>
    </arguments>
</type>
```

The pattern `^11\.\d+\.` matches any MariaDB 11.x.y version (including 11.10, 11.11, etc.). The `[5-9]` single-digit class used in some examples only covers 11.5–11.9 and silently fails on 11.10+. This merges additively with core DI — no vendor edits required.

## Infrastructure Cache Flush

Magento's `cache:flush` only clears Magento-registered cache backends. Also flush:

```bash
# Redis (if used for cache/session)
redis-cli -h <host> FLUSHALL
# Or selective flush by DB:
redis-cli -h <host> -n 0 FLUSHDB  # default cache DB

# Varnish
varnishadm -S /etc/varnish/secret ban req.url ~ .
# Or restart:
service varnish restart

# CDN: use provider CLI or web console (Fastly, Cloudflare, etc.)
```
