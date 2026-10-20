# 06 – Template compiler

Frontend templates are written in the command language `(CMS:…)`, **compiled to PHP**
on save and stored as files. At runtime, plain PHP is executed – no parser, no
template engine in the request. The meaning of the individual commands for template
programmers is described in the template programmers' manual; this chapter describes
what lies behind them and how to add your own commands.

## The pipeline

```
source     var/templates/{instance_id}/{area}/{name}.cms.php          ← leads
   │
   ├── backend: the form loads the file, saving writes it back
   └── file changed/created/deleted by hand: TemplateFiles::sync() (every minute via cron)
          │
          └── Template::save()
                 ├── hookOnSave()  → TemplateParserTrait::parse($code, $areaId)
                 │        ├── phpCode    → only as a file, NEVER into the database
                 │        ├── styledCode → template.styled_code  (display)
                 │        ├── errors     → template.error_text   (errors and notes)
                 │        └── code       → template.code         (copy, for searching only)
                 └── hookOnSaved() → var/php/{area_id}/{name}.php

frontend request → Page → include var/php/{area_id}/{name}.php   (inside an IIFE)
```

| Location | Content | Role |
|---|---|---|
| `var/templates/{instance_id}/{area}/{name}.cms.php` | source | **leads** – what is edited |
| `var/php/{area_id}/{name}.php` | compiled PHP | is executed |
| `template.code` / `styled_code` / `error_text` | copy, highlighted version, messages | search and display in the backend |

**Code never travels from the database to the file system.** Compilation only uses the
files; if a file is missing, the template loads with an empty source. Whoever can write
to the database cannot execute PHP code this way.

### Synchronising files and database

`TemplateFiles::sync()` reconciles the file system with the database: new directory
below an instance → new template area; new file → new template; changed file → save
and compile again; deleted file → delete the template; deleted directory → area to the
trash. Names must match `^[a-z][a-z0-9_]+$`. If a whole instance directory is missing,
`sync()` deletes nothing there. Renaming in the file system means delete + create –
assignments are lost; better rename in the backend. Conversely, the backend creates,
renames and moves files and directories.

`php Core/cron/camindo.php cache templates` recompiles all files
(`Template::rebuildFiles()`) and synchronises afterwards.

Syntax errors do **not** block saving – they are stored in `error_text` and shown in
the backend with line numbers.

### Referring to templates

`template.name` is only unique **within its template area**. Wherever a template is
named (`CMS:INCLUDE`, `resulttemplate`, `successtemplate` …, template part of the URL):

| Notation | Meaning |
|---|---|
| `my_template` | the template of this name – there may only be one within the instance's scope |
| `area.my_template` | the template in template area `area` |

`Cms::templateFile()` resolves the name (once per reference and request) and aborts
with a `RuntimeException` for an ambiguous or unknown name.

## Syntax

```
(CMS:COMMAND value key=value key[sub]=value key[]=value …)
```

* The command name is case-insensitive.
* Parameters are separated by whitespace; line breaks are allowed.
* Values in `"…"` or `'…'` are taken as they are, including PHP interpolation (`$var`,
  `{$expression}`); everything else is put in double quotes.
* Sub-keys in square brackets; the same key path used several times becomes an array.
* Between key and value there is either `=` or a comparison operator (`==`, `!=`, `!`,
  `!!`, `<`, `>`, `<=`, `>=`, `<<`, `>>`; the longest match wins, so `key=>5` is still
  `=` with the value `>5`). The compiler prepends the operator to the value in its
  two-character form (`<` → `<<`, `>` → `>>`, `!` → `!=`):
  `filter[a] < "{$x}"` becomes `$_a["filter"]["a"] = '<<' . "{$x}";`.
  `Cms::extractOperator()` strips exactly this prefix, the rest is the plain value –
  `filter[a] < ">2"` compares with the text `>2`. This is only allowed for
  parameters with `'operator' => true` in the command definition (`filter` of
  `CMS:SELECT`, `CMS:ZIP`, `CMS:USER_AUTH`, `CMS:USER_PROTECT`); for all others the
  compiler reports an error and leaves the parameter out. Keys with a PHP part cannot
  be checked and always get the prefix.
* `<?php … ?>` blocks remain unchanged. Commands are recognised inside PHP comments as
  well – `(CMS:…)` does not belong there.

```
(CMS:SELECT filter[area]=article resulttemplate=article_list)
```

