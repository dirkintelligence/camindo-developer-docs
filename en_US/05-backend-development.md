# 05 – Backend development

How a backend page is built – from the file to the HTML.

## Structure of a module

```
Modules/Blog/                    (core modules: Core/Modules/{Name}/)
├── Blog.php                     dataset class (table blog)
├── BlogBackend.php              backend logic: list(), info(), edit(), search()
├── install/
│   ├── settings.php             module profile (chapter 08)
│   ├── views.php                views for tools such as the AI assistant (below)
│   ├── schema.sql               tables (CREATE TABLE IF NOT EXISTS)
│   └── migrations/              version steps
├── lang/{locale}.php            snippets for backend AND frontend   (prefix 'blog')
└── backend/
    ├── index.php                entry point of the module tile
    ├── blog_list.php            view scripts, three lines each
    ├── blog_info.php
    ├── blog_edit.php
    ├── blog_search.php
    ├── html/*.html              HTML templates                      (prefix 'blog')
    └── lang/{locale}.php        backend-only snippets               (prefix 'blog')
```

Namespaces: core modules `Camindo\Core\Modules\{Name}`, add-on modules
`Camindo\Modules\{Name}`. Directory name = class name = StudlyCase.

### The pattern: view script → backend class → dataset

The scripts in `backend/` are deliberately **three lines long**: get the backend class
from the container, call a method.

```php
<?php
// Modules/Blog/backend/blog_info.php
// @var Camindo\Core\ContainerInterface $app
use Camindo\Modules\Blog\BlogBackend;

$app->get(BlogBackend::class)->info();
```

| Layer | File | Role |
|---|---|---|
| Routing | `backend/blog_edit.php` | entry point, no code |
| Controller + presentation | `BlogBackend.php` | reads the request, orchestrates, outputs HTML |
| Model | `Blog.php` (`extends Dataset`) | schema, validation, persistence, hooks |
| Templates | `backend/html/*.html` | HTML with `{placeholders}` |
| Texts | `lang/`, `backend/lang/` | snippets |

Large modules split the backend class – `Record`, for instance, into
`RecordBackendEdit`, `RecordBackendList`, `RecordBackendInfo`, `RecordBackendSearch` …

Workflows that several interfaces need do not belong in the backend class but in a
**service class** without request and without output (returns `null`/`false`,
messages in `Cms::$errors`). Example `RecordService`:

| Method | Purpose |
|---|---|
| `loadForAction($id, $action)` | load a productive record, user group rights for `read`, `edit`, `delete`, `copy`, `move` |
| `loadForChange($id, $action, $areaId, $unlock)` | plus area, foreign lock and the user's own still open form |
| `lockInfo()`, `lockMessage()`, `ownOpenCopy()` | edit locks |
| `checkOut($data, $copy, $dependents)` | working copy including embedded content, assigned to the session |
| `saveFields()`, `checkIn($tmpId, $fields, $data)` | check-in including embedded content in one transaction |
| `updateOne(…)` | change without a form in one step (batch processing, API) |
| `delete($id, $withChilds)` | soft delete, large subtrees in the background |
| `urlFilter($input)` | search field `_url` (id, slug, URL) |

The edit form, moving, batch processing, the import and the API ([16](16-api.md))
use it. Also shared: `RecordPresenter` – columns and values of the record list
(`listFields()`, `listValue()`), the same in the backend list, the lookup preview and
`record.search`. A working copy of the
session user does not lock the user out – but `checkOut()` discards it. Code that
changes a record without the form therefore checks with `loadForChange()` whether the
same user still has it open in a form.

### The four standard pages

| Method | Script | Purpose |
|---|---|---|
| `list()` | `{module}_list.php` | result list with sorting and pagination |
| `info()` | `{module}_info.php` | detail view |
| `edit()` | `{module}_edit.php` | create, edit, copy, delete |
| `search()` | `{module}_search.php` | search form and search |

`edit()` is a small state machine controlled by `$_REQUEST['function']`:

