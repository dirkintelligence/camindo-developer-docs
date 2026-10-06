# 07 – Media and background processes

## Terms

| Term | Meaning |
|---|---|
| ***Processor*** | the implementation – a class that converts files (`ImageGd`, `Magick`, `FFmpegVideo` …) |
| ***Converter*** | a **configured instance** of a processor: a row in `converter` |
| ***Thumbnail field*** | a `file` field with `display_type = thumbnail` and `parent_field_id` pointing to the field it is generated from |

A processor alone does nothing. Only the converter defines which media types it reacts
to and with which settings; a thumbnail field refers to one or more converters.
Processors only run when a record is saved with a new file (or via Tools → Data
"Regenerate thumbnails").

## Core processors

| Processor | Type | Based on |
|---|---|---|
| `ImageGd` | image | PHP GD – simple image editing without external programs, no background mode |
| `Magick` | image | ImageMagick (`paths.magick`) – also TIFF, PSD, HEIC, AVIF; always under its own security policy ([14](14-security-web.md#imagemagick)) |
| `FFmpegVideo` | video | ffmpeg – video conversion |
| `FFmpegAudio` | audio | ffmpeg – audio conversion |
| `FFmpegImage` | image | ffmpeg – still image from a video |

## Writing a processor

```
Processors/MyProc/                 (core: Core/Processors/…)
├── MyProc.php                     extends Camindo\Core\MediaProcessor
├── install/settings.php           version, type (image|audio|video|document|other), label, description, requires
├── lang/{locale}.php              prefix proc.myproc
└── backend/
    ├── form.php                   insert for the converter form
    ├── info.php                   insert for the converter detail view
    ├── html/*.html                prefix proc.myproc
    └── lang/{locale}.php
```

Namespace: `Camindo\Processors\{Name}` (add-on) or `Camindo\Core\Processors\{Name}`.
An add-on processor **overrides** a core processor of the same name.

```php
namespace Camindo\Processors\MyProc;

use Camindo\Core\MediaProcessor;

class MyProc extends MediaProcessor
{
    protected function hookDefinition(): void
    {
        // suggestions for the converter form
        $this->defaults['media_types_incl'] = "image/jpeg\nimage/png";
        $this->defaults['media_types_excl'] = '';

        // own settings – stored in converter.details (JSON)
        $this->schema = [
            'details.width' => ['type' => 'integer', 'label' => '#proc.myproc.field_width',
                                'default' => '300', 'min' => 0, 'suffix' => 'px'],
        ];
    }

    public function convert(string $srcPath, array $srcData, string $destPath,
                            array $details, bool $forceBgproc): array
    {
        // …
        return ['size' => $size, 'width' => $w, 'height' => $h,
                'mime_type' => 'image/webp', 'extension' => 'webp'];
    }

    // hooks of the converter record are passed through:
    public function hookOnSave(array &$fields, array &$data, array &$previousData): void {}
    // likewise hookOnCreate, hookOnLoaded, hookOnSaved, hookOnDelete
}
```

| Parameter of `convert()` | Meaning |
|---|---|
| `$srcPath`, `$destPath` | source and target, **relative to `paths.media_dir`** |
| `$srcData` | metadata of the source (`mime_type`, `extension`, `width`, `height` …) |
| `$details` | the converter's settings |
| `$forceBgproc` | force execution as a background process |

Return value: the target file's data; on error `['size' => 0, 'errors' => '…']`;
`['unsupported' => true]` if the processor cannot handle the file – the next converter
of the thumbnail field then gets its turn. Additional information about the **source**
(`width`, `height`, `data`) is taken over.

Call external programs only via `Execute` with an **argument array**, never via a shell
string. File names in the command come from the CMS, not from user input.

Registration: `php Core/cron/camindo.php component install MyProc --activate`. The
runtime list `var/config/components/processors.php` then contains the processor, and
the converter form offers it.

## Converters

| Column | Meaning |
|---|---|
| `processor` | processor name |
| `type` | `image`, `audio`, `video`, `document`, `other` |
| `media_types_incl` / `media_types_excl` | patterns the converter reacts to or not (one per line) |
| `extension` | extension of the target file |
| `details` | JSON with the processor's settings |

Patterns (`VirtualFieldsTrait::_matchMediaType()`): exact MIME type (`image/jpeg`),
wildcard (`image/*` – does not match SVG) or extension (`.pdf`).

The converters of a thumbnail field are sorted (`rel_field_converter.position`);
thumbnail generation takes the first one whose pattern matches and that can process the
file. `Converter::settingsHash()` (processor, extension, media types, `details`)
recognises converters across databases – the XML import uses it for mapping.

`details.background_process = 1` sends the conversion to a background process.
`ImageGd` has no background mode: if an image is created in the background (e.g. by
`FFmpegImage`), further versions of it can only be created with a processor that
supports background mode (`Magick`).

## Thumbnail generation

```
upload field  (file, upload)
   └── thumbnail field A  (file, thumbnail, parent_field_id = upload)
          └── thumbnail field B  (parent_field_id = A)      cascade possible
```

Sequence on save:

1. `secureUploads()` checks the file (blocked extensions, MIME type matches the
   extension) and stores it immediately under the working copy's id – marked as
   `temporary`, `thumbnails_pending`.
2. On check-in, `promoteTmpFiles()` renames the files to the `record_id`, writes
   `data_fil` and increments `version` (cache buster `?v=…`).
3. `writeDeferredThumbnails()` creates the thumbnails with the final paths – images
   immediately, videos in the background as configured – recursively along the
   cascade; every field at most once per operation. If one step was created in the
   background, all following steps run in the background too.

This way an upload survives a failed validation, and the expensive conversion runs only
once.

## Storage and delivery

```
public/media/{instance_id}/.htaccess                  domain restriction, nosniff, sandbox for active content
public/media/{instance_id}/{field_id}/{record_id}[_{language_id}].{ext}
public/media/{instance_id}/{field_id}/.htaccess       only with details.user_auth: Require all denied
public/media/default/                                 placeholder images
```

`MediaAccess` writes the `.htaccess` files (after every change in the Instance module;
for protected fields when the field is saved); if an area changes its instance,
`MediaAccess::moveAreaMedia()` moves the directories. The domain restriction only lets
through requests via the domains of the instance and of the instances allowed to read
it. It checks the `Host` header and prevents hotlinking – it is not access control;
that is what `user_auth` is for ([12](12-community.md#protected-media)).

The same `.htaccess` contains the rule for **readable file names**: `CMS:MEDIA` writes
the original file name with a comma in front of the extension
(`/media/1/12/1977,beach-north.jpg`, `Cms::mediaUriName()`); the rewrite rule drops the
addition.

**The backend never links directly** into the media directory, but always to
`backend/file.php?file={field_id}/{record_id}[/{language_id}]&v={version}`
(`RecordBackendServerFile::filePreview()`): a redirect to the file if it is
productive, unprotected and a web raster image, otherwise delivery by the script – only
with read permission on the area. In the frontend, `public/file.php` (`MediaDelivery`)
delivers protected files and downloads (`CMS:MEDIA auth=true`, `download="name"`).

For files that are not web-capable and for versions currently being created in the
background, `Backend::defaultImg()` returns a placeholder image. Web-capable are
`image/jpeg`, `image/png`, `image/gif`, `image/webp`, `image/bmp`.

Files too large for an HTTP upload can be placed in `var/upload/` and selected in the
form.

## Background processes

### `Execute` – starting and monitoring processes

Platform-independent (POSIX and Windows):

```php
$execute->execute($command, $stdout, $stderr, $timeout);                // blocking
$execute->executeBackground($command, $stdoutFile, $stderrFile, $error); // → PID
$execute->processState($pid, $token);
$execute->terminateProcess($pid, $force, $token);
$execute->tokenizeParameters($parameters);   // string → argument array
```

`$command` is always an array; arguments are escaped correctly, no shell is involved.

### `BackgroundProcess` – table `bgproc`

```php
$bgproc->start($command, $type, $token, $stdoutFile, $stderrFile, $error);
$bgproc->setProgressByToken($token, $progress, $message);
$bgproc->setStatusByToken($token, $status);
$bgproc->list($adminId);          // reconciles stored and actual state
$bgproc->abort($token);  $bgproc->remove($token);
$bgproc->lock($type);    $bgproc->unlock();
```

Status: `STATUS_FINISHED` 0, `STATUS_RUNNING` 1, `STATUS_TIMEOUT` 2, `STATUS_FAILED` 3.
A background process reports its progress via its **token** – without a session.
`abort()`/`remove()` only act on processes from `bgproc`, and `Execute::matchToken()`
checks the command line against the token so that a recycled PID does not hit the
wrong process. The command contains the placeholder
`BackgroundProcess::TOKEN_PLACEHOLDER`; `start()` inserts the token.

In the backend, `cApp.polling` queries the list as long as something is running and
shows a badge on the hourglass in the header.

Background tasks started from the backend (mass deletion, batch processing, import,
export, thumbnails) pass their job as a JSON file in `var/tmp/` and run with the rights
of the user who started them (`Session::setCliAdmin($id, withAccess: true)`); the
result arrives as an internal message.

## Cron

```
* * * * *  php /path/to/project/Core/cron/cron.php
```

This is the **only** cron job.

`cron.php` starts `TriggerCron::run()`, a **self-locking loop**: a lock file prevents
parallel runs; after almost ten minutes it ends and the next cron call takes over. In
maintenance mode it exits immediately.

| Interval | Task |
|---|---|
| `tick2s()` | triggers for background conversions (`trigger_ff.php`, `trigger_magick.php`) |
| `tick1m()` | scheduler (`scheduler.php`), template sync (`template_sync.php`), heartbeat for the operation notices. Everything else that recurs – maintenance included – is a scheduler task |

### Triggers

`Core/cron/TriggerMedia.php` is the common base of all triggers. There is **one**
trigger per `bgproc.type` (the progress format depends on the tool, not on the
processor): `TriggerFFmpeg` reads ffmpeg's `-progress` file, `TriggerMagick`
ImageMagick's `-monitor` output. Per entry: record the progress, stop overdue
processes, on completion `MediaProcessor::rectifyParams()` (metadata of the finished
file), add `data_fil` (`VirtualFieldsTrait::promoteBackgroundThumbnails()`), start
thumbnails based on it, remove the entry.

Which processor may read the finished file is stored as `proc` in `bgproc.message` –
put there by `MediaProcessor::startBackgroundProcess()`, which processors use instead
of `BackgroundProcess::start()`.

A new processor using an existing progress format needs nothing on the trigger side:
set `BGPROC_TYPE` accordingly, implement `rectifyParams()`, start with
`startBackgroundProcess()`. A new format needs a subclass of `TriggerMedia`
(`BGPROC_TYPE`, `PROGRESS_PREFIX`, `DEFAULT_PROCESSOR`, `readProgress()` →
`['percent' => int, 'finished' => bool]`), a CLI script following `trigger_ff.php` and
a call in `TriggerCron::tick2s()`.

## Scheduler (module Scheduler)

Scheduled tasks in the `scheduler` table:

| Type | Task |
|---|---|
| `url` | fetch a URL (optionally with HTTP basic auth) |
| `php` | run a script from `var/custom/` on the command line |
| `dbbackup` | `mysqldump` to `var/dbbackup/` with rotation (`Backup`) |
| `logcleanup` | clean up log files and login attempts according to retention periods |
| `template` | run a frontend template with `template.type = 'scheduler'` (`Page::renderScheduled()`) |
| `fulltext` | rebuild the full text of the areas in `details.area_ids` (empty = all except folders) (`RecordService::rebuildFulltext()`); stops at `max_minutes` |
| `cachecleanup` | clean up the `CMS:INCLUDE` output cache (`IncludeCache::cleanup()`) |
| `trash` | empty the trash (`ToolDatabase::purgeTrash()`, `admin.trash_retention_days`), lock `trash`, audit origin `trash` |
| `tmpcleanup` | `var/tmp` (`admin.tmp_retention_hours`; `*.lock`, `*.stamp` stay) and `RecordExport::cleanup()` |
| `privacy` | `PrivacyJobs::runDue()` in rounds of `RUN_LIMIT` as long as a round was full and without errors |
| `retention` | `IpAnonymizer::run()`, `AuditLog::purge()`, `Message::purgeExpired()`, `RecordUser::purgeUnconfirmed()` – each step on its own |
| `auditverify` | `AuditLog::verifyAll()` + `storeVerifyResult()` (for `SystemHealth`); findings = error |
| `updatecheck` | `UpdateCheck::run()`; nothing to do without `paths.composer` |

The last six (`Scheduler::MAINTENANCE_TYPES`) have no `details`; they used to be
hard-wired in `TriggerCron` or a separate cron job. The installer creates
`Scheduler::DEFAULT_TASKS` (`Scheduler::createDefaults()`, step `tasks` after `runtime`,
because `adminError()` needs the group files): spread between 2 and 4 a.m., `tmpcleanup`
hourly, label as the snippet `#scheduler.label_type_{type}`. `camindo.php scheduler
defaults` adds missing types; `scheduler list` and `scheduler run ID|TYPE` serve
operation and testing. `SystemHealth` reports maintenance types without an enabled task
(check `maintenance`).

Timing uses cron syntax (minute, hour, day, month, year, weekday).
`TriggerCron::tick1m()` starts `cron/scheduler.php`; `SchedulerRunner::dispatch()`
starts every due task as a separate process and records result and message in
`last_*`.

**As which user.** Every task has a user (`scheduler.admin_id`, by default the user
`scheduler` created by the installer, `Installer::createSchedulerAdmin()`).
`SchedulerRunner::run()` opens a session of its own for them – `ApiSession::runAs()`,
interface `cli`, channel `scheduler` ([16](16-api.md)) – and closes it afterwards. Their
user group rights apply, and the audit log shows them with the origin `scheduler`. A `php`
script receives the session as environment variable `CAMINDO_API_SESSION`:

```php
require '/path/to/project/load_camindo.php';
$app->get(\Camindo\Core\Api\ApiSession::class)->resume(getenv('CAMINDO_API_SESSION'), 'scheduler');
$result = $app->get(\Camindo\Core\Api\Api::class)->call('record.search', ['area_id' => 8]);
```

A template runs in the same process within this session. A `url` call never gets the
session. Without a valid user (deleted, blocked, user group without `cli`,
`Scheduler::adminError()`), the run fails.

**Type `template`.** `Page::renderScheduled()` sets up the environment of a page request
without a record: the template's instance plus its readable instances
(`Cms::setInstanceAccess()`), the content language from `details.language_id` (0 =
primary language) with time zone, snippet language and locale. `$set['last_run']` /
`$set['this_run']` (time zone of the language, plus `_utc`) form the time window
`(last_run, this_run]`. Its end is stored in `details.window_end` (UTC) after every
**successful** run – the table's `last_run` cannot be used, `dispatch()` already
overwrites it at the start. A run with messages in `Cms::$errors` counts as failed; the
window then stays where it was. Protection: `Page` always answers a template with
`type = 'scheduler'` requested as an address (`/0,name`) with 404, and
`Template::saveAccess()` sets `standalone = 0` for such templates.

## Configuring external programs

```php
'paths' => [
    'magick'    => '/usr/bin/magick',     // ImageMagick 6: convert
    'ffmpeg'    => '/usr/bin/ffmpeg',
    'php'       => '/usr/local/bin/php',  // starts the triggers
    'mysqldump' => '/usr/bin/mysqldump',
],
```

`TriggerCron` checks `is_executable(paths.php)` and stops with a message if the path
is wrong. The installer looks for the programs and suggests the paths.
