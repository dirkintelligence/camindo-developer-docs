# 09 – Installation, update and command line

The administrators' manual describes the installation step by step. This chapter
explains the mechanics behind it: packages, versions, migrations, the installer and
update procedures, and the command line.

## Principle: three levels, one direction

Every installation, update and uninstallation touches three levels:

```
FILES                     DATABASE                      RUNTIME
Composer                  migrations                    generated files under var/
Core/ Modules/ …          schema, seed, migrations/     var/config/components/*.php
vendor/                   tables component, system      var/config/core.php, caches
```

1. **Adding and updating: files → database → runtime.** Composer puts the files in
   place; the database follows in a deliberate, separate step; the runtime files are
   derived from both. Never automatically from a Composer hook – it also runs on build
   servers without a database.
2. **Removing: runtime → database → files.** Uninstalling needs the component's files;
   so first `component uninstall`, then `composer remove`.
3. **The runtime never asks the database about the state.** What is active, which
   version the core has, whether maintenance is running – all of that is in files under
   `var/`.
4. **There may be a window between files and database, but no crash.** A component with
   a pending update is inactive (the function is missing, nothing breaks); for the core,
   maintenance mode applies.

## Packages

| Package | Composer type | Installed to | Content |
|---|---|---|---|
| `camindo/cms` | `project` | project root | the skeleton: `composer.json`, `load_camindo.php`, `public/` with stubs, empty directories (source: `Core/install/draft/`) |
| `camindo/core` | `camindo-core` | `Core/` | the CMS |
| `camindo/composer-plugin` | `composer-plugin` | `vendor/` | puts the camindo package types in their place |
| `{vendor}/camindo-module-{name}` | `camindo-module` | `Modules/{Name}/` | add-on module |
| `{vendor}/camindo-processor-{name}` | `camindo-processor` | `Processors/{Name}/` | add-on processor |
| `{vendor}/camindo-extension-{name}` | `camindo-extension` | `Extensions/{Name}/` | extension |

**The Composer plugin** is a subclass of Composer's `LibraryInstaller` that only
determines the target path (`Core/`, `Modules/{name}/` …). `{name}` comes from the
component's `extra.installer-name` (StudlyCase, mandatory) – Composer enforces
lower-case package names, but directories are class names. Without a valid entry, the
plugin refuses the installation. It is a separate plugin rather than
`composer/installers` because the latter only knows a fixed list of framework types.

The skeleton's `composer.json` (abridged):

```json
{
    "name": "camindo/cms",
    "type": "project",
    "require": { "php": ">=8.1", "camindo/composer-plugin": "^1.0", "camindo/core": "^9.0" },
    "autoload": { "psr-4": { "Camindo\\": "" } },
    "config": { "allow-plugins": { "camindo/composer-plugin": true } },
    "scripts": {
        "post-create-project-cmd": "@php Core/cron/camindo.php install",
        "post-install-cmd":        "@php Core/cron/camindo.php status --after-composer",
        "post-update-cmd":         "@php Core/cron/camindo.php status --after-composer"
    }
}
```

`Core/composer.json` brings `Camindo\Core\`; `Camindo\` → `./` covers the add-on
components. The hooks are **passive**: `status --after-composer` writes nothing,
tolerates a missing database and only reports what is pending:

```
camindo 9.1.0 – core update pending (database: 9.0.3)   -> php Core/cron/camindo.php upgrade
Modules/Blog 1.2.0 – update pending (installed: 1.1.0)  -> php Core/cron/camindo.php component update Blog
```

**Composer only writes to `Core/`, `Modules/`, `Processors/`, `Extensions/`, `vendor/`
and `composer.lock`.** Everything that makes an installation individual lives outside –
which is why the document root must never be inside `Core/`.

### Developing the core

If you develop the core itself, keep `Core/` as a Git checkout and do not let Composer
touch it. The development project's `composer.json` declares via `replace` that it
provides `camindo/core` itself (template `Core/install/draft/composer-dev.json`):

```json
{
    "name": "camindo/cms-dev",
    "replace": { "camindo/core": "9.x-dev" },
    "require": { "camindo/composer-plugin": "^1.0", "phpmailer/phpmailer": "^7.0", "…": "…" },
    "autoload": { "psr-4": { "Camindo\\": "", "Camindo\\Core\\": "Core/src/" } }
}
```

Add-on components requiring `camindo/core: ^9.0` are satisfied by this. A component
under development lives as its own checkout directly in `Modules/{Name}/`;
`component install` works with the files regardless of where they come from.

## Versions

| Location | Value | Written by |
|---|---|---|
| `CAMINDO_VERSION` in `Core/Core.php` | `9.1.0` | developer, at release |
| Git tag of the core repository | `v9.1.0` | developer – Composer derives the package version from it |
| `system.core_version` + `var/config/core.php` | `9.1.0` | installer and `upgrade`, after successful migrations |

`var/config/core.php` (`['version' => …, 'upgraded_at' => …]`) is the **version stamp**
for the runtime: if it differs from `CAMINDO_VERSION`, the frontend answers with 503
and the backend shows superusers the hint to run `upgrade` – new code never reaches the
frontend on an old schema. If the stamp (and `var/install.lock`) is missing, the
installation is considered not present.

For add-on components, `install/settings.php['version']` is authoritative; the
installed version is stored in `component.version`, a newer file version in
`update_version`.

## Schema and migrations

`Core/install/database/schema.sql` (structure) and `seed.sql` (base data) always
describe the **current** state; a fresh installation needs no migration. Migrations
bring existing installations there. The rule for every schema change in the core:

1. write a migration: `Core/install/migrations/{version}.sql` and/or `.php`
2. update `schema.sql`/`seed.sql` – via `php Core/cron/camindo.php schema export`, not
   by hand (only with `system.development_version`)
3. both in the same commit

The file name is the version that **introduces** the migration. When updating from
`from` to `to`, all files with `from < v <= to` run, sorted by `version_compare()`,
`.sql` before `.php` of the same version. Components have the same in
`install/migrations/`; core components have none of their own – their tables are core
tables.

**`.sql`** – statements separated by `;` at the end of a line, executed one by one (an
error names the statement). `DELIMITER` is not supported.

**`.php`** – returns a closure:

```php
<?php
use Camindo\Core\Install\InstallContext;

