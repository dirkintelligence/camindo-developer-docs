# 16 – API

The API is **another interface** to the backend – for programs, the command line, the
scheduler, custom tools and an LLM chat. It has no users and no permission model of its
own: the one logged in is always an administrator, and the rights of their user group
apply ([11](11-permissions.md)). What the API does shows up in the audit log as an
action of this administrator.

Classes: `Camindo\Core\Api\` (directory `Core/src/Api/`).

| Class | Purpose |
|---|---|
| `ApiSession` | log in, work along, resume, log out, information |
| `Api` | find, call and describe methods |
| `ApiContext` | interface and channel of the process, check per call, conversion of errors and languages |
| `ApiMethod` | attribute that makes a method callable |
| `ApiException` | error with a machine-readable code |
| `ApiValues` | field values in neutral format |
| `SystemApi`, `RecordApi`, `AreaApi` | the methods (see below) |

## Interfaces and channels

An ***interface*** is the way an admin logs in: `backend`, `cli` (command line), `php`
(PHP API). Which of them a group may use is set by the superuser in the group
(`group.details.interfaces`, without it only `backend`). Every session belongs to exactly
one interface ([13](13-security-authentication.md#sessions-and-cookies)).

The ***channel*** says who is working through the session. It appears in the audit log
as the origin (`AuditLog::setContext()`); the admin remains the actor.

| Channel | Who |
|---|---|
| `llm` | an LLM in the backend chat, in the admin's session |
| `tool` | a custom tool running from a backend request |
| `scheduler` | a scheduler task |
| `cli`, `php` | the interface itself |

## Sessions

| Method | Interface | Session |
|---|---|---|
| `login($name, $password, $maxLifetime, 'php')` | `php` | its own; with a second factor first `ApiException` `second_factor` with `uid`, then `loginCode($uid, $code, …)` |
| `runAs($admin, $channel, $maxLifetime)` | `cli` | its own, **without a password** – only with `PHP_SAPI === 'cli'`: whoever has the server's command line is trusted anyway |
| `attach($channel)` | `backend` | no new one: works along in the loaded backend session (LLM chat, custom tool) |
| `resume($sessionId, $channel)` | `cli`, `php` | resumes an existing session in another process or request |
| `logout()` | – | own session: discard working copies, delete the row; after `attach()` only end working along |
| `whoami()` | – | admin, group, interface, channel, module rights, instances with languages (codes) and read/write permission |

Own sessions (`cli`, `php`) expire after `api.max_inactive_sec` of inactivity and at the
latest at the maximum lifetime (`$maxLifetime`, capped by `api.max_lifetime`). They have
no cookie and no login device. Renewing the password and setting up TOTP only work in the
backend – until then `login()` refuses. Passkeys cannot be confirmed via the API.

With `attach()` the API sees exactly what the admin sees and leaves the admin's traces.
Only then are there preview links – they depend on a backend session
([14](14-security-web.md#fixed-backend-domain)).

## Calling

```php
use Camindo\Core\Api\{Api, ApiSession};

$app->get(ApiSession::class)->resume($sessionId);
$result = $app->get(Api::class)->call('system.whoami');

