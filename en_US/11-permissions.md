# 11 – Permissions

Backend permissions are attached to ***user groups*** (`group`). Every user
belongs to exactly one user group. The frontend community has its own permission model
([12](12-community.md)).

## The system in brief

| Level | Maintained in | Right |
|---|---|---|
| Interface | user group → form | backend, command line (`cli`), PHP API (`php`) – through which users of the user group may log in |
| Module | user group → form | access yes/no per listed module, below it the module's **extra rights** |
| Instance | same | assigned instances; only their areas appear |
| Language | same (across instances) | no access / read / read and write – for content |
| Status | same, and status → form ("May be set by") | which statuses the user group may give a record (flat workflow); everyone may read all statuses. New user group: all |
| Area | user group → tab "Area rights" and area → tab "Access" | read · own user group: write/delete · other user group: write/delete · "extended" (change owner); template areas only read/write; folders have no rights |

User group **1 (superuser)** may do everything, has no rights rows and cannot be edited.
User group management, area rights, XML import/export and some tools are reserved for
superusers.

Derivations that apply everywhere:

* Read permission on an area = read permission on all its records; write requires read.
* The **user group of a record** is the user group of its owner (`record.owner_id` →
  `admin.group_id`); accordingly, the "own user group" or "other user group" rights apply.
* **Moving** requires write and delete in the source area and write in the target area.
* **Editing** requires, besides the write right, the right to set the record's **current
  status** (`Access::recordWrite()`): what the user group may not put into a status, it may
  no longer edit in that status either.
* **Linking** only requires read permission on the target area. Without read
  permission on at least one target area, a link field is not shown, not touched on
  save, and the record can neither be moved nor copied.
* A newly assigned instance gives the user group all its areas (full rights without
  "extended") and languages (read); a new area gets the rights of the user groups selected
  when it is created, a copied area those of the original.

## Data model

```
group                 group_id, label, details JSON {"modules": {"Record": ["batch"], "Tools": []},
                                                    "interfaces": ["backend", "php"]}, sig
rel_group_instance    group_id, instance_id                                     (row = access)
rel_group_area        group_id, area_id, group_write, group_delete,
                      other_write, other_delete, set_owner, sig                 (row = read right)
rel_group_language    group_id, language_id, can_write, sig                     (row = read right)
rel_group_status      group_id, status_id, sig                                  (row = may set)
record.owner_id       owner in the backend (FK admin, ON DELETE SET NULL)
```

