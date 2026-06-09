# magento-upgrade

A Claude Code skill for upgrading Magento 2 / Adobe Commerce installations — patch, minor, and major versions.

## What it does

When invoked, the skill walks Claude through a structured 20-phase upgrade process: environment detection, backup, branch, UCT compatibility check, version bump, composer update, root file sync, build commands, smoke test, and a final lessons-capture step that can feed improvements back into the skill itself.

## Trigger conditions

Claude will automatically activate this skill when:

- A target Magento version is specified
- `composer.json` version constraints need updating
- Extension or module compatibility is unknown
- A maintenance window is being planned
- UCT (`upgrade:check`) warnings need interpretation
- Deprecated or removed APIs are found
- A version bump is blocked by constraint conflicts
- DB migration is required after a version change

## Phases

| # | Phase | Notes |
|---|---|---|
| 0 | Environment detection | Docker check — all commands run inside the container |
| 1 | Backup | DB dump + file backup verification |
| 2 | Branch | Check for existing upgrade branch before creating |
| 3 | Release notes review | Flag breaking changes for the target version |
| 4 | Vendor sync | `composer install` |
| 5 | UCT / compatibility check | Required — never skipped |
| 6 | Rollback plan | Documented before any code or DB changes |
| 7 | Cron disable + maintenance on | Cron disabled first to prevent mid-migration corruption |
| 8 | Infrastructure gate | Major upgrades only — PHP, DB, ES/OS, Redis, Composer |
| 9 | Version bump | `composer.json` metapackage constraint update |
| 10 | Composer update | `-W` flag for major upgrades |
| 11 | Root file sync | `magento2-base` conflict resolution |
| 12 | Commit | Composer files + resolved root conflicts |
| 13 | setup:upgrade | Critical — DB schema sync |
| 14 | DI compile | `setup:di:compile` |
| 15 | Static content deploy | Locale/theme explicit in production |
| 16 | Cache clean | Magento + infrastructure caches (Redis, Varnish, CDN) |
| 17 | Reindex + cron re-enable + maintenance off | |
| 18 | Smoke test | Must pass before deployment checklist |
| 19 | Deployment checklist | Written to `tasks/<version>/deployment-checklist.md` |
| 20 | Lessons + skill update | Captures learnings; prompts to apply them back into this skill |

## Reference files

| File | Purpose |
|---|---|
| `references/compatibility-check.md` | UCT output interpretation, severity levels, third-party module handling |
| `references/version-bump.md` | Metapackage patterns, constraint widening, minimum-stability, downgrade procedure |
| `references/build.md` | setup:upgrade failures, DI compile errors, static content deploy, SqlVersionProvider workaround |
| `references/root-file-sync.md` | magento2-base managed files, conflict resolution strategy |
| `references/smoke-test.md` | Storefront/admin verification, JS errors, log triage |
| `references/deployment-checklist.md` | Full checklist template and QA notes rules |
| `references/common-mistakes.md` | Indexed mistakes across all upgrade phases including Docker/markshust gotchas |

## Installation

Place this directory at `~/.claude/skills/magento-upgrade/` (or wherever your Claude Code skills directory is configured). Claude Code picks it up automatically via the skill registry.

## Self-improvement

Phase 20 prompts you to apply lessons from each upgrade back into `SKILL.md` and the reference files. Over time this means the skill improves with every project it runs on. Skill updates are committed separately from the upgrade branch so the history stays clean.
