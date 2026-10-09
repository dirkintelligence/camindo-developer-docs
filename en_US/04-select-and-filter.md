# 04 – Searching and filtering: `Cms::select()`

`Cms::select()` is the central read access – the same code for backend lists, backend
search and the template command `(CMS:SELECT …)`.

```php
$result = $cms->select($param, $tableObj = null, $completedSchema = []);
while ($row = $cms->fetch($tableName, $schema)) { … }
```

| Parameter | Meaning |
|---|---|
| `$param` | filter and sort criteria (see below) |
| `$tableObj` | dataset whose table is searched. **Default: `record`** – only then are virtual fields available |
| `&$completedSchema` | schema including virtual fields – only needed if readable labels should appear in the filter text |

The result is then in the current `Database` result and is fetched row by row with
`$cms->fetch()` or `$dataset->fetch()`. Return value of `select()`:

```php
[
    'filter_text'  => 'Name = Hen*',   // only with filtertext=true
    'sort_text'    => 'Name ascending',
    'default_sort' => 'name ASC',
    'used_sort'    => 'name ASC',
    'sort'         => ['name'],
    'order'        => ['ASC'],
]
```

Hit counts are in `$db->num_rows` and – with `count => 'true'` –
`$db->num_rows_unlimited`.

## Parameters

| Key | Meaning |
|---|---|
| `filter` | `field_name => value` or `field_name => [value1, value2]` |
| `filterop` | comparison operators per field, index-aligned with the values |
| `filtermode` | combination of values of the same field: `OR` (default) or `AND` |
| `sort` | field name(s), optionally with direction: `'name ASC'` or `['name', 'position']` |
| `order` | directions, index-aligned with `sort` |
| `skip`, `limit` | offset; maximum (`< 1` = unlimited) |
| `count` | `'true'`: additional count query for pagination |
| `result` | `'false'`: count only |
| `filtertext` | `'true'`: create a readable filter and sort text |
| `debug` | `'true'`: print SQL and bind values |
| `request` | which of the keys `filter`, `filterop`, `filtermode`, `skip`, `limit`, `sort`, `order` may additionally be taken from `$_REQUEST` (`'all'` or a list) |

**Only from `$param`, never from the request:**

| Key | Meaning |
|---|---|
| `sql` | additional WHERE condition(s), raw text – string or list, each appended in parentheses |
| `src_id`, `relation` | records the link field `relation` of record `src_id` points to; `relation` without `src_id`/`dest_id` finds nothing |
| `dest_id`, `relation` | reverse direction: records pointing to `dest_id` via `relation` |
| | `relation`: one field name, several comma-separated (OR) or `any` (any row in `data_rel`) |
| `nextof` | with `src_id`/`relation` (exactly one field): only the neighbours of record `nextof` in the link list |

Values from `$param` override request values. Wrongly typed request values are
discarded and registered as `error_param_invalid`.

```php
$cms->select(['request' => 'skip,limit,sort,order', 'filter' => ['area_id' => 8]], $record);

// the entry following 2411 in the gallery of 2422 – or, with order DESC, the previous one
$cms->select(['src_id' => 2422, 'relation' => 'gallery', 'nextof' => 2411, 'limit' => 1]);

// all records pointing to 4902 via any link field
$cms->select(['relation' => 'any', 'dest_id' => 4902]);
```

`filter[relation_any]=4902,4903` is the same as a filter value (several targets, OR).
The backend list (`RecordBackendList`) passes `relation` + `dest_id` from the request
after validation ("All references to this record" in the meta fields tab) and, without
a fixed area, restricts to the areas the user group may read (`Backend::areaAccessSql()`).
The search field `_url` (id, slug with/without slug prefix, or URL) is resolved by
`RecordService::urlFilter()` into `record_id` or `slug` (+ `area_id`, `instance_id`).

## Automatic conditions

`select()` adds without being asked:

