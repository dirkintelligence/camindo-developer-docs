# 12 – Frontend community

In camindo, a "frontend user" is a registered website visitor with an account – not the
backend user, who is simply called "user". A frontend user has **no table of their own**:
their representation in the CMS, the profile record, is a
record in an area of type `profile`, contributions are records in areas of type `ugc`.
Reading goes through `Cms::select()`, writing through `Record` – working copy,
validation, uploads and thumbnails included.

| Class | Purpose |
|---|---|
| `Modules\Record\RecordUser` | runtime of the `CMS:USER_*` commands |
| `UserSession` | frontend sessions (cookie `usess`), CSRF token (cookie `ucsrf`) |
| `Modules\Record\UserSecondFactor` | second factor for profiles |
| `MediaDelivery` | delivery of protected files (`public/file.php`) |
| module `User` | backend view: reviewing new contributions (`record.needs_review`) |

How template programmers use the commands is described in the template programmers'
manual; this chapter describes what happens behind them.

## Area types `profile` and `ugc`

`record`, `profile` and `ugc` are all record areas (`Area::CONTENT_TYPES`) – the same
for form, lists, search and `select()`.

| Type | Additional settings |
|---|---|
| `profile` | `area.login_name_field_id` (field of type `chars` or `encrypted`), `area.login_password_field_id` (field of type `password`), `details.second_factor` |
| `ugc` | `details.profile_area_ids` – profile areas whose frontend users may write here |

