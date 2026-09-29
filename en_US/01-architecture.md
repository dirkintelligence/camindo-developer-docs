# 01 – Architecture

## Directories

```
project/                     project root (skeleton camindo/cms)
├── composer.json            PSR-4: Camindo\ => ./ ; the core brings Camindo\Core\
├── load_camindo.php         loads configuration and bootstrap
├── .camindo.php             configuration of the installation – written by the installer, never in Git
├── .camindo.key             key for encryption, signatures, pepper
├── .sqlpwd                  credentials for mysqldump
│
├── public/                  DOCUMENT_ROOT
│   ├── index.php file.php csp_report.php     stubs → Core/public_master/…
│   ├── .htaccess            rewrite rules, partly generated
│   ├── media/               media storage {instance_id}/{field_id}/{record_id}.{ext}
│   └── backend/
│       ├── index.php module.php preview.php file.php     stubs
│       ├── assets → ../../Core/public_master/backend/assets   (symlink or copy)
│       └── ext/{name}/      published extension assets
│
├── Core/                    package camindo/core – replaced as a whole on update
│   ├── Core.php             bootstrap
│   ├── src/                 core classes (namespace Camindo\Core)
│   ├── Modules/ Processors/ Extensions/      core components
│   ├── lang/ backend/       core snippets and templates
│   ├── cron/                command line entry points
│   ├── install/             database schema, migrations, templates, skeleton
│   ├── docs/                manuals (Markdown, per language)
│   └── public_master/       entry scripts and backend assets the stubs point to
│
├── Modules/ Processors/ Extensions/          add-on components (Composer target)
├── var/                     everything writable
└── vendor/                  Composer
```

`Core/public_master/` is deliberately not called `public`: when setting up the web
server, nobody should point the document root at a directory inside the core.
Composer only writes to `Core/`, `Modules/`, `Processors/`, `Extensions/` and
`vendor/`; everything that makes an installation individual (`public/`, `var/`, the
dot files) lives outside.

## Bootstrap

There is exactly one way into the system:

```
entry script
  └── define('CAMINDO_STARTED', true)       protects includes from direct calls
      require load_camindo.php
        ├── require .camindo.php            configuration → array $_cfg (missing: [])
        └── require Core/Core.php
              ├── PHP version check, define('CAMINDO_VERSION', …)
              ├── require vendor/autoload.php
              ├── new Container(); register Config from $_cfg; unset($_cfg)
              ├── $GLOBALS['app'] = $container
              ├── BackendDomain::enforce()     backend entry points only
              └── SecurityHeaders::send()      before any output
```

`load_camindo.php` deliberately lives outside `Core/`: it belongs to the project and
survives core updates. `.camindo.php` itself checks that `CAMINDO_STARTED` is defined.
**Do not use `$_SERVER` variables** in `.camindo.php` – the file is also loaded on the
command line.

If `.camindo.php` is missing, the bootstrap continues with an empty configuration;
frontend and backend then show a notice that installation is pending (`SystemState`).

### Configuration

`Config` receives the `$_cfg` array, merges it recursively over its own defaults and
derives paths (`paths.core_dir`, `paths.camindo_dir`, `paths.var_dir`,
`paths.media_dir`, `paths.backend_dir`). Access uses dot notation:

```php
$config->get('paths.media_dir');
$config->get('admin.max_inactive_sec');
```

Every key has a default in `Config`; `.camindo.php` only contains what differs. The
annotated template of all important keys is `Core/install/templates/camindo.php.dist`.
The constructor also sets locale, time zone and the PHP error log
(`var/log/{date}_php_errors.log`).

## Dependency injection container

`Container` implements `ContainerInterface` (`get()`, `has()`, `set()`) and resolves
classes by **autowiring**:

1. instance already created → return it
2. registered factory (`set()`) → call it, remember the result
3. otherwise resolve the constructor parameters recursively via reflection