becomes

```php
<?php /* --- CMS:SELECT --- */ $_a = [];
$_a["filter"]["area"] = "article";
$_a["resulttemplate"] = "article_list";
$this->db->pushResult(); $this->cms->pushRow($cms ?? null);
$_r = $this->cms->cmsSelect($_a);
if (!empty($_r)) {
    $this->cms->includeDepth++;
    require $this->cms->templatePath . "/{$_r}.php";
    $this->cms->includeDepth--;
}
$this->db->popResult(); $cms = $this->cms->popRow(); ?>
```

(The generator keeps **lines aligned**: line numbers in the compiled PHP match those in
the source.)

## Command definitions (`Core/Modules/Template/commands.php`)

Every command is an array entry:

```php
'select' => [
    'method'     => 'cms.cmsSelect',            // {object}.{method}; without a dot: cms
    'php_prefix' => 'push_result',              // snippet before the call
    'php_suffix' => ['include', 'pop_result', 'select_count'],  // … and after it
    'buffer_set' => true,                       // set=… captures the whole output
    'note'       => '#template.cmd_select',     // language snippet: note on the command
    'do'         => 'resulttemplate',           // parameter for a value without a key
    'oneof'      => 'filter,relation',          // at least one of them
    'params'     => [
        'resulttemplate' => [
            'mandatory'   => true,
            'subparam'    => 0,                 // 0 = no sub-key, 1, 2
            'default'     => '',
            'value'       => '#template.cmdval_template_name',  // expected value
            'description' => '#template.cmd_select__resulttemplate',
            'ref'         => 'template',        // reference: template | area | record
            'php_value'   => false,             // true: pass an unquoted $var with its type
            'operator'    => false,             // true: key < value allowed (filter)
        ],
    ],
],
```