return static function (InstallContext $ctx): void {
    if (!$ctx->columnExists('record', 'owner_id')) {
        $ctx->db->query("ALTER TABLE `record` ADD `owner_id` bigint UNSIGNED NULL");
    }
    $ctx->db->query("UPDATE `record` SET `owner_id` = `created_by` WHERE `owner_id` IS NULL");
    $ctx->log('owner_id filled');
};
```

Rules:

* **A pointer, not a log.** The state reached (`system.core_version` or
  `component.version`) is updated after **every** version; after an abort, the next
  run starts there. The log is in `var/log/{date}_upgrade.log`.
* **Idempotent where cheap** (`IF NOT EXISTS`, `columnExists()`). DDL is not
  transactional in MySQL – the safety net is the backup before the run.
* **No backend classes** (`Backend`, `Session`). Dataset classes run with new code
  against a half-migrated schema – when in doubt, use plain SQL.
* **Migrations do not build caches** – `upgrade` does that at the end.
* No seed row with id 0 in tables that have an edit form (0 means "new" there).

## The installer

```
composer create-project camindo/cms mysite     skeleton + core + vendor
    └─ post-create-project-cmd: camindo install
```

Also available later: `php Core/cron/camindo.php install`. There is deliberately **no
web installer** – whoever runs Composer has a shell, and a web installer would be a
second, unprotected path into the database.

| Step (`--step=`) | Does |
|---|---|
| `requirements` | PHP version and extensions, write permissions, look for programs (`magick`, `ffmpeg`, `mysqldump`) |
| `key` | create `.camindo.key` (32 random bytes, `0600`) – **losing it = encrypted content unreadable** |
| `config` | `.camindo.php` from `Core/install/templates/camindo.php.dist` |
| `sqlpwd` | `.sqlpwd` for `mysqldump` |
| `dirs` | `var/…`, `public/media/default`, symlink `public/backend/assets` (`--copy-assets`: copy) |
| `database` | `schema.sql` + `seed.sql` |
| `admin` | first user (superuser); the password must meet the policy |
| `core` | `system.core_version` |
| `components` | **discover** add-on components, do not install them |
| `runtime` | runtime lists, caches, `.htaccess` block, version stamp |
| `finish` | `var/install.lock`, summary with cron lines |

If `var/install.lock` exists, `install` ends immediately with exit code 0;
`--force --yes` deliberately starts over (the database is emptied). Single steps can be
repeated with `--step=NAME` to repair a damaged installation.

**Non-interactive** (Docker, CI): every question has a parameter and an environment
variable – `--db-host` / `CAMINDO_DB_HOST`, `--db-name`, `--db-user`, `--db-pass-file`,
`--admin-name`, `--admin-email`, `--admin-pass-file`, `--admin-lang`, `--timezone`,
`--locale`, `--https=y|n`, `--backend-domain` (each `CAMINDO_…`). Passwords only from
files or environment variables, never as an argument (process list).
`CAMINDO_DB_PASS` also applies at runtime ([13](13-security-authentication.md#secrets-from-the-environment)):
if the password comes from there, the installer does not write it to `.camindo.php` and
creates no `.sqlpwd`; with `CAMINDO_KEY`, no `.camindo.key` is created. With
`--no-interaction`, a missing mandatory value aborts with exit code 2 before anything
is written. `--create-db` creates the database, `--fixtures` loads a development data
set (`dev.sql`, if present) instead of the base data.

## Core update

```
php Core/cron/camindo.php maintenance on        recommended
composer update camindo/core                    files; the hook reports "core update pending"
php Core/cron/camindo.php upgrade               database and runtime
```

`upgrade` in detail (every step is logged; an error aborts and leaves maintenance mode
on):

1. set maintenance mode if not set
2. backup: `mysqldump` → `var/dbbackup/upgrade-{from}-{to}-{time}.sql`
3. core migrations from `system.core_version` to `CAMINDO_VERSION`
4. reconcile core components in `component`
5. `ComponentManager::refresh()`; unless `--no-components`, update all add-on
   components with a pending update in dependency order – one whose `requires.core`
   does not allow the new version stays inactive
6. runtime: lists, snippet and HTML caches, extension assets, `InstanceCache`,
   `.htaccess` block, recompile templates, manual index
7. `system.core_version`, `var/config/core.php`
8. lift maintenance mode (only its own, or with `--unlock`)

`upgrade` is **repeatable**: without a version change, only the runtime part runs –
which is also the "rebuild everything" command after a deployment.

**Rollback** is a recipe, not a command – DDL cannot be rolled back, and the files
belong to Composer:

```
php Core/cron/camindo.php maintenance on
mysql … < var/dbbackup/upgrade-9.0.3-9.1.0-….sql      database back
git checkout composer.lock && composer install         files back
php Core/cron/camindo.php upgrade                      runtime anew
php Core/cron/camindo.php maintenance off
```

Hence: `composer.lock` belongs in the **project's** Git repository.

## Add-on components

```
composer require acme/camindo-module-blog                 files to Modules/Blog/
php Core/cron/camindo.php component install Blog --activate

