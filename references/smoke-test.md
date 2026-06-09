# Smoke Test Reference

## Minimum Passing Criteria

All of these must pass before the upgrade is considered successful:

- [ ] Storefront homepage loads (HTTP 200, no blank page)
- [ ] Category page loads with products visible
- [ ] Product detail page loads
- [ ] Add to cart works
- [ ] Checkout page loads (guest checkout)
- [ ] Admin login page loads
- [ ] Admin login succeeds (full dashboard visible, no JS errors)
- [ ] No critical errors in `var/log/system.log` or `var/log/exception.log`

## Storefront Checks

```bash
# Quick HTTP check
curl -sI https://<store-url>/ | head -5
curl -sI https://<store-url>/catalog/product/view/id/1 | head -5

# Check for blank white page (Magento silent 200 with empty body)
curl -s https://<store-url>/ | wc -c
# Should be > 10000 bytes for a real page
```

Open in browser and verify:
- Navigation menu renders
- Category images visible
- No "Something went wrong" messages
- No raw PHP errors visible

## Admin Checks

```bash
curl -sI https://<store-url>/admin | head -5
```

In browser:
- Login page renders with Magento logo
- Login succeeds → dashboard visible
- Open browser console → no critical JS errors (red)
- Check one grid (e.g., Orders) loads without 500

> **Credential tip:** If the admin password is unknown, reset it via CLI (safe — `admin:user:create` updates the password if the username already exists, it does not create a duplicate):
> ```bash
> bin/magento admin:user:create --admin-user=admin --admin-password=Test1234! \
>   --admin-email=admin@example.com --admin-firstname=Admin --admin-lastname=Test
> ```

> **"Page Cache invalidated" banner:** The yellow admin banner flagging `full_page` as invalidated immediately after upgrade is expected — `setup:upgrade` marks it stale internally even after `cache:flush`. Run `bin/magento cache:clean full_page` or click Cache Management. This is not a sign of a failed upgrade.

## Log Triage

```bash
# Check for new errors since upgrade
tail -100 var/log/exception.log
tail -100 var/log/system.log

# Filter for critical errors
grep -i "critical\|fatal\|exception" var/log/exception.log | tail -50
```

Common post-upgrade errors to look for:
- `Class not found` → stale generated code; re-run `setup:di:compile`
- `Table doesn't exist` → `setup:upgrade` didn't complete; re-run
- `Invalid return type` → third-party module type-hint mismatch with upgraded Magento
- `Memory exhausted` → check `php.ini` `memory_limit` — some upgrades need more

## Search Check

```bash
# If Elasticsearch/OpenSearch in use:
curl -s <es-host>:9200/_cat/health
curl -s <es-host>:9200/_cat/indices | grep magento
```

In storefront: search for a product keyword and verify results appear.

## Indexer Status

```bash
bin/magento indexer:status
```

All indexers should show `Ready`. If any show `Reindex required`, run:

```bash
bin/magento indexer:reindex
```

## Cron Status

```bash
bin/magento cron:status
```

Confirm cron groups are running. If not:

```bash
bin/magento cron:install
bin/magento cron:run
```
