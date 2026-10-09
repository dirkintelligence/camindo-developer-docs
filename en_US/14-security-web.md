# 14 – Security II: web

## CSRF

A logged-in user opens a foreign page; that page triggers a request to the backend. If
the browser sent the cookie along, the request would run under the user's session.
camindo sets three hurdles:

| Hurdle | Where |
|---|---|
| **Token** | `Session::csrfToken()` = HMAC over the session id (nothing is stored). `cConfig.csrf_token`, sent by `cApp.system.csrfHeaders()` with **every** request as header `X-Camindo-Token`, alternatively the POST field `_token`, never from the query string. `Backend::backendPath()` checks it for every method except GET/HEAD – extension endpoints included. |
| **POST binding** | Every endpoint with side effects requires POST: `if (!$backend->requirePost()) …`. The UI triggers it via `replacePost()`, `openPost()`, a form or a context menu entry with `data-method="post"`. |
| **Origin** | `Backend::originAllowed()`: for POST, `Origin` (alternatively `Referer`) must match the backend's host or `admin.allowed_origins`; if both are missing, the token decides. |

Failures answer with `##accessdenied##` and are recorded in the `security` log. On top,
the session cookie has `SameSite=Lax`.

**Login dialog with an existing session:** `Start/login.php` is exempt from the token
check when a valid session already exists (e.g. logged in in a second window while the
dialog was still open here). `StartBackend::login()` then does not log in again but only
returns the token of the existing session and closes the dialog
(`cApp.layers.loginComplete()`) – it changes nothing, and a foreign page cannot read the
response. The origin check still applies.

**Rules for new code:** a GET must not change anything. Reloading a page opened via
POST turns into a GET that the server rejects – intended: a writing action should not
repeat on F5. What a `<link>` or download window needs (editor style sheets, downloading
an export file, `file.php`, `preview.php`) works without a token and may therefore only
read.

**Edit form and working copy:** a GET on `record_edit.php` (bookmark, F5, external
link, frontend editing) only shows a placeholder with an "Edit" button; only a POST
creates the working copy and the lock. Layers opened from the UI carry a marker and load
the form via POST immediately – the user notices nothing.