| Element | Purpose |
|---|---|
| `record.user_id` | owner in the frontend = profile record; basis of the ownership check |
| `record.owner_id` | owner in the backend (user), see [11](11-permissions.md#owner) |
| `record.status_id = 2` | "unconfirmed" – after registration, until confirmation |
| `record.needs_review` | 1 for everything a frontend user creates or changes |
| `user_session`, `user_basket`, `user_token`, `user_login_attempt` | sessions, basket, one-time tokens, failed attempts |
| `user_second_factor`, `user_recovery_code` | second factor |

Passwords are stored as hashes in `data_pwd` and **never** in `record_cache`; `fetch()`
always returns password fields empty.

## Sessions

| | |
|---|---|
| Cookie | `usess` (name via `cookiename`), HttpOnly, `user.cookie_secure`, `user.cookie_samesite` (`Lax`), browser session |
| Expiry | `user.max_inactive_sec` (86400) without access; `create()` removes expired sessions |
| anonymous | `record_id` NULL – `USER_BASKET` creates such sessions for a basket without login |
| Login | `USER_LOGIN` **always issues a new session id** (against session fixation); basket and details of an anonymous or own session move over, the old row is deleted |
| End | `USER_LOGOUT` deletes row and cookie; `UserSession::closeAllOf($recordId, $exceptId)` ends all sessions of a profile (profile deleted, password changed) |

Cookies must be set before the first output – the frontend deliberately does not
buffer. The `USER_*` commands therefore belong at the top of a template.

## Write protection: token, throttling, form guard

**CSRF token.** All writing commands require a token from `CMS:USER_TOKEN` in the
request (POST field `_token` or header `X-Camindo-Token`, never from the query string):
`USER_LOGIN`, `USER_REGISTER`, `USER_PW`, `USER_INPUT` (as soon as `submit` is filled),
`USER_DELETE`, `USER_LOGOUT`, `USER_BASKET` except `load`. The token is an HMAC over the
cookie `ucsrf` (signed double submit), not bound to the session – login, registration
and reset run without a session. The first `CMS:USER_TOKEN` per request must come
before any output because it sets the cookie. It can be switched off per command
(`csrf=false`) or globally (`user.csrf`). Writing forms submit via POST;
`$_POST['submit']` is the binding way to fill `submit`.

**Throttling** (`RateLimiter`, table `rate_limit`, keys stored only as HMAC): counts
**every** request in a sliding window, limits in `user.throttle.{bucket}` =
`[maximum, seconds]`.

| Bucket | Default | for |
|---|---|---|
| `pw_value`, `pw_ip` | 3 / h, 10 / h | "forgot password" per entered value or origin |
| `register_ip`, `register_value` | 5 / h, 3 / day | registration per origin or e-mail |
| `mail_instance` | 200 / h | mails from `USER_PW` + `USER_REGISTER` per instance |
| `login_ip` | 30 / 10 min | failed logins per origin across all names |
| `login_code` | 3 / 10 min | "send new code" per profile |
| `email_ip` | 5 / h | `CMS:EMAIL throttle=true` |
| `zip_ip` | 30 / h | newly built `CMS:ZIP` archives per origin (cache hits do not count) |

`pw_value` counts the **entered** value before the lookup – otherwise only an existing
address would ever be blocked, revealing which addresses have an account. The origin is
determined by `ClientIp` (`X-Forwarded-For` only from addresses in
`security.trusted_proxies`, IPv6 shortened to the /64).

**Brute-force protection of the login:** `LoginGuard::forUsers()` – table
`user_login_attempt`, `user.bruteforce_max` (5) failed attempts per area and name
within `user.bruteforce_delay` (600 s).

**Form guard** (`FormGuard`): `(CMS:USER_TOKEN formguard=true)` outputs a signed
timestamp `_ft` and an invisible honeypot field; `formguard=true` on `USER_REGISTER`,
`USER_PW` and `CMS:EMAIL` rejects the request if the honeypot is filled or the form was
submitted too quickly (`user.formguard_min_seconds`) or too late. Extension point for an
external service: `security.form_challenge` = class with
`FormChallengeInterface::verify($_POST)`.

Unconfirmed profiles are deleted for good by cron after `user.unconfirmed_days` (7).

## The commands

Results end up in `$set`; after `USER_LOGIN`, `USER_REGISTER`, `USER_REGISTER_CONFIRM`
and `USER_PW`, the profile is also the current record `$cms` of the success template.
Errors go to `Cms::error()` and appear via `CMS:MESSAGE`; `message[index]` replaces a
standard text (snippets `user.error_*`). The standard texts follow the page's
`language.backend_code` – for languages without their own backend language, the
template sets `message[…]` itself.

Form data is named `data[field_name]` (multilingual `data[field][language_id]`), uploads
under the same name in `$_FILES` – the same shape as the backend form, so
`Record::save()` understands it directly.

| Command | Effect |
|---|---|
| `USER_AUTH` / `USER_PROTECT` | pick up the session; `PROTECT` includes `errortemplate` and stops if nobody is logged in |
| `USER_LOGIN` | login with name + password or with the reset token; asks for the second factor if required |
| `USER_LOGIN_CODE` | second login step (app, e-mail or recovery code), `resend` |
| `USER_2FA` | set up, switch, disable the second factor, new recovery codes |
| `USER_LOGOUT` | end the session |
| `USER_REGISTER` / `USER_REGISTER_CONFIRM` | create a profile (status 2) / redeem the token (status 0) |
| `USER_PW` | reset token for "forgot password" in `$set['hash']` |
| `USER_INPUT` | create or change a record (see below) |
| `USER_DELETE` | move an own record or the own profile to the trash |
| `USER_BASKET` | basket (see below) |
| `USER_TOKEN` | CSRF token, optionally with form guard |
| `USER_PASSWORD_RULES` | password policy as JSON for the `Password` extension in the frontend |

**No `USER_*` command sends e-mails itself.** Confirmation link, reset link, login code
and notices ("new device", "password changed") are sent by the success template with
`CMS:EMAIL` – the commands put tokens, codes and events into `$set`. If you want the
code by SMS, call your SMS service in the template instead of `CMS:EMAIL`.

### `USER_INPUT` – who may write what

| Area | without `ignoreuser` |
|---|---|
| `ugc` | logged in, the frontend user's profile area in `details.profile_area_ids`; change only own records (`user_id`) |
| `profile` | only the own profile; no creation (use `USER_REGISTER`) |
| `record` | never |

`ignoreuser=true` lifts **all** checks (contact form without login) – the template then
bears the responsibility. Sequence: permission → check out own record or `tmpInit()` →
take over only user-defined fields (restricted by `fields`, meta fields only via
`overwrite`) → `user_id`, `needs_review = 1` → `Record::save()` with all of the area's
checks. If saving fails, the working copy and its uploads are discarded.

**Changing the password** of a profile requires the current password
(`data[current_password]`) – unless the session was logged in via the reset link and
the permission is still valid. Afterwards, all other sessions of the profile end,
`$set['password_changed']` = 1.

**Uploads from the frontend** are checked more strictly than in the backend
(`VirtualFieldsTrait::setUntrustedUploads()`): active content (HTML, SVG, XML/XSL,
JavaScript – by extension **or** detected MIME type, `MediaAccess::isActiveContent()`)
is always rejected, even if the field allows it.

**Owner:** `USER_REGISTER` and `USER_INPUT` (when creating) set `owner_id` and
`created_by` from the `owner` parameter or `user.default_owner_id`; if that user does
not exist, nothing is saved. `changed_by` stays NULL.

### `USER_BASKET`

Basket per session; `category` separates several baskets. `function`: `load` (items as
a result set for `CMS:RESULT_START` – replaces the current result set without saving
it), `add`, `update` (quantity 0 removes), `update_addon`. Only records that are
published and visible to the request may go into the basket
(`RecordUser::basketAllowedIds()`); without `container` only from areas of type `record`
or `ugc`, with `container` only from the areas named. A shop should always set
`container`. The same check applies when reading: an item taken offline in the meantime
no longer counts.

## Second factor

`area.details.second_factor` per profile area (`UserSecondFactor::needed()`):

| Value | Effect |
|---|---|
| `none` (default) | never |
| `optional` | only with a factor the frontend user set up (`USER_2FA`) |
| `new_device` | own factor, otherwise a code by e-mail on an unknown device (`LoginDevices::isKnown()`) – the first device is unknown too |
| `required` | app code if set up, otherwise a code by e-mail |

Flow: `USER_LOGIN` checks the password, if needed creates an anonymous session holding
the pending login (`details.mfa_pending`) and includes `codetemplate`, which sends the
code from `$set['login_code']`. `USER_LOGIN_CODE` checks the code (wrong codes count in
the `LoginGuard` for the same name; after three, the pending login expires) and then
completes the login. The `LoginGuard` counter is only cleared after the complete login.
The TOTP secret is encrypted (`Crypt::encrypt`, AAD `utotp|{id}`), replay is blocked via
the last time step, ten recovery codes are stored as HMACs. Reset in the backend:
profile detail view → "Second factor".

## Protected media

Upload and thumbnail fields have the setting `details.user_auth`:

| Value | Effect |
|---|---|
| 0 | `CMS:MEDIA` links directly into the media directory |
| 1 | `MediaAccess::protectField()` locks `media/{instance_id}/{field_id}/` (`Require all denied`); `CMS:MEDIA` links to `/file.php?file={field_id}/{record_id}[/{language_id}]`; `MediaDelivery` only delivers to a logged-in user from one of the profile areas in `details.profile_area_ids` |

`file.php` also serves `CMS:MEDIA auth=true` and `download="name"`. Every failure is a
404. Active content is always sent as `attachment` with `Content-Security-Policy:
sandbox` ([14](14-security-web.md#uploads-and-active-content)).

The check itself lives in `MediaDelivery::resolveFile()` (instance, publication, field
belongs to the area, `user_auth` with profile area). `CMS:ZIP` (`CmsZip`) applies it to
every file of an archive – a ZIP only contains what could also be fetched through
`file.php` one by one.

## Configuration (`user.*`)

```
max_inactive_sec 86400   basket_days 30       token_expiration 86400
password_token_expiration 3600                login_code_expiration 600
bruteforce_max 5         bruteforce_delay 600 bruteforce_ip false
cookie_secure true       cookie_samesite Lax  csrf true
throttle […]             formguard_min_seconds 3   formguard_max_seconds 3600
unconfirmed_days 7       default_owner_id 0
```