`group.details.modules`: key present = module access, values = granted extra rights.
`group.details.interfaces`: permitted ***interfaces*** from `Access::INTERFACES`
(`backend`, `cli`, `php`); without the key, only `backend`.
All rights rows are **signed** – a row inserted or changed via SQL counts as not present
([13](13-security-authentication.md#signatures)).

## Runtime: group file and `Access`

`InstanceCache` writes one file per user group, `var/cache/group-{id}.php` (visible
instances, readable areas, module, area, language and status rights). `Session::load()` loads it
into `Camindo\Core\Access` on every request – after that, all checks are array lookups.
For non-superusers, the session also sets `Cms::setInstanceAccess()` and
`Cms::setAreaAccess()`: **every** query via `Cms::select()` on `area`, `record`,
`template` is thereby automatically restricted to the user group's instances and readable
areas – lists, search, lookups, exports.

```php
$access = $backend->access();            // or $app->get(Access::class)

$access->isSuperuser();
$access->module('Record');               $access->moduleRight('Record', 'batch');
$access->instanceIds();                  $access->instance($instanceId);
$access->areaRead($areaId);              $access->readableAreaIds(['record']);
$access->areaWrite($areaId, $ownerId);   $access->areaDelete($areaId, $ownerId);
$access->areaCreate($areaId);            $access->areaSetOwner($areaId);
$access->templateRead($areaId);          $access->templateWrite($areaId);
$access->languageRead($languageId);      $access->writableLanguageIds();
$access->statusWrite($statusId);         $access->settableStatusIds();
$access->recordWrite($areaId, $ownerId, $statusId);   // areaWrite() + statusWrite()
$access->interface('php');               $access->groupInterface($groupId, 'cli');   // without loading
$access->script('Tools', 'index_developer.php');
$access->denied();                       // "access denied" message and exit
echo $access->deniedHtml();              // just the message
```

### Interfaces

The API is another interface to the backend: the same users, the same rights of their
user group ([16](16-api.md)). Whether a user group may use an interface is checked at login
(`Admin::loginByName()`, `ApiSession::runAs()`) and each time a session is loaded
(`Session::load()`): if an interface is revoked from a user group, its running sessions on
that interface end with the next request. Unlike the other methods, `Access::interface()`
answers no when unloaded. Superusers may use all interfaces.

Without loaded rights (frontend, cron without a user), `Access` is **unloaded and
denies nothing** – the frontend checks via `RecordUser`, cron runs as the system.
Background jobs started by a user in the backend (import, export, batch processing,
mass deletion) load that user's rights (`Session::setCliAdmin($id, withAccess:
true)`); otherwise every check could be bypassed via "run as background process".

The group files are rebuilt automatically on every change to user groups, instances,
languages, areas and rights; manually via `InstanceCache::rebuild()` or Tools →
Developer.

## Enforcement in two layers

1. **UI** – lists, forms, buttons, trees and selection lists only show what is allowed
   (`Backend::access()`). Only readable languages appear as tabs; a language without
   write permission shows its value as grey text instead of an input and is not listed
   in `shown` – its values stay untouched on save.
2. **Data layer** – `Dataset::accessCheck()` in `save()`, `softDelete()`, `restore()`,
   `purge()`, `checkOut()`, `tmpInit()`, `movePrepare()` refuses independently with
   `error_access_denied`. `record` and `template` are checked; the other tables (Admin,
   Field, Area …) are protected by module access. `save()` deliberately does not check
   language rights – the form's `shown` mechanism takes care of that. `save()` does check
   the **status** of a `record`: if `status_id` stays the same, there is nothing to check;
   if it changes, the user group must be allowed to set the new status
   (`error_status_denied`). Editing a record in a status the group may not set is blocked
   by `RecordService::loadForAction('edit')` (form, batch processing, API) and by the UI
   (button greyed out, no "Edit" in the context menu).

In addition, `Backend::backendPath()` checks `Access::script()` for **every** script:
module access, the script's extra right (`script_access`), exceptions (`shared`,
`personal`). A refusal appears as a message in the layer.

## Extra rights

A listed module defines rights that a user group gets individually in addition to module
access – in `install/settings.php`:

```php
'access' => [                          // internal name [a-z0-9_]+ => label snippet
    'publish' => 'blog.access_publish',
],
'script_access' => [                   // script => required extra right
    'blog_publish.php' => 'publish',
],
'shared'   => ['blog_list.php'],       // may be called by other modules as a lookup (src_field set)
'personal' => ['blog_prefs.php'],      // may be called by every admin, even without module access
'entries'  => ['index.php', 'index_archive.php'],   // tile entry points, the first permitted wins
'superuser' => true,                   // module for superusers only
```

Snippet convention: `{module}.access_{right}`. In code:
`$backend->access()->moduleRight('Blog', 'publish')`. After changing `settings.php`:
`camindo.php refresh`.

Extra rights in the core (excerpt):

| Module | Rights |
|---|---|
| `Record` | `import_csv`, `import_media`, `export_csv`, `import_docx`, `export_docx`, `batch`, `personal_data` |
| `Admin` | `admin_read_group`, `admin_read_other`, `admin_create`, `admin_change` |
| `Tools` | `server`, `database`, `data`, `developer` (one per tab) |

## Owner

Every record belongs to a user (`record.owner_id`): when created in the
backend, to the session user; in the frontend, to the `owner` parameter of
`CMS:USER_REGISTER` / `CMS:USER_INPUT` or `user.default_owner_id`. Only users with
"extended" in the area may change it (`areaSetOwner()`) – and only to users of their
own user group (superusers: anyone); otherwise `Record::hookOnSave()` restores the previous
value.

## Testing permissions

Create a user group, put a test user into it, call `Session::create($adminId)` – then
query `Access` directly or test via HTTP with the `sessid` cookie against
`/backendModules/…`. Remove the test user and the session afterwards.