```php
public function edit(): void
{
    $this->id       = (int)($_REQUEST['id'] ?? 0);
    $this->function = (string)($_REQUEST['function'] ?? '') ?: 'edit';
    $this->data     = (array)($_REQUEST['data'] ?? []);

    $this->backend->setNaviLabel('#button_' . $this->function);
    echo $this->backend->template('container_header');

    if ($this->function === 'create')                           $this->_editCreate();
    if (in_array($this->function, ['edit', 'copy', 'delete']))  $this->_editLoad();
    echo $this->editHeadline();
    if ($this->function === 'save')                             $this->_editSave();
    if ($this->function === 'delete_confirm')                   $this->_editDeleteConfirm();
    if (in_array($this->function, ['create', 'edit', 'copy', 'errors'])) $this->_editForm();
    if ($this->function === 'delete')                           $this->_editDeleteForm();

    echo $this->backend->template('container_footer');
}
```

| `function` | Meaning |
|---|---|
| `create` | empty form with defaults from `init()` |
| `edit` (default) | load and show the record |
| `copy` | load, treat as a new record |
| `save` | save; on errors continue with `errors` |
| `errors` | show the form again with input and errors |
| `delete` / `delete_confirm` | confirmation / perform the deletion |

Private helper methods carry the public method's name as a prefix:
`edit()` → `_editSave()`, `_editForm()`.

