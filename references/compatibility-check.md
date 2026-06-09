# Compatibility Check Reference

## Upgrade Compatibility Tool (UCT)

Adobe's official static analysis tool. Install if not present:

```bash
composer require magento/upgrade-compatibility-tool --dev
```

**Magento 2.4.8+ install conflict:** UCT 3.0.21+ requires `symfony/dependency-injection ^6.4` but Magento 2.4.8 locks to `symfony/dependency-injection v7.4.x`. The install fails — this is not resolvable without breaking Magento's Symfony 7 dependencies. Workarounds:
- **Zero custom modules:** Document the skip in `tasks/<version>/uct-result.md` and proceed. Risk is zero with no code to scan.
- **Custom modules present:** Run UCT from a separate standalone installation (clone UCT separately, point it at the project path via `--magento-path`).
- **With `-W`:** Resolves only if ALL transitive deps can satisfy both constraint sets simultaneously — rarely succeeds on 2.4.8.

Run against the target version:

```bash
bin/magento upgrade:check --coming-version=<target_version> /path/to/magento
# Or via vendor/bin:
vendor/bin/upgrade-compatibility-tool upgrade:check --coming-version=<target_version> /app
```

## Severity Levels

| Level | Meaning | Action |
|---|---|---|
| **Critical** | Removed class/method used in custom code | Must fix before upgrade proceeds |
| **Error** | Deprecated API that will break at target version | Fix or find alternative |
| **Warning** | API deprecated but not yet removed | Fix if feasible, track for next upgrade |

## Output Interpretation

UCT reports are noisy. Focus on:
1. Issues in `app/code/` and `app/design/` (your custom code)
2. Issues in installed third-party modules
3. Ignore issues in `vendor/` unless it's a path-repo module you own

## Third-Party Marketplace Modules

**UCT cannot assess Marketplace compatibility** — it only scans code paths it can reach. For each third-party module:

1. Check the module's page on Adobe Commerce Marketplace for supported version range
2. Check the module's `composer.json` `require` constraint against target Magento version
3. Check the module vendor's changelog / release notes
4. If no compatible version exists: evaluate fork, remove, or find alternative

## Common False Positives

- Reports on `vendor/` classes your custom code extends — check if the API actually changed
- "Unused" warnings on event observers — these are runtime-wired, not statically analysable
- Deprecation warnings on code scheduled for removal in a version *beyond* your target

## PHP Version Check

```bash
php -v
# Compare against: https://experienceleague.adobe.com/en/docs/commerce-operations/installation-guide/system-requirements
```

Magento version → PHP compatibility matrix (key versions):
- Magento 2.4.9.x → PHP 8.4
- Magento 2.4.8.x → PHP 8.4
- Magento 2.4.7.x → PHP 8.2, 8.3
- Magento 2.4.6.x → PHP 8.1, 8.2
- Magento 2.4.5.x → PHP 8.1
- Magento 2.4.4.x → PHP 8.1

Always verify against the official system requirements page for the exact target version.