// ['ok' => true,  'data' => [...], 'warnings' => []]
// ['ok' => false, 'error' => 'invalid_param', 'message' => '…', 'errors' => [...], 'warnings' => []]
```

Before each call, `call()` checks session, loaded rights and interface
(`ApiContext::require()`) – without them `Access` would be unloaded and deny nothing.
Parameters are those of the PHP method in snake_case (`$areaId` → `area_id`); unknown,
missing or wrongly typed parameters are an error. Values are only converted without loss
(`"12"` → `12` for `int`, `"true"` → `true` for `bool`). Unexpected errors go to
`var/log/{date}_api-errors.log`; the response then only says `internal`. A transaction
left open is rolled back.

| Error code | Meaning |
|---|---|
| `not_logged_in` | no valid session, or the group may not (or no longer) use the interface |
| `second_factor` | login still needs the code |
| `access_denied` | the group's rights are not sufficient |
| `not_found` | object not found |
| `locked` | another admin is editing the record |
| `open_in_backend` | the admin still has the record open in an edit form |
| `validation` | validation failed – details in `errors` |
| `invalid_param` | unknown method or parameter |
| `internal` | unexpected error |

`errors` is a list of `field`, `language` (code), `label` and `message`
(`ApiContext::errorsFromCms()` builds it from `Cms::$errors`; keys `field.{language_id}`
become field and language code).

`system.describe` returns a **JSON schema** of the parameters for each method, with
descriptions – usable as a tool definition for an LLM. The texts appear in the admin's
language; `Api::describe($method, $lang)` returns them in another backend language (the
assistant uses `en_US`).

## Methods

`system.describe` returns all parameters of a method with their descriptions; this is
the overview.

| Method | Parameters (selection) | Result |
|---|---|---|
| `system.whoami` | – | admin, group, interface, channel, module rights, instances with languages |
| `system.methods`, `system.describe` | `method` | directory, JSON schema |
| `docs.search`, `docs.read` | `query`, `book`, `language`, `limit` or `page`, `anchor` | handbook sections by relevance (page, anchor, excerpt); a page or section as Markdown |
| `area.list` | `instance_id`, `types` | readable content areas with number of records and the group's rights |
| `area.info` | `area_id` | instance, languages, default and sort field, tree, slug prefix, rights |
| `area.fields` | `area_id`, `with_meta` | input fields: type, multilingual, required, limits, options, target areas |
| `record.search` | `area_id` (one id or a list, 0 = global), `filter`, `filterop`, `filtermode`, `sort`, `order`, `skip`, `limit`, `url`, `language`, `relation` + `src_id`/`dest_id` | result list like the backend list: `columns`, `rows` (values as text), `filter_text`, `paging` (`total`, `next_skip`) |
| `record.info` | `record_id`, `tab` (`content`, `meta`, `history`, `fulltext`), `language` (code, `all`), `fields`, `truncate` | header (area, status, times, lock, `open_in_backend`, rights, languages) and the fields in neutral format; shortened texts with `truncated`, relations with `total`/`more` |
| `record.field_value` | `record_id`, `field`, `language`, `skip`, `limit` | full content of a field, relations page by page, embedded content (`link`) with its fields |
| `record.update` | `record_id`, `data`, `fields`, `relations`, `files`, `parent_id`, `previous_id`, `owner_id`, `unlock`, `simulate` | check out, change, check in in one step; `record_id`, `simulated`, `record` (header) |
| `record.create` | `area_id`, `data`, `relations`, `files`, `parent_id`, `previous_id`, `owner_id`, `simulate` | new record (not in dependent areas) |
| `record.copy` | `record_id`, `data`, `relations`, `files`, `simulate` | copy as in the form (files, embedded content, after the original), with changes |
| `record.move_options` | `record_id`, `target_area_id` | per source field: has content, matching target fields, preselection |
| `record.move` | `record_id`, `target_area_id`, `mapping`, `auto_map`, `unlock`, `simulate` | mapping used, `dropped` (fields whose content is discarded) |
| `record.delete` | `record_id`, `with_childs`, `unlock` | recycle bin (`deleted`) or background process (`background`) |
| `record.restore` | `record_id` | back from the recycle bin, as in the recycle bin module |
| `record.message` | `record_id`, `to` (id or name), `subject`, `body` | internal message, the record is linked |
| `record.preview_url` | `record_id`, `language` | preview address – only after `attach()` |
| `record.batch` | `area_id`, `mode` (`edit`, `move`, `delete`, `fulltext`), scope: `ids` \| `filter`/`filterop`/`filtermode` \| `all`; `data`, `relations`, `files` (edit), `target_area_id`, `mapping`, `auto_map` (move), `simulate`, `background` (`auto`, `yes`, `no`) | `total`, `done`, `failed`, `errors` per record – or `background.token`; extra right "batch processing" |
| `record.export_fields` | `area_id`, `type` (`csv`, `docx`) | selectable field keys with preselection and subfields |
| `record.export` | `area_id`, `type` (`csv`, `docx`, `xml`), scope `ids` or `filter` (otherwise all), `fields`, `languages` (CSV: codes; DOCX: [from, to]), `options` (`header`, `bom`, `separator`, `enclosure`, `eol`; XML: `files`), `background` | `zip`, `path`, `size`, `records`, `valid_until` – or `background.token`; rights as in the form |
| `record.exports` | – | existing export files |
| `system.processes`, `system.process`, `system.process_abort` | `token` | own background processes: `status` (`running`, `finished`, `timeout`, `failed`), `progress`, `message` |

The parameters of `record.search` are those of `Cms::select()` ([04](04-select-and-filter.md));
there is no `sql`, `request` or `debug`. The list returns the values in the short form of
the backend list (text in the admin's locale, shortened) – `record.info` provides the
complete values in neutral format. Without `language`, the backend's language applies
(the admin's preferred language version, otherwise the primary language).

Large amounts run as a ***background process*** as in the backend – from
`admin.mass_delete_threshold` (batch) or `admin.export_background_threshold` (export),
or with `background: yes`. The response then contains `background.token`;
`system.process` returns the state, and the result also arrives as an internal message.
In the audit log, a batch run carries the channel along (`llm-batch`, `scheduler-batch` …).

### Severity

Every method carries a ***severity*** (`ApiMethod::LEVEL_*`, returned as `level` by
`system.methods` and `system.describe`): how easily its effect can be undone. The
[assistant](#assistant-add-on-module-camindomodule-ai) uses it to decide whether to ask before acting.

| Level | Constant | Methods |
|---|---|---|
| 0 | `LEVEL_READ` | read, create, copy, export – all not listed |
| 1 | `LEVEL_CHANGE` | `record.update`, `record.restore`, `record.message`, `system.process_abort` |
| 2 | `LEVEL_DELETE` | `record.delete` (trash) |
| 3 | `LEVEL_MOVE` | `record.move` – can only be undone via a detour |
| 4 | `LEVEL_MASS` | `record.batch` |
| 5 | `LEVEL_ADMIN` | administrative changes (areas, fields, status, rights) – no method yet |

### Writing

Writing methods take the same path as the edit form (`RecordService`): group rights,
another admin's edit lock (`locked`, can be overridden with `unlock` as in the form), the
admin's own still open form (`open_in_backend` – the API does not discard that working
copy), validation against the field definitions, check-in in one transaction. With
`simulate`, only the checks run.

```json
{
  "record_id": 5508,
  "data":      {"title": {"en_EN": "Summer sale", "de_DE": "Sommerschlussverkauf"},
                "online_at": "2026-10-01T10:00:00+02:00", "price": "19.90"},
  "relations": {"tags": {"add": [3014], "remove": [3056]}, "categories": [2681]},
  "files":     {"image": {"serverfile": "fotos/sale.jpg"},
                "pdf":   {"content": "<base64>", "filename": "flyer.pdf"},
                "old":   null}
}
```

* `data`: values in neutral format (`ApiValues::input()`), multilingual as
  `{code: value}` – only languages with write permission; a single value applies to the
  primary language. Points in time with or without offset (without: the admin's time
  zone), wall time without offset, decimal point, amounts as `"19.90"`.
* `relations`: a list replaces all relations; `set`, `add`, `remove`, `order` (only
  fields with manual sorting) can be combined.
* `files`: `serverfile` (below `var/upload`), `content` (base64, at most
  `api.base64_max_bytes`), `tmpfile` (below `var/tmp`), `null` removes; multilingual
  `{code: …}`. The API deliberately does not fetch files from a URL.
* Tree and order via `parent_id` and `previous_id`, the owner via `owner_id` (right
  "change owner"). Embedded content (`link`) is changed via its own `record_id`.

## From templates and custom tools

```php
$result = $cms->api('record.search', ['area_id' => 8, 'filter' => ['status_id' => 1]]);
if ($result['ok']) foreach ($result['data']['rows'] as $row) { … }
```

`Cms::api()` uses the running API session if there is one (scheduler, CLI, PHP API).
Otherwise the call works along in the loaded backend session – channel `tool`, detached
again afterwards: a trigger template running on save in the backend thus acts as the
admin who saves. Without an admin session (frontend), the API answers `not_logged_in`. A
scheduler script (type `php`) takes the session from `CAMINDO_API_SESSION`
([07](07-media-processing.md#scheduler-module-scheduler)).

## Command line

```bash
php Core/cron/camindo.php api login --admin=NAME [--lifetime=SEC]    # prints session_id
export CAMINDO_API_SESSION=…                                         # or --session=…
php Core/cron/camindo.php api whoami
php Core/cron/camindo.php api methods
php Core/cron/camindo.php api describe system.describe
php Core/cron/camindo.php api call system.describe '{"method": "system.whoami"}'
php Core/cron/camindo.php api call METHOD --params-file=params.json
php Core/cron/camindo.php api logout
```

Output is JSON; the exit code is 1 if the API reports an error. The admin's group needs
the interface "command line".

## Custom API methods

A module lists its API classes in `install/settings.php` (relative to the module
namespace) and marks the callable methods with the attribute:

```php
// Modules/Blog/install/settings.php
'api' => ['BlogApi'],
```

```php
namespace Camindo\Modules\Blog;