composer update acme/camindo-module-blog                  new files
php Core/cron/camindo.php component update Blog           migrations – right afterwards

php Core/cron/camindo.php component uninstall Blog [--purge]
composer remove acme/camindo-module-blog
```

* `install` creates tables and the row; the component is **inactive** afterwards
  (creating the schema and switching it on are separate decisions); `--activate` is the
  shortcut.
* `update` keeps the status; between `composer update` and `component update`, new code
  runs on the old schema – so run both directly one after the other, in a deploy script
  under maintenance mode.
* `uninstall` keeps the tables by default; `--purge` drops those listed in `tables`.
  Because `tables` is stored in the row, even an orphaned component (files already
  gone) can be removed completely.
* A backup runs before `install`, `update` and `uninstall` (`--no-backup` turns it
  off).

## Maintenance mode

`var/maintenance.lock` (time and reason) – set by `maintenance on`, `upgrade` and in the
Tools module.

| Where | Behaviour |
|---|---|
| frontend (`SystemState::guardFrontend()`, before any database access) | 503 with `Retry-After`, static page (can be overridden by `var/custom/maintenance.html`) |
| backend | superusers: notice bar, everything reachable; others: maintenance page |
| cron scripts | exit immediately |
| command line | runs |

The same behaviour applies when the version stamp does not match the code version.

## The command line

One entry point, sub-commands, no library:

```
php Core/cron/camindo.php <command> [arguments] [--options]
```

| Command | Does |
|---|---|
| `status [--after-composer] [--json]` | files vs. database: versions, pending migrations and updates, maintenance mode |
| `install [--no-interaction --fixtures --create-db --copy-assets --force --yes --step=…]` | first installation |
| `upgrade [--no-components --no-backup --unlock]` | core migrations, component updates, runtime, caches |
| `maintenance on [--reason=…] \| off \| status` | maintenance mode |
| `backup [--reason=…] \| list \| prune --keep=N` | backups to `var/dbbackup/` |
| `schema export` | `schema.sql`, `seed.sql`, `dev.sql` from the development database |
| `cache snippets\|html\|docs\|templates\|include\|all [--prefix=… \| --remove=… \| --template=…] \| list` | caches |
| `component list \| install \| activate \| deactivate \| update \| uninstall NAME` | life cycle of a component; `install --all`, `update --all` |
| `refresh` | directories → `component`, runtime lists |
| `admin password NAME [--pass-file=…] \| sign` | set a backend password; rewrite all signatures |
| `key status \| rotate \| retire KID` | key ring of `.camindo.key` ([13](13-security-authentication.md#key-rotation)) |
| `audit verify [TYPE ID] \| restore [--dry-run] \| migrate` | verify the audit log chain; after restoring a dump, cut the files back to the database state |

Exit codes: 0 ok, 1 error, 2 usage error; `--quiet` suppresses everything except
errors. Commands are classes under `Core/src/Cli/{Name}Command.php`. Installation and
core update exist on the command line only – a core update from the backend would
replace the code that is running it.

Further scripts under `Core/cron/`: `cron.php` and `trash.php` (cron),
`rebuild_trees.php` (repair trees), `check_dateparse.php`, `check_nested_select.php`,
`check_tree_positions.php`, `check_webauthn.php` (self-tests).
