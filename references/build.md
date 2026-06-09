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
```

Then run `setup:install` without `--cleanup-database` (the DB is already empty).

### Unsupported DB version (SqlVersionProvider)

`--skip-db-validation` only bypasses the initial pre-flight check. `SqlVersionProvider` runs again during "Installing database schema" and throws even with that flag set. The correct fix is a DI array merge in `app/etc/di.xml`:

```xml
<type name="Magento\Framework\DB\Adapter\SqlVersionProvider">
    <arguments>
        <argument name="supportedVersionPatterns" xsi:type="array">
            <!-- Add unsupported MariaDB minor versions here -->
            <item name="MariaDB-11.5-11.9" xsi:type="string">^11\.[5-9]\.</item>
        </argument>
    </arguments>
</type>
```

This merges additively with `vendor/magento/magento2-base/app/etc/di.xml` — no vendor edits required. The `getMariaDbSuffixKey()` method falls back to `10.6.11` for unrecognised 11.x versions, which is functionally correct for schema compatibility.

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
