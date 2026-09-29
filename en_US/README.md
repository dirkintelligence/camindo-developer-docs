# camindo CMS – Developer Documentation

This documentation is for everyone who wants to **extend or develop** camindo CMS 9:
write modules, media processors and extensions, provide their own `(CMS:…)` commands,
understand or change the core.

If you maintain content, run an installation or write templates, you will find
everything you need in the three manuals (editors, administrators, template
programmers), available in the backend under "Manuals". This documentation assumes
their terminology and describes what lies behind it.

## Requirements

* PHP ≥ 8.1 with the extensions `mysqli`, `intl`, `gd`, `json`, `mbstring`
* MySQL ≥ 8.0.17 (JSON columns, multi-valued indexes)
* Composer 2
* optional: ImageMagick (`magick`/`convert`), `ffmpeg`/`ffprobe`, `mysqldump`
* working knowledge of PHP 8 (types, constructor property promotion), SQL and jQuery

camindo CMS is deliberately **framework-free**: bootstrap, dependency injection
container, database abstraction, template compiler, command line and backend UI are
all its own code. External dependencies are limited to a few libraries (PHPMailer,
Parsedown, and jQuery and Bootstrap in the backend).

## The basic idea in five sentences

1. ***Content areas*** (`area`) define which ***input fields*** (`field`) exist – the
   data model is put together in the backend, not programmed.
2. ***Records*** (`record`) only hold metadata; field contents are stored per type in
   `data_txt`, `data_int`, `data_fil`, `data_rel` … (EAV pattern with type
   partitioning).
3. `Cms::select()` joins these "virtual fields" at runtime – filtering and sorting feel
   like working on a real table.
4. ***Templates*** are written in a command language of their own, `(CMS:…)`,
   maintained as files under `var/templates/` and **compiled to PHP** when saved; at
   runtime, plain PHP is executed. Code never travels from the database to the file
   system.
5. ***Processors*** create derived media versions on save (thumbnails, video formats);
   long conversions run as background processes.

## Contents

| Chapter | Contents |
|---|---|
| [01 – Architecture](01-architecture.md) | Directories, bootstrap, configuration, DI container, the four request paths, backend layers, core classes |
| [02 – Data model](02-data-model.md) | Tables, EAV principle, multilingual content, trees, relations, working copies, media storage, deletion and trash |
| [03 – Dataset API](03-dataset-api.md) | The table abstraction: schema, hooks, `init/load/save/delete`, validation |
| [04 – Searching and filtering](04-select-and-filter.md) | `Cms::select()`: parameters, filter syntax, sorting, pagination |
| [05 – Backend development](05-backend-development.md) | Structure of a module, the `Backend` class, snippets, HTML templates, JavaScript API, widgets |
| [06 – Template compiler](06-template-compiler.md) | From source to PHP, command definitions, custom commands, runtime of the main commands |
| [07 – Media and background processes](07-media-processing.md) | Processors, converters, thumbnails, cron, triggers, scheduler |
| [08 – Components and extensions](08-components-and-extensions.md) | Component contract, `settings.php`, life cycle, extensions for input fields |
| [09 – Installation, update, command line](09-installation-and-update.md) | Packages, installer, migrations, core update, maintenance mode, CLI |
| [10 – Publishing components](10-publishing-components.md) | Composer packages, versioning rules, releases |
| [11 – Permissions](11-permissions.md) | User groups, module, instance, area and language rights, the `Access` class |
| [12 – Frontend community](12-community.md) | Website users, sessions, the `CMS:USER_*` commands, protected media |
| [13 – Security I: authentication, keys, audit](13-security-authentication.md) | Passwords, signatures, sessions, second factor, key rotation, audit log |
| [14 – Security II: web](14-security-web.md) | CSRF, HTML filter, security headers, script policy, uploads, ImageMagick, backend domain |
| [15 – Conventions](15-conventions.md) | Names, code style, SQL, languages, text direction, file permissions |
| [16 – API](16-api.md) | The API as an interface: sessions, channels, calls, severity levels, custom API methods, command line, assistant (add-on module) |

Newcomers are best served by reading 01 → 02 → 03 → 05; if you only want to write a
media processor or an extension, read 01 and then 07 or 08.

## License

camindo CMS is licensed under the **GNU AGPL v3 or later** (`Core/LICENSE`, notice and
additional permission in `Core/LICENSING.md`). The additional permission under section 7
excludes templates, the files they deliver and all content: they may be licensed under
any terms. Modifications of the core and components (modules, processors, extensions)
remain subject to the AGPL. Anyone running a **modified** core over a network must offer
its users the source code of that version (section 13). The licenses of third-party
libraries are listed in `Core/THIRD-PARTY-NOTICES.md`.

## Notation

* `Class::method()` denotes a method, static or not.
* Classes without a namespace live in `Camindo\Core\` (directory `Core/src/`), module
  classes in `Camindo\Core\Modules\{Module}\` (`Core/Modules/{Module}/`).
* Paths are relative to the project directory (the directory containing
  `composer.json`).
* Configuration keys are written in dot notation: `admin.max_inactive_sec` means
  `$_cfg['admin']['max_inactive_sec']` in `.camindo.php`.
* Terms in ***bold italics*** are explained at that point.

## Glossary

| Term | Meaning |
|---|---|
| **Administrator** | backend user (table `admin`) |
| **User** | website visitor with an account; the profile is a record in an area of type `profile` |
| **Instance** | a website or tenant (table `instance`) |
| **Content area** | container with its own field definition; types `record`, `profile`, `ugc`, `template`, `folder` |
| **Field** | user-defined input field of an area |
| **Meta field** | field defined in the PHP schema, a real table column |
| **Virtual field** | user-defined field; its content is stored in a `data_*` table |
| **Working copy** | temporary record (`tmp = 1`) during editing |
| **Primary language** | `instance.primary_language_id` – fallback for all multilingual fields |
| **Component** | umbrella term for module, processor and extension, each either core or add-on |
| **Processor** | implementation of a media conversion (e.g. `ImageGd`) |
| **Converter** | configured instance of a processor (table `converter`) |
| **Extension** | JavaScript enhancement for an input field in the backend |
| **Snippet** | multilingual text module, `$cms->snippet('admin.field_admin_name')` |
