# 08 – Components and extensions

## Terms

| Term | Meaning |
|---|---|
| ***Component*** | umbrella term for module, processor and extension |
| **Core component** | lives under `Core/Modules`, `Core/Processors`, `Core/Extensions`; replaced only together with the core, always installed and active |
| **Add-on component** | lives under `Modules/`, `Processors/`, `Extensions/`; can be installed, activated, updated and uninstalled individually |
| **Module** | backend functional unit with scripts, dataset classes, snippets, templates; optionally its own `(CMS:…)` commands |
| **Processor** | media conversion ([07](07-media-processing.md)) |
| ***Extension*** | JavaScript (+ CSS, + optional endpoints) for an input field in the backend |
| **Prefix** | snippet and template prefix: `{module}`, `proc.{name}`, `ext.{name}` (lower case) |

## The component contract

Every component has the same skeleton:

```
{Name}/
├── {Name}.php                 module: dataset class(es); processor: MediaProcessor; extension: optional
├── install/
│   ├── settings.php           profile (mandatory)
│   ├── Installer.php          optional: {Namespace}\Install\Installer
│   ├── schema.sql             optional: tables – CREATE TABLE IF NOT EXISTS
│   ├── seed.sql               optional: base data
│   └── migrations/            {version}.sql and {version}.php
├── lang/  backend/lang/       snippets
├── backend/                   backend scripts, html/, lang/
├── public/                    extensions only: js/, css/ (published)
└── docs/{language}/           optional: the component's manual (Markdown)
```

Namespaces: `Camindo\Modules\{Name}`, `Camindo\Processors\{Name}`,
`Camindo\Extensions\{Name}` (core: `Camindo\Core\…`). Directory name = class name =
StudlyCase (`^[A-Z][A-Za-z0-9]*$`). An add-on module must not have the name of a core
module; an add-on processor, however, overrides a core processor of the same name.

### `install/settings.php`

```php
<?php
return [
    'version'     => '1.2.0',                  // SemVer – drives the migrations
    'type'        => 'blog',
    'label'       => '#blog.module_label',
    'description' => '#blog.module_description',
    'icon'        => 'article',
    'listed'      => true,                     // module: in permission management and start page
    'position'    => 50,                       // sort order (higher = further down)
    'requires'    => [
        'core'       => '^9.0',                // constraint against CAMINDO_VERSION
        'php'        => ['gd', 'intl'],        // PHP extensions
        'bin'        => ['ffmpeg'],            // paths.* in .camindo.php, executable
        'components' => ['Record' => '^9.0'],  // other components, must be active
    ],
    'tables'      => ['blog', 'blog_tag'],     // the component's tables (for --purge)
];
```

Only keys from a fixed list are taken over (`ComponentManager::SETTINGS_KEYS`):

