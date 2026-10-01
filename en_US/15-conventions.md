# 15 – Conventions

## Names

### Files

| Kind | Style | Example |
|---|---|---|
| classes, interfaces, traits | PascalCase | `AdminBackend.php`, `TreeManagerTrait.php` |
| view and configuration scripts | snake_case | `admin_edit.php`, `settings.php` |
| HTML templates | snake_case | `admin_table_row.html` |
| assets (JS/CSS) | kebab-case | `backend-layers.js` |

### PHP (PSR-12)

| Kind | Style | Example |
|---|---|---|
| class, interface, namespace | PascalCase | `Camindo\Core\Modules\Admin` |
| method, function, variable, property | camelCase | `sendEmail()`, `$isAuthorized` |
| constant | UPPER_CASE | `DEFAULT_PORT` |

Private helper methods of a public method carry its name with an underscore as a
prefix: `edit()` → `_editSave()`, `_editForm()`.

### JavaScript

Classes PascalCase, functions and variables camelCase, global constants UPPER_CASE,
private fields with `#`. The only global objects are `window.cApp` and
`window.cConfig`.

### CSS

Ids and classes kebab-case, BEM variants with `--`. Project classes carry the prefix
`c9-` (`c9-layer`, `c9-ctx-row`). Customisations of Bootstrap itself do not belong in
`backend.css` but in the build of the Bootstrap files.

### Database

| Element | Convention |
|---|---|
| table | singular, snake_case: `admin`, `data_txt`, `rel_area_template` |
| primary key | `{table}_id` (exception: `bgproc.id`) |
| edge table | `rel_{a}_{b}` |
| data table | `data_{suffix}` according to `Cms::$dataSuffix` |
| index / foreign key | `idx_*` / `fk_*` |

### Snippet keys

| Pattern | Example |
|---|---|
| `field_{table}_{column}` | `field_admin_name` |
| `message_{table}_{action}` | `message_admin_saved` |
| `alert_{table}_{action}` | `alert_admin_delete` |
| `error_{what}` | `error_value_mandatory` |
| `button_{name}`, `headline_{what}`, `tooltip_field_{name}` | `button_back` |
| `speak_{what}` (screen reader) | `speak_open_menu` |
| `access_{right}` | `access_publish` |
| `module_label`, `module_description` | – |

With prefix: `admin.field_admin_name`, `proc.imagegd.field_width`,
`ext.markdownvisual.label`. Keys contain no dots.

## Code style

* Type consistently, including return values; constructor property promotion for
  dependencies.
* Braces: classes and functions on the next line, control structures on the same line.
* Identifiers (classes, methods, variables, columns, snippet keys) in English.
* Always take over input typed (`(int)$_REQUEST['id']`); `bit` via
  `filter_var(…, FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE)`.
* Protect includes: `if (!defined('CAMINDO_STARTED')) die('access denied.');`
* Encapsulate with an IIFE where the global namespace must stay clean (that is how
  `Page` includes templates).

| Constant | set in |
|---|---|
| `CAMINDO_STARTED` | every entry script |
| `CAMINDO_BACKEND` | backend entry points (`index.php`, `module.php`, `file.php`, `preview.php`) |
| `CAMINDO_PREVIEW` | `backend/preview.php` |
| `CAMINDO_CLI` | scripts under `Core/cron/` |
| `CAMINDO_VERSION` | `Core/Core.php` |

## Output and headers

* **The frontend does not buffer.** Headers and cookies belong at the top of a
  template, before the first output – `CMS:HEADER`, `CMS:CSP`, `CMS:USER_*` and the
  first `CMS:USER_TOKEN`.
* In the backend, a backend method outputs HTML; widgets **return** it.
* Escape values in HTML that do not come from `fetch()`; escape `"` in attributes, use
  `Backend::jsString()` in JavaScript ([14](14-security-web.md#html-filter)).
* No handler attributes in an `echo`, no `eval`
  ([14](14-security-web.md#script-policy-in-the-backend)).

## SQL

* **Always prepared statements**: `Database::query($sql, $bindTypes, $values)`.
  Sanitise identifiers to `[a-z0-9_]` beforehand.
* Multi-step write operations in one transaction (`Dataset::transactional()`).
* Wrap sub-queries during an iteration with `pushResult()`/`popResult()`.
* Store date and time values in UTC (`UTC_TIMESTAMP()`), convert for display.
* In field values, only SQL functions from `Cms::$funcWhitelist`.
* Read content via `Cms::select()` – instance, area and deletion conditions then apply
  automatically. If you query directly, add them yourself.
* Write tables with rights signatures only through the classes that sign
  ([13](13-security-authentication.md#signatures)).

## Language, time zone, text direction

| Context | Time zone | Snippet language |
|---|---|---|
| backend | `admin.settings.timezone` | `admin.settings.backend_code` |
| frontend | `language.timezone`, empty = `system.timezone` | `language.backend_code` |
| CLI, no session | `system.timezone` | `en_US` |

Values are stored in UTC and displayed in `Cms::$userTZ`. Date and number formats follow
the user's locale or `system.locale` in the backend, and the content language in the
frontend (`language.code`, otherwise `language.backend_code`).

**Text direction (RTL).** If a backend language from `Cms::RTL_LANGUAGES` is set
(Arabic, Persian, Urdu, Hebrew …), the backend carries `<html dir="rtl">`;
`Cms::isRtl()` / `Cms::textDirection()` return the direction for your own output (e.g.
e-mails). For modules and extensions to follow:

* logical CSS properties for anything asymmetric: `margin-inline-start/-end`,
  `padding-inline-*`, `border-inline-*`, `inset-inline-*`, `text-align: start/end` –
  also in inline styles of templates
* Bootstrap utilities in BS5 notation (`ms-*`, `me-*`, `ps-*`, `pe-*`, `text-start`,
  `float-end`)
* icons with a direction (arrows, chevrons) get the class `icon-dir`
* JavaScript that calculates pixel positions checks
  `document.documentElement.dir === 'rtl'`

## File permissions and writable directories

Files created by the CMS `0664`, directories `0775` (default, configurable in
`permissions.file` / `permissions.dir`), owned by the web server user; `.camindo.php`, `.camindo.key`,
`.sqlpwd` with `0600`. `System::createDir()` creates directories with the configured
permissions.

Only `var/` and `public/media/` are writable:

```
var/cache/       snippets, HTML templates, instances, groups, triggers, update check
var/config/      runtime lists, version stamp, ImageMagick policy
var/templates/   template sources                  var/php/      compiled templates
var/custom/      snippets per instance, scheduler scripts
var/extensions/  project-specific extension files
var/audit/       audit log contents                var/log/      logs
var/tmp/ var/upload/ var/import/ var/export/ var/download/ var/deleted/ var/dbbackup/
public/media/    uploads and derived media versions
```

`var/` lives outside the document root.

## Tests

camindo does not ship a test suite, but self-tests for delicate areas
(`Core/cron/check_*.php`) and the headless path for backend scripts
([05](05-backend-development.md#testing-without-a-browser)). Run the self-tests after
changing `Dataset`, trees or the template compiler.