* `tmp = 0` and `deleted = 0` if the table has these columns – working copies and
  deleted records never show up by accident.
* **Instance restriction** for `area`, `record`, `template` as soon as
  `Cms::setInstanceAccess()` is set: in the frontend, the domain's instance plus its
  readable instances; in the backend, the instances of the user group.
* **Area restriction** for `record`, `template` (`Cms::setAreaAccess()`): in the
  backend, only areas the user group may read ([11](11-permissions.md)).

`select()` has **no** publication filter – the backend sees everything. The frontend
command `CMS:SELECT` adds it via `sql` (`Cms::publishedSql()`: status 0, `online_at`
reached, `offline_at` not reached). If you query via `Database::query()` yourself, you
have to add instance and publication conditions yourself.

## Pseudo field names

| Name | Effect |
|---|---|
| `_default` | the default field of every area (JOIN on `data_dft`); for other tables `getDefaultFieldName()` |
| `area` | an area by its **name**: `filter[area]=article`, `=article,topic`, `=!article`; a number counts as `area_id`. `sort=area` sorts by area name |
| `instance` | sorting only, for `area`/`template`: order of the instances in the backend |
| `link.field` | sorting by a field of the linked record |

`filter[area]` is translated into `filter[area_id]` via the instance file cache before
the query is built – no sub-query. Allowed operators: `=`, `==`, `!`, `!=`, `!!`. An
unknown name returns **nothing** in a positive search and is registered as
`error_filter_value_invalid`.

## Filter syntax

Applies equally to the backend search form, to `filter[…]` in code and in templates.

* All filled fields are combined with **AND**.
* Several values of **one** field, separated by commas, are combined with **OR**
  (`filtermode` switches to `AND`).
* If a value contains ` OR ` or ` AND `, it is not split at commas but evaluated as an
  expression.

### Operators

| Input | Meaning |
|---|---|
| `henrik`, `=henrik` | equal (default), `*` is a wildcard |
| `==henrik` | exactly equal – `*` is an ordinary character |
| `!henrik`, `!=henrik` | not equal |
| `!*mann` | does not end with "mann" |
| `!!*mann` | not equal to the literal `*mann` |
| `>100`, `>=100`, `<100`, `<=100` | comparison |
| `=` or `==` alone | empty value; `0` for numbers |
| `(empty)` / `!(empty)` | no own content in this language (empty or fallback) / has one; on file fields (bare or `.filename`): no file (`NOT EXISTS` of a `data_fil` row with a filename) / a file; on reference fields (relation/pointer, including n:1): no reference to a productive record / at least one – edges to working copies and deleted targets do not count |

`*` stands for any number of characters (internally `LIKE` per spelling variant, see
below; `%` and `_` in the search value are escaped): `Hen*`, `*mann`, `*berg*`. Expressions in one field: `Hen* OR
Her*`, `>2 AND <4`, `!2 AND !3`. A value starting with an HTML tag (`<section>`,
`</p>`, `<!--`) is not an operator (`Cms::extractOperator()`) – to search for
containment: `*<section>*`.