`method` becomes `$_r = $this->{object}->{method}($_a);`; `$this` is the `Page`
instance. `'no_action' => true` generates no call – `CMS:RESULT_START`/`RESULT_END`
work purely through their snippets. `'ref'` is used by export/import (rewriting
references, [05](05-backend-development.md#import-and-export)) and by the editor (links
to the referenced template).

The parser does not evaluate `note`, `description`, `value` and `default`; they appear in
the command picker of the template input mask (tooltip, placeholder, lead text) and in the
VS Code extension. `Template::commandText()` reads them: a value of the form `#prefix.key`
is a language snippet, anything else is shown unchanged. The core texts are snippets in
`Modules/Template/backend/lang/{lang}.ini` (section “Command reference”), as plain text
without HTML entities:

| Key | Snippet |
|---|---|
| `note` | `#template.cmd_{command}` |
| `description` | `#template.cmd_{command}__{parameter}` |
| `value` | `#template.cmdval_{hint}` – shared vocabulary (“Text”, “Record ID or slug” …) |
| `default` | `#template.cmddef_{hint}` – only for descriptive defaults (“(current record)”) |

Pure syntax stays literal: value hints such as `true / false`, `asc / desc` or the list of
operators (the VS Code extension offers values separated by ` / ` as choices) and real
default values (`false`, `data`). Add-on modules may enter plain text or their own snippets.

Parameter values otherwise always become strings (`cacheparams=$p` → `"$p"`). With
`'php_value' => true` the parser takes an unquoted value that consists of a single
variable only – `$p`, `$cms['id']`, `$set["a"]["b"]`, `$obj->name` or the same in
`{…}` – as a PHP expression (`TemplateParserTrait::phpValueToken()`); arrays and
numbers arrive unchanged. Quoted values remain strings. Used by
`CMS:INCLUDE cacheparams`.

`'buffer_set' => true` gives the command the parameter `set` for its ***entire***
output including the snippets (list template, included template): the parser puts
`Cms::setBufferStart($_a['set'])` before it and `setBufferEnd()` after it – a stack in
`Cms`, because included templates overwrite `$_a`. Used by `CMS:SELECT`,
`CMS:SELECT_LANGVERSION` and `CMS:INCLUDE`.

| Object | Class | loaded |
|---|---|---|
| `cms` | `Cms` | always |
| `db` | `Database` | always |
| `pagination` | `Pagination` | on first access |
| `recordUser` | `Modules\Record\RecordUser` | on first access |
| `email` | `Email` | on first access |
| `zip` | `CmsZip` | on first access |

The snippets live in `Core/Modules/Template/php/`:

| File | Content |
|---|---|
| `push_result.php` / `pop_result.php` | save or restore database result, row counter and `$cms` |
| `include.php` | include the result template, count `includeDepth` |
| `select_count.php` | `CMS:SELECT` only: with `hits=false`, apply the total count after `pop_result` (`Cms::selectCountEnd()`) |
| `include_cached.php` | like `include.php`, then `Cms::includeEnd()` – `CMS:INCLUDE` only ([output cache](#output-cache-of-cmsinclude)) |
| `include_and_stop.php` | the same, then `exit()` (success templates) |
| `loop_start.php` / `loop_end.php` | result loop |

`admin.max_include_depth` (20) limits the nesting.

### Custom commands from add-on modules

An add-on module brings its own commands by setting two keys in its
`install/settings.php`:

```php
'commands' => 'commands.php',                                // relative to the module
'services' => ['pdf' => 'Camindo\\Modules\\Pdf\\CmsPdf'],     // object in the template context => class
```

* `commands.php` has the same structure as the core file; command names should carry
  the module name as a prefix (`'pdf_page'` → `CMS:PDF_PAGE`). A name the core already
  uses stays with the core.
* `services` extends the lazily loaded services of `Page`: `'method' =>
  'pdf.cmsPdfPage'` becomes `$this->pdf->cmsPdfPage($_a)`, and the class comes from
  the container on first access (autowiring). The names `cms`, `db`, `pagination`,
  `recordUser`, `email`, `zip` are reserved.
* The method receives the parameter array `$_a` and either outputs or returns text,
  or – for commands with `include` – returns the name of the template to include.

If the module is inactive, the parser no longer knows its commands ("Unknown command"),
and an already compiled template fails when accessing the service. Example: the `Pdf`
module.

## The commands at a glance

| Command | Method | Purpose |
|---|---|---|
| `CMS:SELECT` | `cms.cmsSelect` | query, includes `resulttemplate` |
| `CMS:RESULT_START` / `RESULT_END` | – | result loop |
| `CMS:TEXT`, `CMS:NUMBER`, `CMS:DATE`, `CMS:DATETIME`, `CMS:TIME`, `CMS:RAW` | `cms.cms…` | formatted output of values |
| `CMS:MEDIA` | `cms.cmsMedia` | file or image version |
| `CMS:RELATION` | `cms.cmsRelation` | linked records |
| `CMS:URL` | `cms.cmsUrl` | URL of a record |
| `CMS:INCLUDE` | `cms.cmsInclude` | include another template, optionally with the [output cache](#output-cache-of-cmsinclude) |
| `CMS:SELECT_LANGVERSION` | `cms.cmsSelectLanguages` | languages of the instance as a result set |
| `CMS:HEADER` | `cms.cmsHeader` | set an HTTP header or (without value) remove it |
| `CMS:SNIPPET` | `cms.cmsSnippet` | snippet (text module) in the request language |
| `CMS:MESSAGE`, `CMS:CONTINUE`, `CMS:SAVEDATA`, `CMS:XML` | `cms.cms…` | messages, flow, data, XML |
| `CMS:CSP`, `CMS:NONCE` | `cms.cmsCsp`, `cms.cmsNonce` | script policy of the page ([14](14-security-web.md#script-policy-in-the-frontend)) |
| `CMS:FED`, `CMS:FED_SCRIPT`, `CMS:FED_CODE` | `cms.cmsFed…` | frontend editing |
| `CMS:JUMP_LIST`, `JUMP_PREV`, `JUMP_NEXT` | `pagination.cmsJump…` | page navigation |
| `CMS:EMAIL` | `email.cmsEmail` | send e-mail from templates |
| `CMS:ZIP` | `zip.cmsZip` | files of several records as a ZIP download |
| `CMS:USER_*` | `recordUser.cms…` | frontend community ([12](12-community.md)) |

## Runtime of `CMS:SELECT`

1. Check `includeDepth` against `admin.max_include_depth`.
2. `filter` or `relation` must be set.
3. Translate parameter names (`filter_op` → `filterop`, `filter_mode` → `filtermode`,
   `hits` → `result`); `relation` without `src_id` gets the current record. If `src_id` stays below 1,
   `select()` returns no rows (instead of dropping the relation filter).
4. Add the **publication condition** (`Cms::publishedSql()`): status 0, `online_at`
   reached, `offline_at` not reached. `future=true`, `archive=true` and `state=N|any`
   relax this per query. This also applies in the preview – there, only the requested
   record itself may be unpublished.
5. Run `Cms::select()` ([04](04-select-and-filter.md)).
6. Hits → return the name of `resulttemplate`; no hits → output `noresult` or return
   `noresulttemplate`; the `include` snippet includes it.
7. `hits=false` (implies `count=true`) loads no rows. `cmsSelect()` pushes the total
   count onto a stack; `select_count` sets it as `num_rows_unlimited` of the calling
   template after `pop_result` (otherwise it would be nowhere, since no `resulttemplate`
   is included). `noresult`/`noresulttemplate` apply only when it is 0.

The parameters `lang` and `collate` are defined but have no effect.

**Nesting.** A `CMS:SELECT` may stand inside a result loop, and its `resulttemplate`
may contain a loop again. The `push_result` / `pop_result` snippets save the database
result, the shared counter `Cms::$row_no` and `$cms` and restore them afterwards.
Self-test: `php Core/cron/check_nested_select.php`. Exception: `(CMS:USER_BASKET
function=load)` replaces the current result set without saving it and therefore does
not belong inside a result loop.

Available in the template:

| Variable | Content |
|---|---|
| `$cms` | the current result row or the page record |
| `$set` | values stored with `set=` |
| `$this->cms`, `$this->db` | `Cms`, `Database` |
| `$this->cms->row_no` | current row number |
| `$this->isPreview()` | true in the preview from the backend |

## Output cache of `CMS:INCLUDE`

`(CMS:INCLUDE teaser caching=5m cacheparams=$params)` stores the ***output*** of the
included template and serves it within the period without running the template.
Implemented in `Camindo\Core\IncludeCache`, wired up in `Cms::cmsInclude()` and the
snippet `include_cached.php`:

```php
$_r = $this->cms->cmsInclude($_a);   // hit: prints the file, returns ''
if (!empty($_r)) { … require …; }    // otherwise: the template runs into an output buffer
$this->cms->includeEnd();            // finish recording, store, print
```

`cmsInclude()` pushes one entry per call onto a stack (nested includes overwrite
`$_r`/`$_a`); `includeEnd()` pops it.

* ***Period*** `caching`: seconds, `30s`, `5m`, `1h`, `1d`, `infinite`; empty or `0`
  switches it off, anything else is a template error (`IncludeCache::ttl()`).
* ***Key***: sha256 of instance, content language and `cacheparams` as text
  (`IncludeCache::paramString()`: scalars via `(string)`, arrays as JSON). The parameter
  carries `'php_value' => true`, so an unquoted variable arrives with its type.
* ***Storage***: `var/cache/include/{area_id}/{name}/{hash}.html`, the first line is the
  expiry time (0 = unlimited). A file is valid only while it has not expired, is younger
  than the period of the call and younger than the compiled template file – saving or
  `cache templates` invalidates a template's cache.
* ***Nonce***: the page nonce is replaced by a placeholder when storing and by the
  current one when serving – cached scripts run under the script policy.
* ***Concurrent calls***: whoever renders creates `{hash}.html.lock` exclusively
  (`fopen(…, 'x')`); the others wait up to `admin.include_cache_lock_wait` (30 s) and
  then read the fresh file; after that they render without storing. No `flock()` – it
  does not lock across processes on every file system. A shutdown function releases
  locks after `exit`/fatal error; orphaned locks (older than twice the waiting time, at
  least 120 s) are taken over. Files are written to a temporary name and `rename()`d.
* ***Not stored*** if `Cms::error()` ran during rendering. ***No cache*** in the preview
  (`Cms::$includeCacheOff`, set by `Page::resolve()`), with front-end editing switched on
  and with `admin.include_cache = false`.
* ***Output only***: `$set`, headers, cookies and other side effects of the included
  template are skipped on a hit.
* ***Clean-up***: scheduler task `cachecleanup` (`IncludeCache::cleanup()`: expired
  files, files of deleted or changed templates, old lock and temporary files, empty
  directories); `camindo.php cache include [--template={area_id}/{name}]` empties it.
  The start page points out a missing task as soon as the cache contains files.

Self-test: `php Core/cron/check_include_cache.php`.

## The `id` parameter

Output commands (`CMS:TEXT`, `NUMBER`, `DATE`, `TIME`, `DATETIME`, `RAW`, `MEDIA`, `URL`,
`SAVEDATA`) normally read from the current record. With `id`, they read from another
one – resolved as in the URL (`Cms::_record()`): digits only = `record_id`, everything
else a path `[slug-prefix/]slug`. Every reference is loaded once per request and
remembered; an unknown reference counts as an empty value (`ifempty` applies).

## URL and slug

`CMS:URL` produces exactly the format `Page` takes apart again (`Cms::recordUrl()`):

```
[/{lang}]/{slug|id}[,{template}]
```

* The language segment is always part of the URL as soon as the instance has more than
  one language; `lang=de_DE` points to another language version – with the slug of
  **that** language (`Cms::languageSlug()`).
* If the record has a field `slug`, it wins. `slug` is an optional virtual field;
  `Record::hookOnSave()` fills it from the default field if it is empty
  (`Cms::textToUri()`: tags removed, special characters to hyphens, lower case,
  umlauts and `/` kept, at most `system.link_length` characters).
* **A slug is never a pure digit sequence** – "2024" becomes `-2024`. `Page` uses the
  same rule: digits only = `record_id`, everything else = slug.
* A slug, once set, is **not** updated when the title changes – shared links stay
  valid.
* **Slug prefix:** an area can have a prefix per language (`area.details.slug_prefix`)
  that is put in front of the slug: `/en_US/colours/blue`. The whole path must be
  unique (`Record::slugPathOk()`); `Page` answers two matches for a path with
  "Ambiguous path".
* **Overview (prefix only):** if the path is exactly an area's slug prefix in the
  request language and the area has an overview template (`area.overview_template_id`,
  `Cms::overviewAreaIdByPath()`), `Page` includes it without a record – like a template
  chosen by the CMS, without the standalone flag and without canonical redirect.
  `Cms::$overviewAreaId` holds the area; `CMS:URL` without `id` then points to the same
  overview (language switcher), `(CMS:URL area=…)` builds the address from elsewhere
  (`Cms::areaUrl()`). Without an overview template: 404. Instead of a record, `$cms`
  holds the area's details (`Page::overviewData()`): `record_id` (0), `area_id`, `area`,
  `area_label` (snippet resolved), `area_type`, `area_icon`, `instance_id`, `parent_id`,
  `language_id`, `slug_prefix` (request language), `overview` (true) – e.g. for
  `filter[area]="{$cms['area_id']}"`. In the backend, `Backend::buttonAddAreaPreview()`
  gives areas with an overview template and a prefix a preview button (area info, record
  list); for this, `buttonAddPreview()` also accepts `language_id => path` instead of a
  path (the prefix is translated per language).
* **Language code only** (`/en`, without a slash): the rewrite rule for the language
  segment requires the slash, so the path arrives as the slug `en`. If it is a language
  code of the site (`InstanceCache::findByCode()`, the same rule as for `/en/`), `Page`
  redirects with 301 to the start page of that language – using the code from the
  `language` table (`/en_US/`), or to `/` if there is only one language. GET/HEAD only,
  not in the preview. With a single language, a record with exactly this slug takes
  precedence; with several, every record address carries a language segment anyway.
* **404:** an unknown record (including one from another instance), an unknown
  language code, an unknown or non-permitted URL template and a missing template file
  end with status 404 (`Page::notFound()`) and a plain-text message. A non-permitted
  template deliberately gets 404 instead of 403. There is no designed error page yet.

## Template assignment

`Page` decides which template a record gets: template from the URL, otherwise
`record.template_id`, otherwise `area.default_template_id`. A template from the URL
needs a permission:

| Where | Meaning |
|---|---|
| `rel_area_template.selectable` | selectable in this area |
| `rel_area_template.override` | may replace the assigned template via URL |
| `template.standalone` | may be called **without** a record (`/0,{name}`) |
| `template.type = 'scheduler'` | never through an address, only by the scheduler ([07](07-media-processing.md#scheduler-module-scheduler)) |

Without permission, the request ends with "Invalid template request".

## Templates in the backend

**List template** (`area.list_template_id`): the backend runs it per record and shows
the output as the last column of the record list and in the detailed lookup preview
(`Page::renderRecord()`). The record is available as `$cms`; all commands work as in
the frontend. An error appears in red in the cell; the list continues.

**Preview** (`backend/preview.php`): `$this->isPreview()` is true, anti-caching and
`noindex` headers are set.

## Trigger templates

A content area can have ***trigger templates*** (`rel_area_trigger`, any number, with
`position`): templates that contain **event methods** instead of output. `Record`
calls them from its hooks (`TriggerTemplates::run()`):

```php
// template "person_trigger" in the template area – methods only, no output
function onSave(array &$affectedFields, array &$data, array &$previousData): void
{
    $data['title'] = trim($data['first_name'] . ' ' . $data['last_name']);
    if (!in_array('title', $affectedFields, true)) $affectedFields[] = 'title';
}
```

| Event | When |
|---|---|
| `onCreate(array &$data)` | `init()` – default values |
| `onLoaded(array &$data)` | after every `Record::load()` (not for `CMS:SELECT`) |
| `onSave(&$affectedFields, &$data, &$previousData)` | before validation and writing – add newly set fields to `$affectedFields` |
| `onSaved(&$affectedFields, &$data, &$previousData)` | after writing, before the commit |
| `onSoftDelete`, `onRestore`, `onDelete`, `onDeleted` (`array $data`) | trash, restore, final deletion before/after the `DELETE` |

* The compiled code is put into the body of an anonymous class extending
  `TriggerTemplate` (`var/cache/trigger/{template_id}.php`). Hence: methods only, no
  statements outside them, no output, fully qualified class names (no `use`).
* Available in the template: `$this->cms`, `$this->db`, `$this->record`,
  `$this->areaId`, `$this->temporary` (true when a working copy is saved temporarily)
  and the lazy services of `Page` (`$this->email` …).
* Except for `onCreate`/`onLoaded`, everything runs in the operation's transaction; an
  exception (`throw new \Exception('message')`) aborts it, and the message appears in
  the form.
* Several templates run one after the other by `position`; all receive the same
  references, so each one sees the changes of the previous ones. An exception ends the
  chain. The ids are kept per area in the `InstanceCache` (`trigger_template_ids`).
* During the call `$GLOBALS['cms']` holds only `['record_id' => …]` (restored
  afterwards). In `onSave`, and in `onSaved` when a working copy is saved temporarily,
  this is the working copy (`Dataset::savingWorkingCopyId()`), which holds the
  relations until check-in (outgoing and incoming, `TemporaryTrait::_copyRelations()`):
  `CMS:SELECT relation=…` / `CMS:RELATION` without a source therefore see the new
  state, and so does the reverse direction with `dest_id` set to the working copy;
  `$data['record_id']` stays the id of the original (stored state). The publication condition and the group rights of
  the caller apply as usual.
* When moving, the trigger templates of the **target area** apply.
* An event does not run nested for the same record per template (protection against
  endless loops).
  A template that cannot be loaded throws an exception.
* The API is available via `$this->cms->api('record.search', [...])`
  ([16](16-api.md#from-templates-and-custom-tools)) – in the backend as the user who
  saves; for a frontend user (`CMS:USER_INPUT`) there is none.

## Editor support and messages

`styled_code` contains the source with markup (`c9-command`, `c9-php`); parameters with
`'ref' => 'template'` become links to the referenced template. The parser reports
errors with line numbers (unknown command, unknown parameter, missing parenthesis) and
**notes** (`TemplateParserTrait::inlineHandlerWarnings()`, shown in yellow): `<script>`
without `(CMS:NONCE)` as well as `on…` attributes and `javascript:` addresses in the
HTML part – both are blocked by an active script policy. The "Template messages" widget
lists all templates with messages.

`TemplateParserTrait::styleCode()` returns the same markup without compiling and without
template links. The Docs module uses it to colour the ```` ```cms ```` blocks of the manuals
(`Docs::render()`); ```` ```cmscode ```` colours the whole block purple without parsing it
(syntax descriptions such as `(CMS:TEXT field [parameter=value …])`); other code blocks
stay unchanged.

## Frontend editing

`(CMS:FED)` outputs, for logged-in users, a button to the form of the current
record (or the one named with `id`) – as `<a href="#" data-c9-fed="42" class="c9-fed">`,
without a handler attribute. `(CMS:FED_SCRIPT)` delivers the function `cFED()` and the
click listener (with nonce) once per page, usually in the footer. Output only happens
if the signed cookie `fed` is valid (`Cms::fedActive()`,
[14](14-security-web.md#frontend-editing)). The target in the backend is thus defined in
exactly one place (`Cms::cmsFedScript()`), not in every delivered page.
