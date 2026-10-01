# 10 – Publishing components

How a module, a processor or an extension becomes a package that others install with
`composer require` – and which versioning rules apply.

## How Composer finds a package

A Composer package is a Git repository with a `composer.json` in its root and **tags**
as versions. There is no "upload" – Composer fetches the tagged state directly from the
repository.

| Way | Entry in the project | Private repository? | Remark |
|---|---|---|---|
| Packagist | none | no | public repository and `license` are mandatory |
| `vcs` | `composer config repositories.blog vcs https://…/camindo-module-blog.git` | yes, with an access token on the user's side | Composer reads tags and branches directly |
| `path` | `{"type": "path", "url": "../camindo-module-blog"}` | yes (local) | development: symlink, changes take effect immediately |

Two rules determine everything else:

1. **`repositories` only counts in the project's `composer.json`.** Composer ignores
   such entries in a package's `composer.json`.
2. **Versions are tags.** `^1.0` only finds a package with a tag `v1.0.0`; a branch
   `main` is `dev-main` to Composer. Do **not** put a `version` field into
   `composer.json` – it would have to be updated with every release.

## One repository per component

```
camindo-module-blog/                repository name is free, usually camindo-{kind}-{name}
├── composer.json
├── README.md                       short: purpose, installation, license, links to docs/
├── LICENSE
├── Blog.php
├── install/
│   ├── settings.php                version, requires, tables …
│   ├── schema.sql                  CREATE TABLE IF NOT EXISTS …
│   ├── seed.sql                    optional
│   ├── Installer.php               optional
│   └── migrations/                 1.1.0.sql, 1.1.0.php, 1.2.0.sql …
├── lang/  backend/  public/
├── docs/de_DE/01-module-blog.md    manual (see 08, "A component's manual")
├── docs/en_US/01-module-blog.md
└── .gitignore                      /vendor/, composer.lock
```

The directory in the project (`Modules/Blog/`) comes from `extra.installer-name`, not
from the repository name.

```json
{
    "name": "acme/camindo-module-blog",
    "description": "Blog module for camindo CMS",
    "type": "camindo-module",
    "license": "MIT",
    "require": {
        "php": ">=8.1",
        "camindo/core": "^9.0"
    },
    "autoload": { "psr-4": { "Camindo\\Modules\\Blog\\": "" } },
    "extra": { "installer-name": "Blog" }
}
```

| Field | Rule |
|---|---|
| `name` | `{vendor}/camindo-{module\|processor\|extension}-{name}`, lower case |
| `type` | `camindo-module`, `camindo-processor` or `camindo-extension` – otherwise the package ends up in `vendor/` |
| `require.camindo/core` | the same value as `settings.php['requires']['core']` |
| `autoload` | optional – the project maps `Camindo\` to `./`; with the entry, the package is fully described |
| `extra.installer-name` | mandatory, StudlyCase |
| `version` | do not set |
| `require.camindo/composer-plugin` | do not set – the project brings the plugin |

Namespaces: modules `Camindo\Modules\{Name}\`, processors `Camindo\Processors\{Name}\`,
extensions `Camindo\Extensions\{Name}\`. Libraries an extension needs come into
`vendor/` via its `require` (asset paths with `vendor:`).

`composer.lock` does **not** belong in a component's repository – only in a project's
Git repository.

## License of a component

A component runs in the same process as the core and uses its classes – it is part of
the combined work and needs an **AGPL-compatible** license, such as AGPL-3.0, GPL-3.0,
LGPL-2.1/3.0, MIT, BSD or Apache-2.0. GPL-2.0-*only* and proprietary licenses do not fit.
Libraries a component pulls in via `require` must be compatible as well. The license
file belongs in the repository as `LICENSE`, the identifier (SPDX) in `license`.

Templates are not affected: the additional permission in `Core/LICENSING.md` exempts
them and all content.

## Versioning rules

`MAJOR.MINOR.PATCH`, tags with a `v` prefix (`v1.2.0`).

| Part | Increase when … |
|---|---|
| **MAJOR** | an installation that worked with the previous version needs to be adapted (removed commands, changed tables others use, changed paths) |
| **MINOR** | new features or a new migration – everything existing keeps working |
| **PATCH** | bug fixes only, **no migration** |

Rule of thumb: **migration ⇒ at least MINOR.** `component update` finds migrations by
comparing versions.

A component has two version declarations, increased in the same commit:

| Location | Source of truth for |
|---|---|
| `install/settings.php['version']` | the `ComponentManager` – controls which migrations run |
| Git tag `v1.2.0` | Composer – controls which files arrive |

The component names the required core twice, with the same value:

```json
"require": { "camindo/core": "^9.0" }            composer.json – checked by Composer on require/update
```
```php
'requires' => ['core' => '^9.0']                 settings.php – checked by ComponentManager on install/activate/upgrade
```

`^9.0` means "9.0.0 up to below 10.0.0". If the component runs unchanged with core 10,
it becomes `^9.0 || ^10.0` in a PATCH release; if it needs changes, a MAJOR release with
`^10.0` follows. Supported syntax in `settings.php` (`Install\VersionConstraint`): `^`,
`~`, `*`, `>=`, `<`, `||`, comma.

Pre-releases (`v1.3.0-beta.1`) sort **before** the release according to
`version_compare()` and do not satisfy `^1.3`; Composer only installs them with an
adjusted `minimum-stability`.

## Releasing a component

```bash
# 1. settings.php: 'version' => '1.2.0'; a new migration install/migrations/1.2.0.sql if needed
# 2. verify on a development installation (component as a checkout in Modules/Blog/)
php Core/cron/camindo.php component update Blog
# 3. commit, tag, push
git -C Modules/Blog commit -am "Release 1.2.0"
git -C Modules/Blog tag -a v1.2.0 -m "Blog 1.2.0"
git -C Modules/Blog push origin main --tags
```

A published tag is never moved again – Composer and Packagist cache it.

## On Packagist

Public repository, `license` in `composer.json`, submit on packagist.org and enable the
GitHub hook or the Packagist app so that new tags appear automatically. The vendor name
must still be available there.

## What users have to do

| The component lives … | Entry in the project | then |
|---|---|---|
| on Packagist | none | `composer require acme/camindo-module-blog` |
| in a public repository | `composer config repositories.blog vcs https://…` | the same |
| in a private repository | the same, plus an access token (`composer config -g github-oauth.github.com …`) | the same |

And always afterwards: `php Core/cron/camindo.php component install Blog --activate`.
The component's README names both commands. Everything else belongs in the component's
manual (`docs/`), which also appears in the backend after installation; the README links
its start page per language:

```markdown
## Manual

- English: [docs/en_US/01-module-blog.md](docs/en_US/01-module-blog.md)
- Deutsch: [docs/de_DE/01-module-blog.md](docs/de_DE/01-module-blog.md)
```
