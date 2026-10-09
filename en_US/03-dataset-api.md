# 03 – Dataset API

`Camindo\Core\Dataset` is the abstract table abstraction. Every table of a module gets
a subclass that essentially defines a **schema** and optionally **hooks**. Validation,
transactions, permission checks, audit log, trash and trees come from the base class.

```php
namespace Camindo\Core\Modules\Status;

use Camindo\Core\Dataset;

class Status extends Dataset
{
    protected function hookDefinition(): void
    {
        $this->tableName        = 'status';
        $this->moduleName       = 'Status';
        $this->defaultSortOrder = 'position ASC';
        $this->defaultFieldName = 'label';
        $this->schema = [
            'status_id'   => ['type' => 'integer', 'label' => 'status_id', 'min' => 0],
            'label'       => ['type' => 'chars', 'label' => '#status.field_status_label',
                              'max' => 64, 'flags' => ['trim', 'not_empty']],
            'icon'        => ['type' => 'chars', 'label' => '#status.field_status_icon', 'max' => 64],
            'color_class' => ['type' => 'chars', 'label' => '#status.field_status_color', 'max' => 64],
            'position'    => ['type' => 'integer', 'label' => '#status.field_status_position', 'min' => 0],
        ];
    }
}
```

```php
$status = $app->get(Status::class);        // always through the container
$data = $status->init();                   // defaults
$data['label'] = 'Released';
$id = $status->save(['label', 'icon', 'position'], $data);
if ($id === null) echo $cms->errorMsg();
```

Dependencies (`Config`, `Database`, `Crypt`, `Cms`, `System`, `Session` …) are
autowired. The primary key is called `{tableName}_id`.

## Schema

A schema is an array `field_name => column_definition`:

```php
'name' => [
    'type'         => 'chars',
    'label'        => '#admin.field_admin_name',   // leading # = snippet
    'tooltip'      => '#admin.tooltip_field_name',
    'display_type' => 'text',
    'min'          => 3,       'min_op' => '>=',    // '>=' (default) or '>'
    'max'          => 64,      'max_op' => '<=',
    'pattern'      => '',
    'prefix'       => '',      'suffix' => '',
    'default'      => '',
    'flags'        => ['trim', 'not_empty', 'name', 'unique'],
    'table'        => 'admin', // for type 'id': target table
    'options'      => [],      // for display_type 'select'/'radio'
    'extension'    => '',      // extension for the input field, e.g. 'Password'
],
```

All schema fields are ***meta fields*** (the constructor sets `'meta' => 1`) – as
opposed to the virtual fields of content areas, which `Record` adds at runtime from
`field`.

**Sub-fields.** A dot in the name means: the value lives in a JSON field.

```php
'details.width'  => ['type' => 'integer', 'label' => '#proc.imagegd.field_width'],
```

`init()` turns this into `$data['details']['width']`. A sub-field is only validated
when its parent field (`details`) is written.

**Fields without their own column: `reltable`.** Data stored in a separate relation
table (such as the languages of an instance) gets the type `reltable`. It behaves like
`json` (value is an array, `min`/`max` count entries, `not_empty` = at least one entry)
but `save()` does not write it as a column: the subclass writes it in `hookOnSaved()`
and reads it in `hookOnLoaded()`. `select()` can neither filter nor sort by it.

**Date values.** Date columns are `NOT NULL`. "No date" is
`Cms::DATE_MIN = '1000-01-01 00:00:00'`, "no end" `Cms::DATE_MAX =
'9000-01-01 00:00:00'`; `_formatDate()` outputs both as an empty string. Smaller values
are rejected, not silently raised. SQL functions (`NOW()`, `UTC_TIMESTAMP()` – only from
`Cms::$funcWhitelist`) are exempt from `min`/`max`. `created_at`/`created_by` are
neither written nor validated on updates.
The column default `0000-00-00 00:00:00` in `schema.sql` (e.g. on `data_dat`) is only a
placeholder for INSERTs without a value; the CMS itself always writes `DATE_MIN`.

**Defaults (`default`).** Scalar types: the value. Array types (`file`, `json`,
relations, `reltable`): **each line** is a selectable option. Date types: `now`,
`today`, optionally with an offset (`now+3d`, `today-2h`; units `s|m|h|d`).

## Schema flags

The constructor derives flags from the schema and checks its consistency:

| Flag | Condition | Effect |
|---|---|---|
| `hasParentId` | `parent_id` in the schema | tree possible |
| `hasDeleted` | `deleted` in the schema | soft delete |
| `hasTmp` | set in `hookDefinition()` | working copies |
| `hasAreaId` | `area_id` in the schema | `unique_in_area` possible |
| `hasChangedAt` / `hasChangedBy` | schema field or `hasTmp` | change metadata |

Hard rules (exception if violated): `parent_id` ⇒ also `parent_ids`, `lft`, `rgt`,
`position`, `depth`; `hasTmp` ⇒ `tmp` and `deleted` in the schema and the columns
`origin_id`, `changed_by`, `changed_at` in the table.

