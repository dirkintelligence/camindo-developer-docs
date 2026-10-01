# 02 – Data model

The reference is `Core/install/database/schema.sql` (structure) with `seed.sql` (base
data). All tables: InnoDB, `utf8mb4`, collation `utf8mb4_0900_ai_ci` (neutral,
accent- and case-insensitive; exceptions such as hashes and tokens are `ascii`/`latin1`; `data_ftx.data` is accent-sensitive `utf8mb4_0900_as_ci`).
`Cms::select()` applies language-specific comparisons via `COLLATE`, see
[Collation per language](04-select-and-filter.md#collation-per-language). Date and time
values are stored **in UTC** and converted only for display.

## The basic principle: user-defined fields in per-type tables

An editor creates ***content areas*** (`area`) and gives them input fields (`field`).
A record (`record`) belongs to exactly one area. Field contents are **not** stored in
`record` but in one `data_*` table per data type:

```
record (metadata)         field (field definition)    fieldname (global name registry)
   │                         │                            │
   └──────────┬──────────────┴────────────┬───────────────┘
              ▼                           ▼
        data_txt / data_int / data_fil / data_rel / …
        PK (language_id, record_id, fieldname_id)
```

### `fieldname` – one name, one type

`fieldname` registers every field name system-wide: `name → (fieldname_id, type)`,
`UNIQUE(name)`. Consequences:

* A field name is bound to **exactly one type**. If `title` is created as `text`, no
  other area can create `title` as `integer`.
* `Cms::select()` can filter by `filter[title]` across areas without knowing the
  `field_id` per area.

The primary key of the `data_*` tables therefore contains `fieldname_id`, not
`field_id`; `field_id` is an extra column for the foreign key used when a field
definition is deleted. Unused `fieldname` entries are removed automatically.

Field names match `^[a-z][a-z0-9_]+$` (flag `name` in `Dataset`: start with a letter,
at least two characters – `_…` is reserved for pseudo field names such as `_default`),
`fulltext` is reserved, and they must not start with the prefix of a
**different** type:

```
bit_ chr_ cur_ dat_ dec_ enc_ fil_ flt_ ftx_ rid_ ids_ int_ jsn_ pwd_ txt_ tim_ tsl_
ptr_ (pointer)  pn1_ (pointer1)  rel_ (relation)  rn1_ (relation1)  lnk_ (link)
```

### Type → table

`Cms::$dataSuffix` is the authoritative mapping:

| Type | Table | Column `data` | Group |
|---|---|---|---|
| `bit` | `data_bit` | `tinyint unsigned` | numeric |
| `chars` | `data_chr` | `varchar(255)` | text |
| `currency` | `data_cur` | `bigint` (cents) | numeric |
| `datetime` | `data_dat` | `datetime` | date |
| `decimal` | `data_dec` | `decimal(20,10)` | numeric |
| `encrypted` | `data_enc` | `mediumblob` + `hash` | text |
| `file` | `data_fil` | own columns, see below | array |
| `float` | `data_flt` | `double` | numeric |
| `fulltext` | `data_ftx` | `mediumtext` + FULLTEXT index | – |
| `id` | `data_rid` | `bigint unsigned` (FK `record`) | numeric |
| `integer` | `data_int` | `bigint` | numeric |
| `json` | `data_jsn` | `mediumtext` + multi-valued index | array |
| `password` | `data_pwd` | `varchar(255)` – hash, never cached | text |
| `pointer`, `pointer1`, `relation`, `relation1`, `link` | `data_rel` | edge table | relation |
| `text` | `data_txt` | `mediumtext` | text |
| `time` | `data_tim` | `time` | – |
| `timestampl` | `data_tsl` | `datetime(3)` | date |

The groups correspond to arrays in `Cms` (`textTypes`, `numericTypes`, `dateTypes`,
`arrayTypes`, `relationTypes`, `singleRelationTypes`, `multiAreaTypes`,
`dependentTypes`, `multilingualTypes`).

**Special cases**

* `data_dft` – exactly one row per record and language with the text content of the
  area's ***default field*** (`area.default_field_id`). This fetches the "title" of any
  record with one JOIN, without knowing which field it is (pseudo field name
  `_default` in `select()`).
* `data_enc` – plain text AES-encrypted (key `.camindo.key`), next to it a
  deterministic HMAC in `hash` for equality searches.
* `data_fil` – instead of `data`, the file's metadata: `extension`, `mime_type`,
  `filename`, `size`, `width`, `height`, `token`, `version`, `file_language_id` and a
  JSON field `data`. The file itself lives in the file system.
* `data_rel` – edge table `(src_id, fieldname_id, dest_id)` with `position` for
  sortable links; a second index serves the reverse direction. It has no
  `language_id`.

## Multilingual content

All `data_*` tables except `data_rel` have `language_id` in their primary key. A
record has **exactly one row per field and language of its instance** (primary
language plus `rel_instance_language`):

| Field | Row per language contains |
|---|---|
| monolingual (`field.multilingual = 0`) | the same value in all languages |
| multilingual | the value of that language; without input, the primary language's value (***fallback***) |
| `file`, multilingual | the language's own file (`file_language_id`), otherwise the primary language's |
| `fulltext` | the full text of the source fields in that language |

The reason: `select()` filters and sorts one language with a single JOIN condition
(`language_id = …`), and the fallback applies **per field**. Whether a row is a
fallback is stored in its `empty` column (1 = no own input); the filter value
`(empty)` uses it to find records that have not been translated yet.

Types that can be multilingual: `bit`, `chars`, `currency`, `datetime`, `decimal`,
`encrypted`, `float`, `integer`, `json`, `text`, `time`, `timestampl` and `file`. Not:
`fulltext` (generated), relations, `id`, `password` and meta fields.

The rows are written by `VirtualFieldsTrait::saveVirtualFields()`. When the language
list of an instance changes, `LanguageData` updates the existing data (new language:
copy the primary language's rows; removed language: delete its rows).

### `record_cache` – content cache per language

```
record_id, language_id, cache MEDIUMTEXT      PK (record_id, language_id)
```

The JSON image of all non-relational virtual field contents of a record, one row per
language, with fallback values:

```json
{"title": ["chars", "Article"], "text": ["text", "English text", 1]}
```

The third element `1` marks a fallback. `select()` joins the row of the requested
language as column `cache`, `fetch()` reads the virtual fields from it – a list can be
rendered without joining all `data_*` tables per row. If the row is missing,
`Cms::createRecordCache()` rebuilds all languages. Passwords never appear there;
encrypted content is stored as Base64 ciphertext.

## Core tables

### `area` – content areas

```
area_id, instance_id, type ENUM('record','template','folder','profile','ugc'), name, label,
icon, details JSON, dependent,
position, depth, parent_id, parent_ids JSON, lft, rgt,       ← tree
default_field_id, sort_field_id, sort_order, default_template_id, list_template_id,
trigger_template_id, overview_template_id, login_name_field_id, login_password_field_id, deleted …
```

* `name` (`^[a-z][a-z0-9_]+$`, unique within the instance group) is the identifier used
  by templates: `filter[area]=article`.
* `type`: `folder` = folder in the tree, `record` = content, `profile`/`ugc` = profiles
  and contributions of the frontend community ([12](12-community.md)), `template` =
  templates.
* `instance_id`: the area's website; `record.instance_id` and `template.instance_id` are
  copies of it (saving `select()` a JOIN) and are updated when it changes.
* `dependent = 1`: ***dependent area*** – its records each belong to one source record
  that holds them via a field of type `link` (see below).
* `list_template_id`: template used only for the backend display (last column of the
  record list, detailed lookup preview).
* `trigger_template_id`: template with event methods that run on every save and delete
  ([06](06-template-compiler.md#trigger-templates)).
* `overview_template_id`: template for the address that consists only of the slug
  prefix (`/en_EN/events`) – an overview without a record ([06](06-template-compiler.md#url-and-slug)).
* `details` (JSON): among others `tabs` (multilingual form with tabs), `slug_prefix`
  per language, `profile_area_ids` (`ugc`), `second_factor` (`profile`).

### `field` – input fields

| Column | Meaning |
|---|---|
| `area_id`, `name`, `type`, `label` | assignment and identity |
| `display_type` | display in the backend (`text`, `textarea`, `select`, `lookup`, `switch`, `upload`, `thumbnail`, `embed` …) |
| `min`, `min_op`, `max`, `max_op`, `pattern`, `flags`, `default` | validation, see [03](03-dataset-api.md#validation) |
| `rel_field_id`, `rel_area_id`, `rel_field_area` | counterpart field and target area(s) of a relation |
| `parent_field_id` | thumbnail field: the field it is generated from |
| `custom_order` | links can be sorted manually |
| `visibility_input`, `visibility_search`, `visibility_list` | where the field appears |
| `default_field`, `meta`, `multilingual` | default field; meta field; content per language |
| `details` (JSON) | among others `extension`/`extension_options`, `user_auth`, `profile_area_ids`, `lookup_detailed` |

The converters of a thumbnail field are stored, sorted, in `rel_field_converter`.
Only `Field::convert()` (button "Change type") changes the type or the multilingual
flag of an existing field: text ↔ number is copied row by row, `relation` ↔
`relation1` keeps only the first link for `…1`, mono- ↔ multilingual switches the
`empty` flag or the files. `conversionTargets()` lists the possible targets,
`conversionLossy()` warns about data loss.

### `record` – records

```
record_id, area_id, instance_id, template_id,
position, depth, parent_id, parent_ids JSON, lft, rgt,     ← tree
online_at, offline_at, status_id, needs_review,            ← publication
created_by, changed_by, created_at, changed_at (datetime(3)),
owner_id,                                                  ← owner in the backend (admin)
user_id,                                                   ← owner in the frontend (profile)
tmp, origin_id, locked, deleted                            ← working copies, trash
```

### Further tables

| Table | Purpose |
|---|---|
| `admin` | users; password hash, signature `sig`, encrypted contact data, `settings` JSON (language, time zone, widgets …), second factor |
| `group`, `rel_group_instance`, `rel_group_area`, `rel_group_language` | user groups and their rights ([11](11-permissions.md)) |
| `admin_session`, `admin_session_record` | backend sessions; which working copy belongs to which session |
| `admin_login_attempt`, `admin_recovery_code`, `login_device`, `admin_passkey` | login protection, recovery codes, known devices, passkeys ([13](13-security-authentication.md)) |
| `instance`, `rel_instance_language`, `rel_instance_domain`, `rel_instance_instance` | websites, their languages, domains and readable instances |
| `language` | languages: `code` (URL segment, e.g. `de_DE`), `backend_code` (snippet language), `timezone` |
| `template` | templates: `code` (copy of the source, for searching only), `styled_code`, `error_text`, `standalone`, `type` (`scheduler` = scheduled) |
| `rel_area_template` | which templates an area may use or force via URL |
| `status` | record status (icon, label, colour) |
| `converter`, `rel_field_converter` | configured processors and their assignment to thumbnail fields |
| `rel_fulltext_text` | source fields of a full-text field (optionally via a relation) |
| `component`, `system` | installed components; version state of the core |
| `bgproc` | background processes |
| `scheduler` | scheduled tasks |
| `message` | internal messages (encrypted) |
| `history` | recently edited records and templates per user |
| `audit_log` | metadata of the audit log; the contents live in `var/audit/` |
| `user_session`, `user_basket`, `user_token`, `user_login_attempt`, `user_second_factor`, `user_recovery_code` | frontend community |
| `rate_limit` | request throttling |
| `doc_section`, `doc_keyword` | index of the manuals |

## Trees: nested set and adjacency list

`record` and `area` maintain both in parallel:

| Column | Role |
|---|---|
| `parent_id` | direct parent node |
| `position` | order among siblings |
| `depth` | level, **0-based** |
| `lft`, `rgt` | nested set – subtrees without recursion |
| `parent_ids` | JSON list of all ancestors – breadcrumbs without a query |

For `record`, every tree belongs to **one content area**: `lft`/`rgt` start at 1 in
each area, so every query on `lft`/`rgt` must also filter by `area_id`. Whether an
area has a tree at all is decided by a meta field named `parent_id` in that area
(`isTreeArea()`); without it, the tree columns stay empty and no tree maintenance takes
place. `area` has a single tree over the whole table.

The order is not entered as a number but as the **preceding record** (`previous_id`);
`TreeManagerTrait::calculatePosition()` computes `position` from it. Reordering only
happens when the request brings `previous_id` or a different `parent_id` – a `load()` +
`save()` leaves the record where it is.

| Method (`TreeManagerTrait`) | Purpose |
|---|---|
| `isTreeArea($areaId)` | does the area have a tree? |
| `loadTree($rootId, $levels, …)` | load a subtree |
| `liftChildren()` | move children up one level |
| `copySubtree()` | copy a subtree |
| `subtreeIds($data, $mode)` | ids of a subtree in processing order |
| `resetTreeFields($areaId)` / `initTreeArea($areaId)` | flatten an area / build its tree |
| `externalRebuildTree($areaId)` | recompute `lft`/`rgt`/`depth` |

If a dataset schema contains `parent_id`, it **must** also contain `parent_ids`, `lft`,
`rgt`, `position` and `depth` – otherwise the constructor throws. Repairing existing
data: `php Core/cron/rebuild_trees.php`.

## Relations

| Type | Meaning |
|---|---|
| `relation` | n:m, visible from both sides (counterpart in `field.rel_field_id`) |
| `relation1` | like `relation`, but this field holds at most one link |
| `pointer` | directed link, several target areas possible (`rel_field_area`) |
| `pointer1` | directed, at most one target |
| `link` | directed like `pointer`, but the targets **belong** to the source (see below) |

"At most one target" is enforced in two places: `createRelations()` rejects several
targets for a `…1` field, and on check-in a target record whose counterpart field
allows only one target loses its previous link
(`TemporaryTrait::_enforceSingleRelations()`).

```php
// VirtualFieldsTrait
createRelations($colDef, $srcId, $destIds, $replace = true, $checkTmp = true);
removeRelations($colDef, $srcId, $destId = null);
swapRelationPosition($colDef, $srcId, $destId1, $destId2);
checkRelation($srcFieldId, $srcId, $destId);
```

### Dependent linked content (`link`)

A `link` field holds records from **dependent areas** as parts of the source – for
example the text/image blocks of a page. `LinkTrait` enforces two rules:

1. **At most one source per target record.** A record that already belongs to another
   one is rejected; a copy gets no incoming `link` edges.
2. **The targets share the source's fate.** Deleting, restoring, purging, copying and
   moving propagate to the content via the record hooks (`cascadeDependents()`,
   recursively).

With `display_type = embed`, the forms of the content are shown inside the source's
form (`RecordBackendEdit::_embedBlock()`, actions via `ajax_embed.php`). The content is
then checked out together with the source and, on check-in, checked in, copied or –
if no edge points to it any more – deleted in the same transaction
(`checkInDependents()`); cancelling discards it as well.

## Full-text search

A field of type `fulltext` collects the content of other fields (`text`, `chars`,
`json`, optionally across a relation; mapping in `rel_fulltext_text`) in `data_ftx`.
Changed source fields are collected on save and processed **after** the check-in
(`refreshAffectedFulltext()`); `refreshFulltext($fieldId)` rebuilds everything. The text
is normalised with `Cms::indexText()` first (HTML tags removed, entities decoded, only
letters and digits; `&shy;` joins, the dot separates; words joined by hyphen or
underscore are stored separated AND concatenated: `sofa-weiss` → `sofa weiss sofaweiss`;
search terms use `indexText($t, forSearch: true)` without the concatenated variant;
folded by the rules of the language, `Müller` → `mueller` in German, see
[Collation per language](04-select-and-filter.md#collation-per-language)).
One row per language of the instance. A source can also be an upload field (`file`,
without thumbnails): it contributes `data_fil.filename`; `Record::renameFile()` rebuilds
the full text.

**Standard full text `fulltext`.** Every record area (`Area::CONTENT_TYPES`) has it –
without a row in `field`. The name is reserved (`Field::FULLTEXT`) and fixed in
`fieldname` (type `fulltext`, master data in `seed.sql`; `Field::cleanupFieldname()`
keeps it, `VirtualFieldsTrait::standardFulltextFieldnameId()` creates it if missing).
Its `data_ftx` rows have `field_id` NULL. Its sources are always all fields of type
`text`, `chars`, `json` and `file` (uploads, no thumbnails) in the area
(`standardFulltextSources()`); it is written by `refreshStandardFulltext()` – after every
save with changed source fields (`refreshAffectedFulltext()`), after `Field::convert()`
and `Field::hookOnDeleted()` for the whole area. `select()` filters by field name and
needs no field definition: `filter[fulltext]` works in every area and across areas; the
search forms offer it by themselves (`RecordBackendSearch`). The import skips a
`fulltext` from older export files. Display: record info, tab "Meta data" → "Show full
text" (`RecordBackendInfo::fulltext()`, all full texts per language); full-text fields do
not appear in the "Contents" tab.

Rebuilding afterwards: batch processing, mode `fulltext` (`RecordBatch`) – writes only
`data_ftx`, without working copy, lock or audit entry; permission: module access `Area`.
Full-text fields are neither in `record_cache` nor in the audit snapshot.

## Working copies

`record` is never edited directly but via a ***working copy*** (`TemporaryTrait`):

```
checkOut()        create a copy: tmp = 1, origin_id = original
                  record_cache rows of the original → working state of the copy
                  copy the links, set the edit lock

save(…, stayTemporary: true)
                  intermediate save (tab change): only the working state in the copy's
                  record_cache; uploads are already stored under the copy, thumbnails not yet

save(…)  = checkIn()
                  merge working state and input, validate as a whole
                  write the original's data_* rows and record_cache
                  rename files (tmp id → record_id), write data_fil
                  create thumbnails ONCE, copy links back
                  delete the working copy

cancel($tmpId)    discard the working copy and its files
```

The working state is stored in `record_cache` in the same format as the productive
state, under the copy's id; `VirtualFieldsTrait::_loadTmpData()`/`_saveTmpData()`
translate it into the working form (`field => [type, value]`, multilingual
`{language_id: input}`).

* **Edit lock:** a working copy locks the original for other users as long as its
  `changed_at` is younger than `admin.max_inactive_sec`. On logout, the session's
  working copies are discarded.
* **Uploads immediately, thumbnails later:** a file survives a failed validation, the
  expensive conversion runs only once.
* An existing working copy of the same user is reused (`findTemporaryByOrigin()`).

## Moving records

A record can move to another area (`MoveTrait`). Because fields belong to the area,
this is a **mapping** `[source_field => target_field | '']` – allowed only for the same
type and the same multilingual setting (`moveCompatible()`). The path leads through a
working copy: `movePrepare()` converts it to the target area, `moveSave()` saves it with
the target area's rules. If validation fails, the backend shows the target area's form
with the errors; cancelling leaves the record unchanged.

## Media storage

```
public/media/
├── {instance_id}/.htaccess                     domain restriction of the instance
├── {instance_id}/{field_id}/{record_id}.{ext}  uploads and thumbnails
├── {instance_id}/{field_id}/.htaccess          protected fields only
└── default/                                    placeholder images
```

Multilingual upload fields are named `{record_id}_{language_id}.{ext}`. Paths are
built exclusively by `Cms::mediaFolder()` / `Cms::mediaPath()`. Executable extensions
are always blocked (`php`, `phtml`, `phar`, `cgi`, `htaccess` …,
`VirtualFieldsTrait::$forbiddenExtensions`); extension and detected MIME type must
match. Access protection and delivery: [07](07-media-processing.md#storage-and-delivery).

## Deleting

**Users only soft-delete.** Only the trash deletes for good:

```
delete()   → deleted = 1, files to var/deleted/          (restore() undoes this)
purge()    → remove the row, delete the files for good   (cron trash.php after
                                                          admin.trash_retention_days)
```

Soft delete exists for `record`, `area` and `admin`; tables without a `deleted` column
delete permanently at once. Every change goes through **single-record functions** that
handle exactly one record including its hooks – `softDelete()`, `restoreOne()`,
`finalDelete()`. Orchestrations (`delete($id, $withChilds)`, `restore()`, `purge()`,
the cascade from area to records, the trash) only collect ids. No record disappears
past its hooks. Details: [03](03-dataset-api.md#deleting-and-restoring).

* **Order is mandatory:** delete leaves first, restore parents first
  (`record.parent_id` has `ON DELETE CASCADE`).
* **Operation timestamp:** all records of one operation get the same `changed_at`;
  `restore()` brings back only those.
* **Files move along:** `var/deleted/{field_id}_{record_id}_{version}.{ext}`; the
  `data_fil` rows stay until the final deletion.
* **Large operations** (more than `admin.mass_delete_threshold` records) run as a
  background process via `MassDeletion` – the same call, not a second deletion path.

## Time zones

`Cms` sets `system.timezone`, then the user's time zone (`admin.settings.timezone`)
or that of the content language (`language.timezone`) – for PHP and for the database
connection. Conversion: `Cms::utcToUserTZ()`, formatting `Cms::_formatDate()` /
`_formatTime()` (ICU, proleptic Gregorian) in the user's locale.