| Key | For | Meaning |
|---|---|---|
| `version`, `type`, `label`, `description`, `icon`, `position`, `requires`, `tables` | all | see above |
| `listed` | module | subject to permission management, gets a tile |
| `access` | module | extra rights `name => label snippet` ([11](11-permissions.md#extra-rights)) |
| `script_access` | module | `script.php => extra_right` |
| `shared` | module | scripts other modules may call as a lookup without module access |
| `personal` | module | scripts every user may call without module access (own settings) |
| `entries` | module | entry scripts of the tile; the first permitted one wins |
| `button` | module | `false`: no tile (e.g. widget only) |
| `superuser` | module | `true`: superusers only, regardless of user group rights |
| `widget` | module | start page widget ([05](05-backend-development.md#start-page-widgets)) |
| `api` | module | API classes relative to the module namespace, e.g. `['BlogApi']` ([16](16-api.md#custom-api-methods)) |
| `commands`, `services` | module | own `(CMS:…)` commands ([06](06-template-compiler.md#custom-commands-from-add-on-modules)) |
| `assets` | module, extension | JS and CSS files ([assets](#assets-and-endpoints)); for modules on every backend page |
| `display_types`, `field_types`, `options_provider` | extension | see below |

`settings.php` is evaluated via `include` – a component is code the operator installs.

### `install/Installer.php`

If installation needs more than `schema.sql`/`seed.sql`, implement
`Camindo\Core\Install\ComponentInstallerInterface` in the class
`{Namespace}\Install\Installer` (it is loaded via `require_once`, not the autoloader):

```php
interface ComponentInstallerInterface {
    public function install(InstallContext $ctx): void;                     // after schema.sql/seed.sql
    public function migrate(string $from, string $to, InstallContext $ctx): void;
    public function activate(InstallContext $ctx): void;
    public function deactivate(InstallContext $ctx): void;
    public function uninstall(InstallContext $ctx, bool $purge): void;     // purge: delete data
}
```

`InstallContext` passes on `db`, `config`, `system`, the component's name and path and
offers `tableExists()`, `columnExists()`, `indexExists()`, `runSqlFile()` and `log()`
(console or backend).

## Life cycle

```
composer require        component install      component activate
 ──────────▶ discovered ──────────▶ inactive ◀────────────────▶ active
                                       ▲                          │ composer update
                                       │ component update         ▼
                                       └──────────────────── update pending
```

| State | Recognisable by | Meaning |
|---|---|---|
| **discovered** | directory present, no row in `component` (or `version = ''`) | nothing installed, not loaded |
| **inactive** | `status = 0` | installed but not loaded |
| **active** | `status = 1` | normal operation |
| **update pending** | file version > installed version | treated as inactive at runtime until `component update` has run |
| **orphaned** | row present, directory missing | after `composer remove` without `uninstall`; can be cleaned up |
| **error** | `requires` not met, name conflict | cannot be installed or activated, reason in the list |

`Camindo\Core\ComponentManager` bundles everything without backend dependencies – the
command line and the Component module call the same methods:

| Method | Does |
|---|---|
| `discover()` / `status()` | read directories, check `requires`, reconcile with `component` – read only |
| `refresh()` | register new components, update file versions, write runtime lists |
| `install(name)` | backup, `schema.sql`, `seed.sql`, `Installer::install()`, create the row, component caches, assets |
| `activate` / `deactivate` | set status, `Installer::(de)activate()`, runtime lists, group cache |
| `update(name)` | backup, migrations from the installed to the file version, `Installer::migrate()`, caches |
| `uninstall(name, purge)` | deactivate, `Installer::uninstall()`, with `purge` drop the tables from `tables`, remove row, caches and assets |

Procedures and commands: [09](09-installation-and-update.md#add-on-components).

### Runtime lists

`var/config/components/{modules,processors,extensions}.php` contain **only the
components active at runtime** (core + active) with `path`, `namespace`, `version` and
the accepted `settings.php` keys. What is missing does not exist at runtime. The lists
are read via `include` per request, never from the database.
`Cms::componentPath($kind, $name)` resolves through them; `Backend::backendPath()`,
`ProcessorFactory`, `Access` and the start page use them. Every time the lists are
written, `ComponentManager` rebuilds the search index of the manuals – so the manuals of
components come and go with them.

### A component's manual

Every component may bring its own manual. The module Docs adds it to the backend's
manuals (selection, search, keyword index, print view, previous/next) as soon as the
component is active:

```
{Name}/docs/{language}/01-{kind}-{name}.md     start page (mandatory), e.g. Modules/Blog/docs/en_US/01-module-blog.md
{Name}/docs/{language}/*.md                    further chapters, also in subdirectories
{Name}/docs/{language}/_/*.svg                 icons (Material Symbols), as in Core/docs
```

| Rule | |
|---|---|
| `{kind}` | `module`, `extension` or `processor` |
| `{name}` | component name in lower case (`TinyMce` → `tinymce`) |
| Languages | the same codes as `Core/docs/` (`de_DE`, `en_US` …); without a start page in a language, the manual does not exist there |
| Title | the first heading of the start page – in the selection and the overview |
| Order | start page, then its links in lists and tables, remaining pages by file name – number the chapters (`02-…`, `03-…`) |
| Links | relative within the manual (`./02-commands.md#section`); links into the core manuals or those of other components stay plain text – name manual and chapter in the text |
| Style | like the core manuals |
| Code examples | blocks with CMS commands as ```` ```cms ```` – the Docs module colours them like the template editor; syntax descriptions that are not a valid command (`(CMS:TEXT field [parameter=value …])`) as ```` ```cmscode ```` – entirely purple, not parsed |

Internally the manual is called `{kind}-{name}` (e.g. `module-blog`); its pages appear
under this name in the API as well (`docs.search`, `docs.read`, parameter `book`). The
component's README stays short: purpose, installation, license and, per language, a link
to the start page in `docs/` (see [10](10-publishing-components.md#one-repository-per-component)).

## Extensions

An extension adds JavaScript behaviour to an input field of a given display type in the
backend. It is assigned **to a field** (in its definition), not globally to a display
type – one area can have a `textarea` with TinyMCE and another one with Markdown. A
field has at most one extension.

| Core extension | Display / field type | Purpose |
|---|---|---|
| `Calendar` | `text` / `datetime`, `timestampl`, `time` | native date/time picker |
| `Datepicker` | `text` / `datetime`, `timestampl` | calendar in the user's locale |
| `MarkdownVisual` | `textarea` / `text`, `chars` | Markdown editor that edits the formatted view |
| `IconPicker` | `text` | choose a Material symbol |
| `Snippets` | `text` | snippet search for label fields |
| `Password` | `password` | strength meter and password policy |

Add-on extension: `TinyMce` (WYSIWYG editor, separate package, GPL like TinyMCE).

### Profile

```
Extensions/TinyMce/
├── composer.json                 type camindo-extension, require tinymce/tinymce
├── install/settings.php
├── lang/  backend/lang/          prefix ext.tinymce
├── backend/
│   ├── form.php                  options in the field form
│   ├── info.php                  options in the field detail view
│   └── *.php                     optional endpoints
└── public/
    ├── js/tinymce.ext.js         registration with cApp.extensions
    └── css/tinymce.ext.css
```

```php
return [
    'version'       => '1.0.0',
    'type'          => 'editor',
    'label'         => '#ext.tinymce.label',
    'description'   => '#ext.tinymce.description',
    'display_types' => ['textarea'],             // where the extension can be selected
    'field_types'   => ['text', 'chars'],        // optional: additionally by field type
    'assets' => [
        'js'   => ['vendor:tinymce/tinymce/tinymce.min.js', 'js/tinymce.ext.js'],
        'css'  => ['css/tinymce.ext.css'],
        'dirs' => ['vendor:tinymce/tinymce/skins'],   // whole directories, recursive
    ],
    'options_provider' => 'TinyMceOptions',      // optional, see below
    'requires' => ['core' => '^9.0'],
];
```

`vendor:` paths point into `vendor/`, all others into the extension's `public/`.

### Assignment to a field

`field.details` gets `extension` and `extension_options`:

```json
{ "extension": "TinyMce", "extension_options": { "toolbar": "basic", "height": "400" } }
```

The field form only offers active extensions matching display and field type and
includes the options block from `{Extension}/backend/form.php` (with `$backend`,
`$cms`, `$options` and `$name` = `data[details][extension_options]`).
`Field::hookOnSave()` rejects an unsuitable or inactive extension. Meta fields get an
extension via `'extension' => 'Name'` in the column definition.

### Rendering and JavaScript API

`Backend::formField()` adds attributes to the input element in the edit form (not in
the search form) – no inline script per field:

```
<textarea … class="… c9-ext" data-c9-ext="tinymce" data-c9-ext-type="text"
          data-c9-ext-options='{"toolbar":"basic","height":"400"}'></textarea>
```

An `options_provider` (a class with `clientOptions(array $options): array`) can enrich
the options on the server first – `Password` delivers the effective rules and
translated texts this way.

`backend/index.php` outputs the published assets of all active extensions as the
manifest `cConfig.extensions`; they are only loaded once a field needs them. The
extension registers itself with three hooks:

```js
cApp.extensions.register('tinymce', {
    init(el, options)  { /* create the editor on el; on change cApp.forms.markChanged(el) */ },
    beforeSubmit(el)   { /* write the content back into el.value */ },
    destroy(el)        { /* remove the instance – the layer is replaced or closed */ }
});
```

`cApp.extensions` calls `init()` after every `cLoadContent`, `beforeSubmit()` before a
form is submitted and `destroy()` before a layer is replaced or closed.
`cApp.forms.markChanged(el)` reports changes to change tracking – editors do not fire
`input`/`change` on the hidden textarea. On the server, the value remains a normal
text field.

### Assets and endpoints

`ComponentManager` **copies** the files listed in `assets` to
`public/backend/ext/{name}/` (copy rather than symlink, so that `vendor/` does not
become reachable from the document root) and lists them in the manifest with
`?v={version}`.

A **module** may list `assets` as well (`['js' => […], 'css' => […]]`, relative to the
module's `public/`). They go to `public/backend/mod/{name}/`, and the backend includes
them on **every** page – stylesheets in the head, scripts before `backend-init.js`,
regardless of the user's module right (the server checks the rights). Such a script
initializes itself (`$(document).ready(…)`) and must not break anything without the
module right. This is meant for things that reach beyond the module, such as a keyboard
shortcut. `camindo.php refresh` republishes the assets of all running extensions and
modules – after changing the files, that is all it takes.

An extension's endpoints live in `backend/` and are reachable via
`/backendExtensions/{Name}/{script}`. They only require a session – extensions have no
module rights. An endpoint must therefore not deliver anything that would need an area
permission check.

### Project-specific files

Editors need project-specific style sheets. They live in `var/extensions/{name}/`
(`Backend::extensionVarDir()`), not in the extension, which is replaced on updates.
`form.php` offers the files for selection (`Backend::extensionVarFiles()`), and a
separate endpoint delivers them because `var/` is not in the document root.

### In the frontend

An extension whose script does not depend on `cApp` can also be used in the frontend.
`Password` shows the pattern: the script equips `input[data-c9-password]` on load by
itself; the attribute value comes from `(CMS:USER_PASSWORD_RULES)`.

### Markdown syntax: editor and renderer

`MarkdownVisual` ships no library. The script (`markdownvisual.ext.js`) contains its
own Markdown parser, the rendering into the editor and a serializer DOM → Markdown.
Pasted Markdown and HTML go through the same block list; whatever it cannot represent
(images, strikethrough, colours) is dropped. Text in the editor is escaped, link
targets are limited to `http(s)`, `mailto`, `tel`, `ftp` and relative addresses.

On top of common Markdown there are `++underline++`, alignment containers
`::: left|center|right … :::`, `<br>` in table cells and the text direction as Unicode
isolates: RLI `U+2067` (right to left) or LRI `U+2066` (left to right) up to PDI
`U+2069`, in the editor `bdi[dir]` (button `dir`; `&#x2067;` … `&#x2069;` is recognised
too; a character without its counterpart stays text). Emphasis does not cross the
boundary of an isolate. The renderer passes the characters through unchanged – browsers
apply them. In the frontend,
`Camindo\Core\Markdown` renders it (Parsedown in safe mode, extended by exactly this
syntax, plus entities and the escapes `\:`, `\&`, `\=`, `\<`) – as a template
command `(CMS:TEXT field markdown=true)`. Script and class must know the same syntax;
whoever changes one changes both. The self-test for it is in
`cron/check_template_params.php`.

The UI texts come from the `options_provider` `Camindo\Core\MarkdownVisualOptions`,
taken from the snippets `ext.markdownvisual.ui_*`; the script replaces missing texts by
English defaults.