use Camindo\Core\Api\{ApiContext, ApiException, ApiMethod};

class BlogApi
{
    public function __construct(private Blog $blog, private ApiContext $context) {}

    /**
     * @param int $postId  id of the post
     */
    #[ApiMethod('blog.post', '#blog.api_post')]
    public function post(int $postId): array
    {
        $data = $this->blog->load($postId);
        if ($data === null) throw $this->context->exceptionFromCms(ApiException::NOT_FOUND);
        return ['post_id' => $postId, 'title' => $data['title'],
                'changed_at' => $this->context->isoTime($data['changed_at'])];
    }
}
```

The texts live in `Modules/Blog/backend/lang/{language}.php`:

```php
'api_post' => 'Read a post.',
'api__post_id' => 'Post (post_id)',
```

After changing `settings.php`: `camindo.php refresh`.

Rules:

* **No logic of its own next to the interface.** What backend and API both need belongs
  in a service class without request and output – for records `RecordService`
  ([05](05-backend-development.md#structure-of-a-module)). The API class checks
  parameters and translates results and errors.
* **Rights as in the backend** – via `Access` and the dataset classes; `Cms::select()`
  restricts queries to the group's instances and areas anyway. Pass parameters on to
  `select()` only via a whitelist: never `sql`, `request` or `debug`.
* **Throw errors** (`ApiException`), don't return them; notes via `ApiContext::warn()`.
* **Neutral values:** integers without separators, decimals with a point, JSON as JSON,
  text as text. Points in time stored in UTC as ISO 8601 with offset in the admin's time
  zone (`ApiContext::isoTime()`), wall time (`datetime`) without offset. Languages as
  codes (`de_DE`, `ApiContext::languageId()` / `languageCode()`).
* **Translatable descriptions:** the description in the attribute is a snippet
  (`'#blog.api_publish'`); the parameters are described by the snippets
  `blog.api_publish__{parameter}` or – shared by all methods of the component –
  `blog.api__{parameter}` (e.g. `api__record_id`), preferably in `backend/lang/`. If a
  snippet is missing, the `@param` line of the docblock applies; a plain text instead of
  a snippet in the attribute stays untranslated.
* **Declare the severity:** `#[ApiMethod('blog.publish', '…', write: true, level:
  ApiMethod::LEVEL_CHANGE)]`. Without it, `LEVEL_READ` applies – the assistant then runs
  the method without asking.
* **Offer a trial run:** a parameter `bool $simulate = false` (only check, change
  nothing) makes the assistant's confirmation more meaningful – it calls the method with
  `simulate` first and shows the result.
* **Respect open forms:** code that changes a record without a form checks with
  `RecordService::loadForChange()` – if the same admin still has a form open (for
  example while chatting with an LLM), the API answers `open_in_backend` instead of
  discarding that working copy.

## Assistant (add-on module `camindo/module-ai`)

The add-on module Ai is an LLM chat in the backend's side panel. The model works
through the API: every method is a ***tool*** – name with `__` instead of the dot
(`record__search`), schema and English descriptions from `Api::describe()` – called
after
`ApiSession::attach('llm')` in the admin's backend session. So it sees only what the
admin sees, and every change appears in the audit log under the admin's name with the
origin `llm`. From level 1 on ([severity](#severity)), the assistant runs a method only
after the admin has confirmed it; the confirmation shows the record (`record_id`),
before/after of the fields in `data` and – if the method has `simulate` – the result of
a trial run. Outside the API there are two UI tools: `ui__views` lists the
backend views the admin may open (catalogs `install/views.php`,
[05](05-backend-development.md#view-catalog-installviewsphp)), `ui__open` opens one of
them – or directly a record's detail view, a filtered list or an edit form – as a layer;
the server builds the address from checked values. What cannot be done through the API
the assistant opens this way and guides the admin; questions about using the system it
answers via `docs.search`/`docs.read`.

For custom API methods this means: they are available to the assistant automatically –
with their (English) description as instructions. A
missing or too low severity lets it act without asking. Providers, settings and
structure are described in the module's documentation.

## Limits

* Frontend requests do not load the backend snippets; messages from backend snippets
  appear there as keys.
* Without a session there is no API – not even in trigger templates triggered by a
  frontend user (`CMS:USER_INPUT`).