In a template the operator may also stand between key and value
(`filter[price] >= 100`); the compiler prepends it to the value in its two-character
form, and the value itself is then free of operators (see
[06](06-template-compiler.md#syntax)).

### Collation per language

Text and character fields (`text`, `chars`, `_default`, file name and OCR text) are
compared and sorted by the rules of the language being searched. The columns use the
neutral collation; where the language differs, `Cms::collate()` appends a `COLLATE` to
the column expression – in conditions and in `ORDER BY`:

```sql
`_vf_0`.`data` COLLATE utf8mb4_de_pb_0900_ai_ci = ?
```

* **Derivation:** `Camindo\Core\Collation::forLocale()` from `language.code` (`de_AT`
  → `de`): German `de_pb` (ä = ae, ö = oe, ü = ue), Swedish `sv` (ö ≠ o), Danish/
  Norwegian (aa = å), Turkish etc. – `_ai_ci` variants only, otherwise neutral (also
  Chinese/Japanese, which MySQL only knows accent-sensitive). Candidates in order:
  `utf8mb4_{x}_0900_ai_ci` (MySQL 8), `utf8mb4_uca1400_{name}_ai_ci` (MariaDB ≥ 10.10),
  `utf8mb4_{name}_ci`; the first one the server knows wins.
* **Cache:** `InstanceCache` stores the result per language in `instances.php`
  (`collations`, `InstanceCache::collation()`); `''` means "same as the column" – then
  no `COLLATE` is added. A collation in the expression prevents a range access on
  `data` in the index; `language_id` and `fieldname_id` remain usable.
* **Which language:** With a language context (frontend) the requested one. Without
  (backend) the languages of the instances from `filter[area_id]` (otherwise all) per
  `$languageMap`; if they lead to different collations, the neutral one stays – a
  `COLLATE` applies to the whole query (`Cms::textCollation()`). Tables other than
  `record` compare with the column collation.
* **Wildcards:** `LIKE` compares character by character; one character never matches
  two there (`weiß` LIKE `'weiss%'` is false in every collation). So
  `Collation::likeVariants()` generates the spellings the collation treats as equal
  (always ß/ss, æ/ae, œ/oe; German ä/ae, ö/oe, ü/ue; Danish/Norwegian å/aa, at most 16),
  joined with `OR`. `LOCATE()` is no substitute: MySQL only compares slices of equal
  byte length there (`LOCATE('e', 'Café')` = 0).
* **Full text:** `MATCH()` knows no per-query `COLLATE`. So `data_ftx.data` uses
  `utf8mb4_0900_as_ci` (accent-sensitive), and `Cms::indexText()` folds text and search
  term with `Collation::fold()` by the rules of the language – via ICU (`\Collator`,
  primary strength), which knows the same CLDR rules as MySQL: neutral "Müller" →
  `muller`, German `mueller`, Swedish keeps "förvaring". Each `data_ftx` row is folded
  in its language (`_refreshFulltext()`), the search term in the search language; if
  several search languages (backend) give different terms, each one is searched (`OR`).
  Meta columns of other tables (`admin.ft`) stay unfolded. When the rules of a language
  change for existing full text – a language is added to an instance or gets another
  code – the Instance and Language hooks mark it as outdated with
  `LanguageData::markFulltextStale()` if `Collation::sameRules()` says no (rebuild see
  [Full-text search](02-data-model.md#full-text-search)).
* **MariaDB:** If the server does not know `utf8mb4_0900_ai_ci` or `_as_ci`, `SqlFile`
  replaces them with the next best when running `schema.sql` and migrations
  (`Collation::portable()`: `uca1400_ai_ci`/`unicode_ci` or `uca1400_as_ci`/`bin`); the
  installer creates the database the same way.

### Date fields

Incomplete values are completed – to the start of the period for `>` and `=`, to its
end for `<`: `>=2023` from 2023-01-01 00:00:00, `<2023-06` until 2023-06-30 23:59:59.
Localised notations of the locale are understood.

### Encrypted fields

The deterministic HMAC (`data_enc.hash`) is compared, not the ciphertext. Hence: only
exact equality and inequality, byte-exact, no wildcards, no ranges. A wildcard or range
operator is an **error** and returns nothing (`1=0`) – deliberately not "drop the
condition", which would return all records.

### Full-text fields

MySQL boolean syntax: `miller`, `+miller -blocked`, `miller meyer`. No wildcards. A word
that `indexText()` splits into several (`sofa-weiss`) becomes the phrase `"sofa weiss"`
(with a trailing `*`: each word with the operator). If no word remains after
`Cms::indexText()` (only tags, punctuation), the condition matches nothing (`1=0`)
instead of being dropped. `filter[fulltext]` hits the
[standard full text](02-data-model.md#standard-full-text-fulltext) of every area – even without `area`. Index and search term are folded by the rules of the
language, see [Collation per language](#collation-per-language).

### Tree filters

`depth` modifies `filter[parent_id]`, relative to the parent node:

| `depth` | Effect |
|---|---|
| missing or `< 0` | all descendants |
| `0` | empty result |
| `1` | direct children |
| `n` | down to relative depth n |

`['parent_id' => '', 'depth' => 1]` returns the root nodes. The parent node's area is
filtered automatically.

### Limits

Not possible: regular expressions, comma **and** `OR`/`AND` in the same field, values
that contain a comma themselves, and wildcards in encrypted and full-text fields.

## How the query is built

1. `filter()` builds the WHERE clause and collects a **JOIN context**; every virtual
   field access registers exactly one JOIN, shared by filter and sorting.
2. Every virtual scalar value is joined to its `data_*` table, restricted by
   `fieldname_id` **and** `language_id` (index `idx_filter(language_id, fieldname_id,
   data)`). The language comes from `Cms::languageExpr()`: the requested one in the
   frontend, otherwise each record's primary language. Because every language has
   exactly one row, the JOIN stays 1:1.
3. For `record`, the language's `record_cache` is joined as column `cache` as well.
4. Relations are built as an `EXISTS` sub-query or as a JOIN group, file filters via
   `EXISTS`; sorting by file metadata (`filename`, `mime_type`, `extension`, `size`,
   `width`, `height`) is possible.
5. The count query only gets the filter JOINs.
6. `DISTINCT` only if a JOIN can produce multiple rows.
7. Everything uses prepared statements.

`debug => 'true'` shows the final SQL.

## Sub-queries

`select()` overwrites the current result. If you run a second query during an
iteration, save it:

```php
$this->db->pushResult();
$cms->select(['filter' => ['parent_id' => $id]], $record);
while ($child = $cms->fetch()) { … }
$this->db->popResult();
```

## Pagination

`Pagination` builds page navigation from `Cms::$list_rows`, `$db->num_rows` and
`$db->num_rows_unlimited`. The class is only loaded on first access
(`LazyServicesTrait`, `$this->pagination` in `Backend` and `Page`).

| Method | Template command | Output |
|---|---|---|
| `jumpList($param)` | `(CMS:JUMP_LIST)` | `1 … 5 6 7 8 9 … 15` |
| `jumpPrev($param)` | `(CMS:JUMP_PREV)` | link to the previous page |
| `jumpNext($param)` | `(CMS:JUMP_NEXT)` | link to the next page |

Important parameters of `jumpList()`: `skip`, `maximum` (7), `boundaries` (always link
first/last page), `separator`, `tpl_link`, `tpl_active`, `ellipsis`, `prefix`,
`suffix`, `ifempty`. `Cms::$list_rows` comes from `admin.skiplist_default` (20). The
backend has ready-made wrappers: `Backend::paginationTop()`, `paginationTopShort()`,
`paginationBottom()`.

`{href}` in the templates is HTML-escaped (`&amp;` instead of `&`), `'` and `\` are
percent-encoded – see [14](14-security-web.md).

## `fetch()` – preparing values

`$cms->fetch($tableName, $schema)` fetches the next row and prepares it according to
the schema: virtual fields from `cache`, decryption, JSON decoding, conversion of dates
into the user's time zone, HTML filter on string values
([14](14-security-web.md#html-filter)).

Formatters for display:

```php
$cms->_formatDate($iso, $displayTimezone = false);
$cms->_formatTime($iso, $displayTimezone = false);
$cms->_formatFloat($value, $fractionDigits = 0);
$cms->_nl2br($text);   $cms->_strlimit($text, $limit);   $cms->_strip_tags($text);
$cms->utcToUserTZ($date);   $cms->indexText($text);   $cms->mb_ucfirst($string);
```