**Frontend:** token from `CMS:USER_TOKEN` (HMAC over the cookie `ucsrf`), mandatory for
all writing `CMS:USER_*` commands
([12](12-community.md#write-protection-token-throttling-form-guard)).

A token does not protect against script already running in the backend – it is in
`cConfig` and readable for injected JavaScript. The actual defence is keeping scripts
out: HTML filter and script policy.

## HTML filter

**Principle:** HTML in input is allowed (rich text), active content is not. No blanket
`htmlspecialchars()` when reading – values also go into URLs, e-mails, JSON and back into
forms. Instead:

**A – write filter.** `Dataset::validateValue()` passes every string value (and every
string element of `json`/`reltable`) without `strip_tags` through
`HtmlSanitizer::clean()`. Exceptions: `password` and the `template` table
(`$sanitizeHtml = false`) – script is content in a template, and code never travels from
the database to the file system anyway.

**B – read filter.** The same filter on every string value in `Cms::fetch()` (i.e.
`CMS:SELECT`, all forms and lists), in the field definitions
(`VirtualFieldsTrait::virtualSchema()`) and in the labels of the `InstanceCache`. This
also protects what bypassed A (old data, direct SQL).

**What `HtmlSanitizer` removes:**

| | |
|---|---|
| including content | `script`, `iframe`, `object`, `applet`, `frameset`, `noembed`, `noframes` |
| the tag only | `embed`, `base`, `meta`, `frame`, SVG animations and handlers |
| attributes | `on…`, `srcdoc`, `formaction`, `xlink:href` |
| URL values | `javascript:`/`vbscript:`, `data:` except raster images – after resolving entities and removing control characters |

Incomplete tags are defused; the filter runs until nothing changes any more.
`system.html_allowed_tags` optionally restricts to a list of allowed tags. Clean text
passes at no cost.

**C – contexts in the backend.** A value like `" onmouseover="…` contains no `<` – no
HTML filter sees it:

* `Backend::template()` escapes `"` in values for placeholders inside an attribute
  value.
* `Backend::jsString($value)` produces a safe JS string literal (for `<script>` and
  handlers).
* Text content that does not come from `fetch()` (request values, file names, messages
  containing user input) is escaped by the caller with `htmlspecialchars()`.

**Frontend:** `CMS:TEXT` outputs values raw on purpose (HTML is content); the read filter
applies there. In attributes, the template escapes with `(CMS:TEXT field
specialchars=true)`. Only commands that write an attribute themselves escape with
`htmlspecialchars()`: `CMS:MEDIA` (all attributes of the `<img>` tag including
`attr[…]`, not the single values via `subfield`), `CMS:USER_TOKEN`,
`CMS:USER_PASSWORD_RULES`, `CMS:XML` and `{href}` in the templates of `CMS:JUMP_*`.
For the latter, `Pagination::baseLink()` takes the `REQUEST_URI`, which comes from the
client unchecked, encodes `'` and `\` as `%27`/`%5C` (the backend also puts `{href}`
into a JS string, `cApp.layers.replace('{href}')`) and then escapes it for the
attribute. `prefix`, `suffix`, `ifempty` and `instead` stay raw for all commands –
they come from the template.

## Security headers

`SecurityHeaders::send()` runs at the end of the bootstrap for every web entry point:

| Context | Headers |
|---|---|
| backend | `X-Frame-Options: DENY`, `Content-Security-Policy: frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; script-src …`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin` |
| frontend, preview | `X-Frame-Options: SAMEORIGIN`, `Content-Security-Policy: frame-ancestors 'self'`, `nosniff`, `Referrer-Policy: strict-origin-when-cross-origin` |
| both, on request | `Strict-Transport-Security` via HTTPS, `security.hsts_max_age` (default 0 = off) |

In the frontend these are defaults: `(CMS:HEADER "Name: value")` at the top of a
template replaces a header, `(CMS:HEADER "Name:")` without a value removes it. Because of
`nosniff`, a template delivering CSS, JavaScript or JSON must set its `Content-Type`.

## Script policy in the backend

```
Content-Security-Policy: … script-src 'self' 'nonce-{nonce}' 'report-sample'
```

Scripts only run from own files or as a `<script>` with the page's nonce. No
`'unsafe-inline'`, no `'unsafe-eval'`, no `'strict-dynamic'` (jQuery would otherwise
execute any injected `<script>` of a loaded fragment).

**One nonce per page.** The backend is one page into which `cLoadContent` loads
fragments; the policy from `index.php`'s header applies. `backend-handlers.js` sends the
page's nonce as header `X-Camindo-Nonce`, and `SecurityHeaders::nonce()` takes it over
for the Ajax response – every fragment thus carries the page's nonce.

**Mark handlers instead of executing them.** `Backend::template()` rewrites handlers in
templates before placeholders are replaced:

```
onclick="CODE"          →  data-c9h-{nonce}-click="CODE"
href="javascript:CODE"  →  href="#" data-c9h-{nonce}-js="CODE"
<script>                →  <script nonce="{nonce}">
```

`backend-handlers.js` collects the attributes carrying the page's nonce, creates **one**
script element with the nonce that defines a function per handler, and binds them via
`addEventListener` (a `MutationObserver` picks up every inserted piece of HTML). Only
what carries the nonce in its **attribute name** becomes code – and the nonce only
appears in server code, never in content from the database. An injected `onclick`
remains an ineffective attribute.

**Rules** (short version in [05](05-backend-development.md#javascript-api-windowcapp)):

```php
// template – gets marked
<button type="button" onclick="{onclick}">…</button>

// HTML from PHP
echo '<a href="#"' . $backend->on('js', "cApp.layers.replace('X/y.php?id={$id}')") . '>…</a>';
echo '<select' . $backend->on('change', 'cApp.forms.updateElement(this)') . '>';
'attr' => ['onchange' => '…']                      // formField(): gets marked

// script in the Ajax response
$backend->script("$('#x').focus();");
echo '<script' . $backend->nonceAttr() . '>…</script>';
```

Not possible: raw `onclick` or `javascript:` in an `echo` or as an entire placeholder
value, `eval`/`new Function`/`setTimeout('string')`, HTML with handlers via
`innerHTML`.

`security.backend_csp`: `enforce` (default) | `report` (everything runs, violations are
only reported – for components being refactored) | `off`. Violations are reported by
`backend-handlers.js` to `Start/csp_report.php`; log `var/log/{date}_csp.log`.

## Script policy in the frontend

Can be switched on per instance (instance form, `instance.details`; default
`security.frontend_csp = off`): `csp` = `off` | `report` | `enforce`, additional script
sources, `'strict-dynamic'`, `form-action`. `SecurityHeaders::cleanSources()` checks the
sources – host expressions only, no keywords, no bare schemes.

```
script-src 'self' 'nonce-N' [sources] ['strict-dynamic']; object-src 'none'; base-uri 'self';
frame-ancestors 'self'[; form-action 'self' …]; report-uri /csp_report.php
```

* **The template programmer sets the nonce:** `<script(CMS:NONCE)>`. The compiler
  inserts nothing; content from the database never carries a nonce – exactly the
  separation this is about. The parser reports `<script>` without a nonce, `on…`
  attributes and `javascript:` as notes.
* `(CMS:CSP script=… form=… mode=…)` adds sources or sets the mode for one page (before
  the first output).
* Only scripts are restricted; if you want more, write the whole policy with
  `CMS:HEADER` and get the nonce with `(CMS:NONCE format=value)`.
* Reports go to `public/csp_report.php` (`CspReport`: POST only, small, throttled).

## Uploads and active content

An SVG or HTML file under `/media` lives in the website's origin; opened directly, the
browser runs embedded script. Three layers:

| Layer | Effect |
|---|---|
| upload check | blocked extensions (`php`, `phar`, `htaccess` …) always; the extension must match the detected MIME type (`error_file_mismatch`); the group `image/*` does not match SVG |
| frontend uploads | active content (HTML, SVG, XML/XSL, JavaScript – extension **or** MIME type) is always rejected, regardless of the field settings |
| delivery | `nosniff` for all files; active content additionally with `Content-Security-Policy: sandbox` and `Content-Disposition: attachment` (instance `.htaccess`, `MediaDelivery`, `backend/file.php`) |

## ImageMagick

ImageMagick detects the format by content, not by extension – an `image.jpg` with script
content would be processed as a script format. The `Magick` processor protects against
this:

* **Coder prefix:** source and target are passed with the coder of the MIME type
  detected on the server (`jpeg:/path/file[0]`); an unknown type is `unsupported`.
* **Own policy, always active:** `MagickPolicy` writes `var/config/magick/policy.xml`
  and sets `MAGICK_CONFIGURE_PATH` for all calls. It can only tighten the system policy:
  limits for memory, time, dimensions and frames; blocked are script and pseudo formats
  (MSL, TEXT, URL, FILE …), the PostScript family and all delegates.
* **SVG and PDF only on request:** `security.magick_allow = ['svg']` or `['pdf']` (PDF
  needs Ghostscript). The converter detail view shows by test run what actually works.
* **Version warning** below ImageMagick 7.1.1 in the converter view.

The policy applies to the programs, not to the PHP extension Imagick – an add-on module
using Imagick works under the system policy. If you call ImageMagick yourself, call
`MagickPolicy::apply()` first.

## Fixed backend domain

`admin.backend_domain` (e.g. `cms.example.com`) binds the backend to one domain – a
prerequisite for passkeys, and a domain that no website reveals keeps login attempts
away.

| Request | Behaviour (`BackendDomain`) |
|---|---|
| backend entry under another domain | 404 (`enforce()` in the bootstrap), no hint at the right domain |
| frontend under the backend domain | 302 to the backend if the domain belongs to no instance |
| `/backend/assets`, `/backend/ext` | under every domain (static files) |

`BackendDomain::url($path)` builds absolute backend addresses; without a fixed domain,
they stay relative.

**Preview via the website domain** (`FrontendHandover`): `preview.php` checks session
and instance permission and redirects with a signed token to the instance's primary
domain (`?c9preview=…`) – always with a fixed backend domain, otherwise whenever the
backend is not running under a domain of the instance (the media `.htaccess` would not
allow images under a foreign domain). The token contains instance, path, language, user, a marker of
the session (not the session id) and an expiry; it is only valid while the backend
session is alive and the user row is validly signed, and only under a domain of the
instance.

### Frontend editing

The `fed` cookie carries a signed value `{mode}.{admin_id}.{expiry}.{HMAC}`;
`Cms::fedActive()` checks it without a database query. The preview sets it under the
website domain; `cFED()` opens the backend via `BackendDomain::url()`.

In addition, the personal settings offer one button per website
(`AdminBackend::_settingsFedSites()`): a link `https://{domain}/index.php?c9fed={token}`
(`FrontendHandover::fedUrl()`, mode = saved setting, valid for one hour, bound to the
session like the preview). `Page::switchFedOn()` checks token and domain, sets the cookie
and redirects to `/`; invalid → 403. The buttons also exist without a fixed backend
domain, for all primary domains except the one the backend is running under.

**Syncing all websites.** A cookie for a foreign domain cannot be set reliably via Ajax
or an iframe (third-party cookies are blocked or partitioned). After saving a changed
setting – and via the button "Apply to all websites" – the browser therefore runs
through the websites as a normal navigation: `FrontendHandover::fedSyncData()` signs the
list of domains, the mode (also 0 = delete the cookie) and the return address to the
backend; the parameter `c9hops` only selects indices from that list (no open redirect).
`cApp.ui.fedSync()` first checks each domain with an image (`c9probe=1`, a 1×1 GIF only
for a matching token and domain); unreachable or foreign domains drop out of the chain.
`Page::fedSyncHop()` sets the cookie and redirects to the next website or back; the hash
of the backend URL survives the redirects.