Only class types are resolved. Scalar constructor parameters cause an exception – such
classes must be registered with `set()` (in the core only `Config`). Every class is a
**shared service**: `$app->get(Admin::class)` always returns the same instance within
a request, dataset classes included. Circular dependencies are not detected; keep
constructors acyclic or load rarely used services lazily (`LazyServicesTrait`).

```php
$app = $GLOBALS['app'];                       // available in every entry script
$adminBackend = $app->get(AdminBackend::class);
```

`ProcessorFactory` resolves processors by name – via the runtime list of active
components; an add-on processor of the same name takes precedence over the core one.

## The four request paths

### 1. Frontend

```
Browser → public/.htaccess → public/index.php → Core/public_master/index.php
        → Page::resolveRequest()
```

The generated block in `public/.htaccess` (between `## CAMINDO SYNTAX START` and `END`,
with a checksum – do not edit by hand, written by `HtaccessWriter`) contains:

| Rule | Target |
|---|---|
| `^backendModules/(.*)$` | `/backend/module.php?_path=$1` |
| `^backendExtensions/(.*)$` | `/backend/module.php?_ext=$1` |
| `^([a-zA-Z]{2}(?:[-_][a-zA-Z]{2})?)/(.*)$` | `/index.php?lang=$1&request=$2` |
| `^(.*)$` (no file, not `/backend`, not `/media`) | `/index.php?request=$1` |

`Page::resolveRequest()` works in this order:

1. **Instance** from the host: `primary_domain`, then `rel_instance_domain` (literal or
   regular expression), otherwise the default instance. The source is the file cache
   `var/cache/instances.php` + `instance-{id}.php` (`InstanceCache`), not the database.
   From this, `Cms::setInstanceAccess()`: the instance itself plus the instances it may
   read.
2. **Language**: the code from the URL must be a language of the instance. If it is
   missing and the instance has several languages, `Accept-Language` decides
   (`InstanceCache::negotiateLanguage()`), otherwise the primary language applies.
   `Cms::setLanguage()` then sets the content language, the time zone
   (`language.timezone`), the snippet language (`language.backend_code`) and the locale
   of date formats (`language.code`, otherwise `backend_code`; ICU only has to know the
   language).
3. **Record**: `request` is split at commas – `{slug|id},{template},…`. A pure digit
   sequence is a `record_id`, everything else a path `[slug-prefix/]slug`
   (`Cms::recordIdsBySlugPath()`). Empty = the instance's start page.