**Write actions only via POST.** Every endpoint with side effects checks
`if (!$this->backend->requirePost()) { echo $access->deniedHtml(); return; }`.
`Backend::backendPath()` checks the CSRF token of every POST automatically
([14](14-security-web.md#csrf)).

## Checking permissions

`Backend::backendPath()` checks module access for every script. Beyond that, the
backend class asks `Backend::access()`:

```php
$access = $this->backend->access();
if (!$access->areaRead($areaId)) {                 // group's read right on the area
    echo $access->deniedHtml();
    echo $this->backend->template('container_footer');
    return;
}
if ($access->areaWrite($areaId, $ownerId)) { … }   // edit button only with the right
if ($access->moduleRight('Blog', 'publish')) { … } // extra right of the module
```

The data layer checks independently (`Dataset::accessCheck()`); the UI should simply
not offer what would fail. Context menus hide entries per row via
`data-ctx-hide="edit,delete"` (row) and `data-ctx="edit"` (entry). Details:
[11](11-permissions.md).

## Snippets – multilingual texts

### Location

| Directory | Prefix | applies to |
|---|---|---|
| `Core/lang/{locale}.php` | – | backend and frontend |
| `Core/backend/lang/{locale}.php` | – | backend only |
| `{Module}/lang/{locale}.php` | `{module}` (lower case) | backend and frontend |
| `{Module}/backend/lang/{locale}.php` | `{module}` | backend only |
| `{Processor}/lang/…`, `…/backend/lang/…` | `proc.{name}` | likewise |
| `{Extension}/lang/…`, `…/backend/lang/…` | `ext.{name}` | likewise |
| `var/custom/snippets/{instance_id}/lang/{locale}.php` | `custom.{instance_id}` | snippets of an instance (Snippets module) |

`{locale}` is a backend language code in the form `de_DE`, `en_US` (13 languages in the
core). `en_US.php` is mandatory and serves as the fallback. Only texts that can appear
in the frontend belong in `lang/` (error messages, labels of meta fields); a frontend
request never loads a `backend/lang` file.

```php
<?php
return [
    'field_blog_title'   => 'Title',                 // field_{table}_{column}
    'message_blog_saved' => 'The entry has been saved.',
    'alert_blog_delete'  => 'Really delete the entry?',
    'count_entries'      => '{n, plural, one{# entry} other{# entries}}',   // ICU
];
```

ICU formats (`plural`, `select`) are only evaluated if the **whole** snippet starts with
them.

### Usage

```php
$cms->snippet('blog.field_blog_title');                 // module Blog
$cms->snippet('proc.ffmpegvideo.headline_output');      // processor
$cms->snippet('button_back');                           // core
$cms->snippet('blog.count_entries', ['n' => 3]);        // with parameters
```

Resolution uses the current language; if a key is missing, the English text was
already filled in when the cache was built; if it is missing there too, the key itself
appears – so every gap is visible during development.

**Labels with `#`.** Schema definitions contain `'label' => '#blog.field_blog_title'`.
`Cms::textOrSnippet()` resolves a leading `#` as a snippet; everything else stays plain
text. Input fields for labels carry `'extension' => 'Snippets'` in the schema – a
search across all snippets. For common field and area names the core ships `label_*`
snippets (`Core/backend/lang`, e.g. `#label_customers`, `#label_images`, `#label_url`)
– editors need no set of their own for them.

**Snippets per instance.** The Snippets module maintains one set per instance
(`custom.{instance_id}`), written exclusively via `var_export()`. In the frontend, the
short form `custom.{key}` is enough – `Cms::snippet()` adds the instance itself; in the
backend, the explicit form `custom.31.welcome` is required. The frontend chooses the
snippet language via `language.backend_code`.

## HTML templates

| Directory | Prefix |
|---|---|
| `Core/backend/html/*.html` | – |
| `{Module}/backend/html/*.html` | `{module}` |
| `{Processor}/backend/html/*.html` | `proc.{name}` |
| `{Extension}/backend/html/*.html` | `ext.{name}` |

```html
<!-- Modules/Blog/backend/html/blog_table_row.html -->
<tr class="c9-ctx-row" data-id="{data_id}" role="button" tabindex="0"
    aria-label="{#blog.speak_info_blog}">
  <td class="text-primary fw-bold">{data_title}</td>
  <td>{data_date}</td>
</tr>
```

```php
echo $backend->template('blog.blog_table_row', [
    '{data_id}'    => $id,
    '{data_title}' => htmlspecialchars($title),
]);
```

* Name = `{prefix}.{file}` without `.html`; without a dot, the unprefixed core template.
* `strtr()` replaces `{simple}` placeholders from the array; `{#name}` is replaced
  automatically by the snippet of that name.
* Values for placeholders inside an attribute value `"…"` get `"` as `&quot;`
  (`attributePlaceholders()`). Text content is escaped by the caller.
* `onclick="…"`, other `on…` attributes, `href="javascript:…"` and `<script>` in a
  template are marked for the script policy and work as usual
  ([14](14-security-web.md#script-policy-in-the-backend)).

## Caches – rebuild after every change

Snippets and HTML templates are served exclusively from caches
(`var/cache/lang/{locale}/{prefix}[.backend].php`, `var/cache/html/cache[.{prefix}].php`).
Without a rebuild, the old version keeps appearing:

```
php Core/cron/camindo.php cache all                     snippets, HTML, templates, manuals
php Core/cron/camindo.php cache snippets --prefix=blog  a single component
php Core/cron/camindo.php cache html --prefix=blog
```

In the backend: Tools → Developer. The builders are `SnippetCache` and `HtmlCache`.
After changing `install/settings.php`: `php Core/cron/camindo.php refresh` or "Refresh"
in the Component module.

## The `Backend` class

`Camindo\Core\Backend` provides all UI building blocks. It is stateful: form, tabs,
buttons and info blocks are built step by step and retrieved at the end.

### Page frame, tabs, buttons

```php
$backend->setNaviLabel('#button_edit');           // label in the path
echo $backend->template('container_header');
echo $backend->headline('#blog.headline');

$backend->tabBarReset();
$backend->tabAdd('#tab_data', "cApp.layers.replace('Blog/blog_edit.php?id=$id')", true);
echo $backend->tabBar();

$backend->buttonBarReset();
$backend->buttonAdd('button_save', 'cApp.forms.submit()', '#button_save');
$backend->buttonAdd('button_back', 'cApp.layers.close()', '', alignRight: true);
echo $backend->buttonBar();

echo $backend->template('container_footer');
```

### Forms

```php
$backend->formInit('blog.blog_edit_form');         // template with {data_*} placeholders
$backend->formHidden('function', 'save');
$backend->formField($blog, [
    'name'  => 'title',
    'value' => $data['title'],
    'id'    => $id,
    // optional: colDef, varName, display_type, options, template, placeholder,
    //           append, selectPlease, lookupOptions, attr, fullWidth, langId, langSuffix,
    //           textLang, dir
]);
$backend->formStatic($blog, 'created_at', $html);
echo $backend->form('edit_form_buttons', [], removePlaceholders: true);
```

`formField()` replaces the placeholder `{data_{name}}`; with `'varName' => 'filter'`
it becomes `{filter_{name}}` – that is how search forms are built. Multilingual fields
get `'langId'` and `'langSuffix'`: the field name becomes `data[name][{language_id}]`.

***Text direction:*** `'textLang'` (language code or locale of the content, default with
`langId`: `langSuffix`) sets `dir` according to `Cms::isRtl()` – an `ar_SA` field is
right-to-left even in a German backend. This applies to texts (`text`, `chars`,
`encrypted` as `text`/`textarea`, also with an editor extension); email, phone, password
and fields with the flag `name` are always `ltr`, everything else inherits the backend's
direction (`<html dir>`). `'dir' => 'ltr'|'rtl'|'auto'` forces a direction. The same rule
is available as `Backend::fieldDirection($colDef, $lang)` for your own output (info view,
read-only languages). Editor extensions take over the field's direction
(`getComputedStyle(el).direction`).

`formField()` checks strictly against two lists in `Backend`: `$displayTypes[$type]`
(which `display_type` fits which field type) and `$validationFlags[$type]`. If
something does not fit, an error message appears instead of the field – schema errors
become visible immediately during development.

| Type | allowed `display_type` |
|---|---|
| `chars`, `encrypted` | `text`, `textarea`, `email`, `tel`, `select`, `radio` |
| `text` | `text`, `textarea`, `select`, `radio` |
| `json`, `reltable` | `checkbox`, `multiselect`, `sortable` |
| `bit` | `switch`, `checkbox`, `radio` |
| `integer`, `currency`, `float`, `decimal`, `datetime`, `time` | `text`, `select`, `radio` |
| `timestampl`, `fulltext` | `text` |
| `id` | `select`, `radio`, `lookup` |
| `relation`, `pointer` | `lookup`, `select`, `multiselect` |
| `relation1`, `pointer1` | `lookup`, `select` |
| `link` | `lookup`, `embed` |
| `file` | `upload`, `thumbnail` |
| `password` | `password` |

`sortable` is the sortable form of `checkbox` (arrow buttons, `backend-smartrows.js`).

**Empty fields.** `serializeFiles()` drops empty values, and a switched-off switch is
not in the POST at all. The record form therefore carries the hidden field `shown`
listing all rendered fields; anything listed there but missing in `data[]` counts as
cleared. Your own forms that must distinguish "empty" from "not shown" need the same
mechanism. For the same reason, a `<button type="submit" name="x" value="y">` transmits
nothing – actions belong in an `<input type="hidden">` set by a handler.

### Info views, lists, references

```php
$backend->infoReset();
$backend->infoAdd($blog, 'title', $data['title']);
$backend->infoStatic('Label', $html);
echo $backend->infoTable();

echo $backend->paginationTop('#blog.headline_list');
echo $backend->tableHeadColumn($colDef, $f, 'title');   // sortable column header
echo $backend->paginationBottom();

echo $backend->teaser('admin', $adminId, linked: true);  // compact reference with icon
```

### Further helpers

```php
$backend->balloon('success', $cms->snippet('blog.message_blog_saved'));   // message in the browser
$backend->script("$('#x').focus();");               // script in the Ajax response (with nonce)
echo '<a href="#"' . $backend->on('js', "cApp.layers.add('Blog/blog_info.php?id=$id')") . '>…</a>';
echo '<script' . $backend->nonceAttr() . '>…</script>';
$backend->jsString($value);                         // safe JS string literal
$backend->sanitizePath($path);
$backend->backendValue($value, $colDef, 'short');   // prepare a value for display
$backend->isSuperuser();
```

## JavaScript API (`window.cApp`)

| Namespace | Purpose |
|---|---|
| `cApp.system` | spinner, messages, CSRF header |
| `cApp.forms` | change tracking (`markChanged(el)`), Ajax submit, `updateElement()` |
| `cApp.ui` | lookups, tabs, QR codes, copying |
| `cApp.layers` | layer stack and hash routing ([01](01-architecture.md#layers-and-hash-routing-in-the-backend)) |
| `cApp.aside` | side panel (`open`, `openPost`, `close`) |
| `cApp.ctxMenu` | context menus (`.c9-ctx-row`; entries with `data-method="post"` use POST) |
| `cApp.polling` | background processes |
| `cApp.extensions` | extensions of input fields ([08](08-components-and-extensions.md#extensions)) |
| `cApp.handlers` | binding of marked event handlers |

All Ajax calls go through `$(el).cLoadContent(options)`:

```js
$('#aside-content').cLoadContent({
    url: 'Blog/blog_edit.php?id=42',
    method: 'POST',
    params: $('#form').serializeFiles(),
    callback: () => { /* … */ }
});
```

`serializeFiles()` builds a `FormData`, drops empty text fields and unused file inputs
and checks the total size against `cConfig.max_upload_bytes`.

**Script policy rules** (in detail in [14](14-security-web.md#script-policy-in-the-backend)):

* Handlers (`onclick`) only in HTML templates or via `$backend->on()`, never raw in an
  `echo`.
* Your own `<script>` blocks only with `$backend->nonceAttr()` or `$backend->script()`.
* No `eval`, `new Function`, `setTimeout('string')`; no HTML with handlers via
  `innerHTML` – listeners belong on elements (`$(document).on('click', '.class', …)`).

## Start page widgets

The start page is a grid of four slots; slot 1 holds the module tiles, slots 2–4 are
filled by each user in their settings (`admin.settings.widgets`, a list of `module`
and `position`). A module offers a widget in its `install/settings.php`:

```php
'widget' => [
    'class'  => 'BlogBackend',        // relative to the module namespace
    'method' => 'widget',             // default 'widget'
    'label'  => '#blog.widget_label', // default '#{module}.module_label'
    'icon'   => 'article',            // Material symbol, optional
],
```

The class name is deliberately relative – a `settings.php` can only instantiate classes
of its own module. The method takes no parameters and **returns the HTML**; the card
around it comes from `Widgets`. An exception ends up in
`var/log/{date}_widget-errors.log`, the start page keeps working. Only users who may
use the module see its widget.

## Blueprints for new areas

`Core/Modules/Area/blueprints/{record|profile|ugc}/*.xml` appear as templates when an
area is created. A blueprint is an XML export file with exactly one `<area>` including
`<fields>` and no records; `RecordImport::importBlueprint()` creates the fields. Copying
an area uses the same path.

## Import and export

The Record module exports records as **XML** (backup format: areas, fields, records in
all languages, links, files, templates, converters), **CSV** (one row per record,
sub-fields in dot notation) and **DOCX** (for translation), and imports the same
formats. Larger amounts run as a background process (`Core/cron/record_export.php`,
`record_import.php`); the result arrives as an internal message.

For **template sets** meant to be transferred to another database, the export (option
"Optimise templates for import into another database") marks references in the
parameters of the `(CMS:…)` commands:

```
(CMS:INCLUDE <!-- CMS:REPLACE template_name=header -->)
(CMS:SELECT filter[area]=<!-- CMS:REPLACE area_name=article --> …)
```

Which parameters carry references is declared per parameter in `commands.php` with
`'ref'` (`template`, `area`, `record`). The import replaces the markers with the names
and ids of the target database. Hence a rule for portable templates: **references
belong as literals in the command parameters** (`area.template`, `filter[area]=name`) –
the import does not rewrite names inside a template's PHP code.

Custom snippets travel along: `<snippets>` contains the snippets referenced by labels
(`#custom.{id}.{key}`) and – with contents – those used by the exported templates
(`Snippets::templateRefs()`: `custom.{key}`, including composed keys like
`"custom.status_{$x}"` → all keys with that prefix). The import adds them to the set of
the target instance (`Snippets::merge()`, existing keys stay); the template code stays
unchanged because the short form refers to the set of the running instance.

Batch processing in the record list (`RecordBatch`) edits, moves or deletes many
records – each one through the same path as the form
(`RecordService::updateOne()`: check-out, `save()`, check-in), optionally as a simulation. Mode `fulltext` only rebuilds the full text
(`refreshFulltext()`, without check-out and without audit entry) and requires module
access to `Area` (`RecordBatch::fulltextAllowed()`).

## New module – checklist

1. Directory `Modules/{Name}/` (add-on module) with `install/settings.php`:
   ```php
   <?php
   return [
       'version'     => '1.0.0',
       'type'        => 'blog',
       'label'       => '#blog.module_label',
       'description' => '#blog.module_description',
       'icon'        => 'article',
       'listed'      => true,
       'requires'    => ['core' => '^9.0'],
       // optional: position, access, shared, personal, script_access, entries,
       //           button, superuser, widget, commands, services, tables (chapter 08)
   ];
   ```
2. Table in `install/schema.sql` (`CREATE TABLE IF NOT EXISTS`, column `{table}_id`).
3. Dataset class with `hookDefinition()` ([03](03-dataset-api.md)).
4. `{Name}Backend.php` with `list()`, `info()`, `edit()`, `search()`.
5. View scripts in `backend/`, including `index.php`.
6. Snippets `lang/en_US.php` (mandatory) and further languages; `module_label`,
   `module_description`.
7. HTML templates in `backend/html/` – handlers only there or via `$backend->on()`.
   Plus `install/views.php`: the views a tool may open (below).
8. `php Core/cron/camindo.php component install {Name} --activate` – creates the
   tables, builds the component's caches and writes the runtime lists.
9. Assign the module (and any extra rights) to the user groups.

## View catalog (`install/views.php`)

Which view scripts a tool may open for the user as a layer – for example the assistant
of the add-on module Ai ([16](16-api.md#assistant-add-on-module-camindomodule-ai)) –
each component describes in `install/views.php`. `Camindo\Core\BackendViews` reads the
catalogs of the active modules, shows only what the user may open (`Access::script()`:
module access, `script_access`, personal scripts) and builds the address only from the
allowed parameters.

```php
<?php
return [
    'blog_list.php' => [
        'description' => 'List of all blog posts with status; starting point to create a post.',
    ],
    'blog_edit.php' => [
        'description' => 'Edit form of a post. function=create: new post; copy / delete with id.',
        'params'      => ['id' => 'int', 'function' => ['create', 'copy', 'delete']],
    ],
];
```

* Only scripts that change nothing when opened: lists, detail views, forms that act only
  when submitted. No actions (`*_run.php`, cache rebuilds …), no ajax scripts.
* The description is English and written for a language model: what the view shows,
  what it is for, what the parameters do.
* Parameter types: `'int'` (positive integer), `'string'` (up to 200 characters) or a
  list of allowed values; `BackendViews::url()` drops any other parameter.
* No `refresh` needed – the file is read when required.

## Testing without a browser

Backend scripts can be run on the command line – without logging in, with a user's
rights:

```php
<?php
define('CAMINDO_STARTED', true);
define('CAMINDO_BACKEND', true);
require '/path/to/project/load_camindo.php';

$app->get(\Camindo\Core\Session::class)->setCliAdmin($adminId);   // withAccess: true = load group rights
$_REQUEST = $_POST = ['id' => 5, 'function' => 'save', 'data' => [/* … */]];
require '/path/to/project/Modules/Blog/backend/blog_edit.php';
```

`setCliAdmin()` only works on the command line. The script is included directly, not
via `Backend::backendPath()` (which requires a session). For HTTP tests,
`Session::create($adminId)` creates a real session (cookie `sessid`; the CSRF token
comes from `Session::csrfToken()`); remove the session afterwards. Rendering a record
form this way creates a working copy – discard it with `Record::cancel()`.