Further properties a subclass may set: `$sanitizeHtml = false` (no HTML filter on
write – only `Template`), `$audit = false` (no audit log), `$auditIgnore` (fields that
are not logged), `auditSecret()` for reversibly encrypted values.

## Traits

```php
class Record extends Dataset {
    use TreeManagerTrait, VirtualFieldsTrait, TemporaryTrait, LinkTrait, MoveTrait;
}
```

| Trait | provides |
|---|---|
| `TreeManagerTrait` | nested set + adjacency list ([02](02-data-model.md#trees-nested-set-and-adjacency-list)) |
| `VirtualFieldsTrait` | user-defined fields, uploads, thumbnails, relations, full text |
| `TemporaryTrait` | working copies, edit locks |
| `LinkTrait` / `MoveTrait` | dependent linked content / moving between areas |

## Hooks

All hooks are `protected` with empty defaults; only `hookDefinition()` is abstract.

| Hook | When |
|---|---|
| `hookDefinition()` | constructor – table, module, schema, sorting |
| `hookOnCreate(&$data)` | in `init()`, after the defaults |
| `hookOnLoaded(&$data)` | in `load()`, after reading |
| `hookOnSave(&$fields, &$data, &$previousData)` | in `save()`, **before** validation and writing |
| `hookOnSaved(&$fields, &$data, &$previousData)` | in `save()`, **after** writing, before the commit |
| `hookOnSoftDelete($data)` | when marking `deleted = 1` |
| `hookOnRestore($data)` | when restoring |
| `hookOnDelete($data)` | on **final** deletion, before the `DELETE` |
| `hookOnDeleted($data)` | on **final** deletion, after the `DELETE`, before the commit |

**Errors in hooks: throw an exception.** Only then does the surrounding transaction
roll back. If the message starts with `#`, it is considered already registered in
`Cms::$errors`.

Because `hookOnSave()` runs **before** validation, a value set there is validated
normally. If the hook sets a field that is not in `$fields`, it has to add it there,
otherwise it is not written.

```php
// Template::hookOnSave() – compile the source
$result = $this->parse($data['code'], $data['area_id']);
$data['styled_code'] = $result['styledCode'];
$data['error_text']  = implode('<br>', (array)$result['errors']);

// Record::hookOnSave() – fill an empty slug from the default field
$data['slug'] = $this->cms->textToUri($title);
```

For records there are also **trigger templates** – event methods a user
assigns per content area ([06](06-template-compiler.md#trigger-templates)).

## Reading and writing

### `init(array $defaults = []): ?array`

Complete data array with all defaults; `$defaults` overrides single values (sub-fields
as a nested array). For `record`, `$defaults['area_id']` is mandatory – only the area
determines the virtual fields.

### `load(string|int $value, string $fieldName = '', bool $forUpdate = false): ?array`

Loads by primary key or another column; `$forUpdate` adds `FOR UPDATE` (only useful
inside a transaction). Not found → message in `Cms::$errors`
(`{module}.error_{table}_not_found`, otherwise `error_record_not_found`), returns
`null`. Password hashes are never returned.

### `save(array $fields, array $data, bool $stayTemporary = false): ?int`

* `$fields` = names of the fields to write; **empty array = all fields**.
* `$data["{tableName}_id"]` decides: 0 = INSERT, > 0 = UPDATE.
* Returns the id, or `null` on errors (messages in `Cms::$errors`).
* Everything runs in one transaction, including audit entry and permission check.
* On UPDATE, fields that were not submitted stay unchanged.

For tables with working copies (`record`), additionally: only a **working copy** of
the session user is saved (call `checkOut()` or `tmpInit()` first); `$stayTemporary =
true` only writes the working state (tab change), `false` is the check-in. On check-in,
the **whole** record is validated – request and working state together. Meta fields not
shown in the form keep their previous value.

`Record::importSave()` is the variant for imports and scripts: values are already in
database format (`$storedFormatFields`), password hashes are taken over
(`$prehashedFields`), the count check of links can be deferred.

### `check(array $fields, array $data): bool`

Checks like `save()` – `hookOnSave()`, validation, rights, uniqueness – and stops before
writing (rollback): no INSERT/UPDATE, no `hookOnSaved()`, so no cache files either.
Messages in `Cms::$errors` as with `save()`. For trial runs (`simulate` of the API,
[16](16-api.md#configuration-datasets-datasetapi)); not for tables with working copies
(`record` checks via `RecordService`).

### `transactional(callable $task, mixed $onError = null)`

Wraps `beginTransaction`/`commit`/`rollback` – `protected`, for your own multi-step
operations in the subclass.

### `fetch()` / `select(array $param = [])`

Thin wrappers around `Cms::fetch()` and `Cms::select($param, $this)`, see
[04](04-select-and-filter.md).

## Deleting and restoring

Single-record functions handle exactly one record and fire its hooks; orchestrations
only collect ids:

| Single-record function | Effect |
|---|---|
| `softDelete(int $id): bool` | `deleted = 1`, `hookOnSoftDelete()` |
| `restoreOne(int $id): bool` | `deleted = 0`, `hookOnRestore()` |
| `finalDelete(int $id): bool` | remove the row, `hookOnDelete()`, `hookOnDeleted()` |

| Orchestration | calls |
|---|---|
| `delete(int $id, bool $withChilds = false)` | `softDelete()` – without `$withChilds`, children move up one level |
| `restore(int $id, bool $withChilds = true)` | `restoreOne()` – only records with the timestamp of the deletion |
| `purge(int $id, bool $withChilds = true)` | `finalDelete()` |

Tables without `deleted` redirect `delete()` straight to `purge()`. An orchestration
opens **one** transaction for the whole operation; a single-record function called
directly opens its own (this is how the trash can skip a failing record). If the id of
a working copy is passed, the original is meant.

**Cascades across tables** pass on the operation timestamp:

```php
// Area::hookOnSoftDelete()
$record = $this->recordDataset();
$record->setMarkTimestamp($this->getMarkTimestamp());
foreach ($record->cascadeIds('area_id', $areaId, 'delete') as $recordId) {
    if (!$record->softDelete($recordId)) {
        throw new \Exception('#' . $this->cms->errorMsgLast());
    }
}
```

To take part, a table needs `changed_at` and `changed_by` in its schema. Large
operations are moved to a background process by `MassDeletion` (threshold
`admin.mass_delete_threshold`); the same call runs there.

## Validation

`validateValue()` runs for every field that is actually written:

1. **Typing** – `bit` via `FILTER_VALIDATE_BOOLEAN` (`1/true/on/yes` or
   `0/false/off/no/""`, otherwise an error); array types as array; otherwise string.
2. **Preparation** – `strip_tags`, `trim` (implicit for `name` and numeric types),
   lower case for `display_type = email`. Without `strip_tags`, every text value goes
   through the HTML filter (`HtmlSanitizer`, [14](14-security-web.md#html-filter)).
   `password` is exempt.
3. **Flags** (table below).
4. **Pattern** – from `name`, `display_type = email` or `pattern`.
5. **Limits** `min`/`max` – character count for text, the value for numbers, the
   number of elements for array types.
6. **Password policy** – for fields with `display_type password` and the `Password`
   extension ([13](13-security-authentication.md#password-policy)).
7. **Uniqueness** – `unique` / `unique_in_area`.

| Flag | Effect |
|---|---|
| `trim` | remove surrounding whitespace |
| `strip_tags` | remove HTML |
| `not_empty` | mandatory; `"0"` counts as filled. An empty password on update keeps the previous one. For upload fields, the check-in verifies that a file exists |
| `not_zero` | not `0` |
| `name` | `trim` and pattern `^[a-z][a-z0-9_]+$` (at least two characters) |
| `unique` | unique within the instance and the instances linked to it (`InstanceCache::uniqueScope()`), table-wide for tables without `instance_id` |
| `unique_in_area` | unique within the area |
| `ft` | goes into the full-text column |

For virtual fields, `uniqueVirtualOk()` checks against the `data_*` table (productive,
non-deleted records); for `encrypted`, the HMAC is compared. Relations and files have
no uniqueness. The `slug` field checks the whole URL path including the slug prefix
instead (`Record::slugPathOk()`).

### Multilingual fields

The value arrives as `language_id => input` (`data[field][{language_id}]` from the
form). The primary language is validated completely, another language only if
something was entered – empty means "not translated". Errors are registered under
`{field}.{language_id}`; `unique` applies per language. A single scalar (import, CLI)
counts as the primary language's value.

### Error messages

```php
$cms->error('Message', 'field_name', 'Field label');
$cms->errorsReset();
$cms->errorMsg('<br>');    // all messages
$cms->errorMsgLast();      // the last one
$cms->getErrorFields();    // affected fields (highlighting in the form)
$cms->hasErrors;
```

Standard snippets: `error_value_mandatory`, `error_invalid_format`,
`error_min_ge_length`, `error_max_le_value`, `error_unique_value`,
`error_record_not_found`, `error_access_denied` …; `{value}` is filled with the limit.

### Date and time input

`dateInputToIso()` and `timeInputToIso()` understand incomplete and localised input
and complete it:

```
2023              → 2023-01-01 00:00:00   (for > and =)   or 2023-12-31 23:59:59 (for <)
2023-06-15 14:30  → exact
06/15/2023        → notation of the locale (CLDR patterns via IntlDateFormatter)
```

A pattern only counts if it consumed the **whole** input. Self-test:
`php Core/cron/check_dateparse.php`.

## Further methods

```php
$dataset->getSchema();            $dataset->getTableName();       $dataset->getModuleName();
$dataset->getDefaultSortOrder();  $dataset->getDefaultFieldName(); $dataset->hasTree();
$dataset->getBindType($type);     // mysqli bind type 'i', 'd', 's', 'b'
```

In subclasses also `areaInfo($areaId)` (name, type, instance of an area from the file
cache instead of the `area` table).