4. **Template**, in this order: from the URL (only if permitted, see
   [06](06-template-compiler.md#template-assignment)), `record.template_id`,
   `area.default_template_id`.
5. **Canonical URL**: with `instance.canonical = 1`, every deviating address is
   redirected with 301 to the single valid one (except the start page).
6. The record must be **published** (`Cms::publishedSql()`: status 0, `online_at`
   reached); an expired record remains reachable at its address.
7. The compiled file `var/php/{area_id}/{name}.php` is included inside an IIFE; `$cms`
   (current record) and `$set` are available as globals.

If the version stamp `var/config/core.php` differs from `CAMINDO_VERSION` or
maintenance mode is on, the frontend answers with 503 before touching the database
([09](09-installation-and-update.md#maintenance-mode)).

**Preview from the backend:** `backend/preview.php` → `Page::resolvePreview()`, the same
flow with an admin session, the instance as a parameter, without publication check for
the requested record and without canonical redirect. With a fixed backend domain, the
preview is handed over to the website domain with a signed token
([14](14-security-web.md#fixed-backend-domain)).

### 2. Backend UI

`backend/index.php` delivers the static frame (header, layer container, aside, main
menu, login dialog), sets `window.cConfig` (paths, CSRF token, nonce, extension
manifest, upload limit) and loads the scripts:

```
backend.js            namespace window.cApp, jQuery extensions (cLoadContent, serializeFiles)
backend-handlers.js   nonce header, binding of marked event handlers (script policy)
backend-system.js     cApp.system    spinner, messages, CSRF header
backend-forms.js      cApp.forms     change tracking, Ajax submit
backend-ui.js         cApp.ui        lookups, tabs, QR codes …
backend-layers.js     cApp.layers    layer stack and hash routing
backend-aside.js      cApp.aside     side panel
backend-ctx-menu.js   cApp.ctxMenu   context menus
backend-polling.js    cApp.polling   background processes
backend-smartrows.js, backend-embedded.js, backend-extensions.js, backend-mail.js
backend-init.js       calls all init()
```

### 3. Backend Ajax

All dynamic content goes through **one** jQuery extension:

```js
$('#target').cLoadContent({ url: 'Admin/admin_list.php?id=5', method: 'POST', params: fd });
```

`cLoadContent()` builds the URL `/backendModules/{path}`, aborts a running request for
the same element, sends CSRF token and nonce as headers, shows a spinner, recognises
the response `##accessdenied##` (opens the login dialog) and sets the focus afterwards.

On the server:

```
/backendModules/{Module}/{script}
  → /backend/module.php?_path={Module}/{script}
      → Backend::backendPath()
          ├── sanitise the path, resolve the component via the runtime list
          ├── Session::load()             no session: ##accessdenied## (except Start/login.php)
          ├── Backend::originAllowed()    foreign origin on POST → ##accessdenied##
          ├── Session::verifyCsrfRequest() missing token on POST → ##accessdenied##
          └── Access::script()            module right, extra right of the script
      → require {component}/backend/{script}
```

`/backendExtensions/{Extension}/{script}` works the same way but only requires a
session (extensions have no module rights). Further entry points: `backend/file.php`
for **every** media file in the backend, `public/file.php` (`MediaDelivery`) for
protected files and downloads in the frontend.

### 4. Command line and cron

```
php Core/cron/camindo.php <command>     installation, update, caches, components, keys …
php Core/cron/cron.php                  every minute via cron: TriggerCron (background work)
php Core/cron/trash.php                 nightly: empty the trash
```

All scripts under `Core/cron/` refuse to run if `PHP_SAPI !== 'cli'` and define
`CAMINDO_CLI`. Details: [07](07-media-processing.md#cron) and
[09](09-installation-and-update.md#the-command-line).

## Layers and hash routing in the backend

The URL hash represents the state of the backend; several paths joined with `~` are
stacked layers:

```
#Record/record_list.php?area=8~Record/record_edit.php?id=42
 └── layer 0 ──────────────┘ └── layer 1 ─────────────────┘
```

| Method | Effect |
|---|---|
| `cApp.layers.add(path)` | put a new layer on top |
| `cApp.layers.replace(path)` / `replacePost(path, params)` | replace the top layer (GET or POST) |
| `cApp.layers.close()` | close the top layer |
| `cApp.layers.refresh()` | reload the top layer |
| `cApp.layers.invalidate([callback], [levels])` | mark the layer below (or `levels` layers) as stale – it reloads as soon as it is on top again |
| `cApp.layers.removeReferer()` | remove the layer below |
| `cApp.aside.open(path)` / `openPost(path)` | load the side panel |

After saving, deleting or importing, the rule is:
`cApp.layers.invalidate(); cApp.layers.close();` (or `replace()` instead of `close()`).

## Core classes (`Core/src/`)

| Class | Purpose |
|---|---|
| `Cms` | the heart: field types, schema of `record`, `select()`/`fetch()`, snippets, formatters, runtime of the `(CMS:…)` commands |
| `Dataset` (abstract) | table abstraction: `init/load/save/delete/restore`, validation, hooks, audit |
| `VirtualFieldsTrait` | virtual fields, uploads, thumbnails, relations, full text |
| `TreeManagerTrait` | nested set + adjacency list, one tree per content area |
| `TemporaryTrait` | working copies: `checkOut`, `checkIn`, `cancel`, edit locks |
| `LinkTrait`, `MoveTrait` | dependent linked content; moving between areas |
| `Backend` | UI building blocks: templates, form fields, tabs, buttons, lists, script policy |
| `Page` | resolve frontend requests, preview, run templates |
| `Database` | mysqli wrapper: prepared statements, result stack, transactions, `afterCommit()` |
| `Config`, `Container`, `System` | configuration, DI, logging/files/directories |
| `Session`, `Access`, `Integrity` | admin session, group rights, signatures – [11](11-permissions.md), [13](13-security-authentication.md) |
| `UserSession`, `MediaDelivery`, `MediaAccess` | frontend community and protected media – [12](12-community.md) |
| `InstanceCache` | file cache for instances, domains, languages, areas and group rights |
| `Crypt`, `KeyRotation`, `PasswordPolicy`, `Totp`, `LoginGuard`, `LoginDevices`, `RateLimiter`, `FormGuard`, `ClientIp` | security – [13](13-security-authentication.md) |
| `SecurityHeaders`, `HtmlSanitizer`, `BackendDomain`, `FrontendHandover`, `CspReport` | web security – [14](14-security-web.md) |
| `AuditLog`, `AuditStore`, `AuditDiff` | audit log |
| `MediaProcessor`, `ProcessorFactory`, `Execute`, `BackgroundProcess`, `MassDeletion` | media and background processes – [07](07-media-processing.md) |
| `ComponentManager`, `SnippetCache`, `HtmlCache`, `HtaccessWriter`, `SystemState`, `Backup`, `UpdateCheck`, `SystemHealth` | components, caches, operation – [08](08-components-and-extensions.md), [09](09-installation-and-update.md) |
| `Pagination`, `Email`, `History`, `Widgets`, `TriggerTemplates` | page navigation, e-mail, "recently edited", start page widgets, trigger templates |

### `Database` – the result stack

`Cms::select()` is called in nested fashion from templates. `Database` therefore keeps
an explicit stack of results:

```php
$this->db->pushResult();   // save the current result
$this->db->query(…);       // sub-query
$this->db->popResult();    // restore the previous result
```

The template compiler does this automatically for `CMS:SELECT`. If your PHP code runs
a second query while iterating over a result, wrap it yourself.

Helpers: `fetch()`, `fetchAll()` (single-column results become a flat list),
`fetchNum()`, `fetchColumn()`, `fetchPairs()`, `fetchAndIndex()`,
`beginTransaction()`/`commit()`/`rollback()`/`inTransaction()` and `afterCommit()` for
callbacks after the outermost commit.

## Core modules

| Module | Purpose |
|---|---|
| `Admin` | administrators, user groups, personal settings, second factor, devices |
| `Area` | content areas and input fields (`Area`, `Field`), blueprints |
| `Audit` | browse the audit log (superusers only) |
| `Component` | manage components (UI for the `ComponentManager`) |
| `Converter` | converters = configured processors |
| `Docs` | manuals in the backend, full-text index |
| `Instance`, `Language` | websites and languages |
| `Logfiles` | logs under `var/log` (superusers only) |
| `Messaging` | internal messages between administrators |
| `Record` | records: form, list, search, import/export, batch processing; `RecordUser` = runtime of the `CMS:USER_*` commands |
| `Scheduler` | scheduled tasks |
| `Snippets` | custom text modules per instance |
| `Start` | start page, login, main menu, background processes |
| `Status` | record status |
| `Template` | templates, parser/compiler, command definitions |
| `Tools` | caches, database maintenance, server information, developer tools |
| `Trash` | trash |
| `User` | backend view of the frontend community (review) |

Core processors: `ImageGd`, `Magick`, `FFmpegVideo`, `FFmpegAudio`, `FFmpegImage`.
Core extensions: `Calendar`, `Datepicker`, `IconPicker`, `Markdown`, `Password`,
`Snippets`.
