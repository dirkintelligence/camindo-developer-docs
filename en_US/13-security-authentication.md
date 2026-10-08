# 13 – Security I: authentication, keys, audit

The administrators' manual describes the security features from an operational point of
view. This chapter describes how they are built and what new code has to observe. Web
topics (CSRF, HTML filter, headers, script policy, uploads) are in
[14](14-security-web.md).

## Threat model

| Attacker | has | does not have |
|---|---|---|
| **DB writer** | write access to the database (leaked DB password, SQL injection in foreign code on the same host, database tool) | the file system, in particular `.camindo.key` |
| **Script in the browser** | JavaScript running in the backend or frontend (stored or reflected XSS) | the server |

Whoever has file system **and** database is the operator; nothing in this chapter helps
against that. The secret almost everything relies on is `.camindo.key`.

## `Crypt` and the key

| Method | Purpose |
|---|---|
| `encrypt($text, $salt = '', $binary = false)` / `decrypt()` | AES-256-GCM with a random IV; `$salt` binds the ciphertext to a context as AAD – copied into another row, it does not decrypt |
| `hash($value, $binary = false)` | deterministic HMAC-SHA256 with the key – search in encrypted fields, signatures, tokens |
| `passwordHash()` / `passwordVerify()` / `passwordNeedsRehash()` | passwords, see below |

Use these methods for new secrets, never home-made constructions, and bind ciphertexts
to their row via AAD (`"purpose|{id}"`).

## Secrets from the environment

Passwords and the key can be kept in environment variables instead of files (Docker,
PHP-FPM `env[…]`, the platform's secret management). A `.env` file is deliberately not
read; there is no library for it.

| Variable | replaces | Precedence |
|---|---|---|
| `CAMINDO_DB_PASS` | `db.pass` | variable |
| `CAMINDO_SMTP_PASS` | `email.smtp.pass` | variable |
| `CAMINDO_AI_API_KEY` | `ai.api_key` (add-on module Ai) | variable |
| `CAMINDO_KEY` | content of `paths.key_file` | **file** |

`Config` applies the values from `Config::ENV_SECRETS` after merging with
`.camindo.php`; `fromEnv()` names the ones that came from the environment. Reading goes
exclusively through `Config::env($name)`: the value is cached per process and then
removed from the process environment, `$_ENV` and `$_SERVER` – child processes
(`Execute`) do not inherit it, and `phpinfo()` does not show it (except Apache `SetEnv`
under "Apache Environment"). A component with a secret of its own reads its variable
through `Config::env()` as well and lets it take precedence over the value from
`.camindo.php`.

For the key, precedence is reversed: `Crypt` uses `paths.key_file` if the file is
readable, otherwise `CAMINDO_KEY` (`keySource()`: `file`|`env`). That way, after
`key rotate` – which always writes the new key to the file – the new key applies
immediately, even while the variable still holds the old one. `envKeyState()` compares
the variable with the key ring (`current`, `retired`, `other`); `key status`, `status`
and `SystemHealth` (check `keyenv`) report an outdated variable. If the variable alone
(without a file) holds a retired key, `Crypt` throws at startup: carrying on with it
would encrypt new data under the old key and treat all signatures as tampered with.

`Backup::mysqldump()` writes an option file (`0600`) under `paths.tmp_dir` for each run
when `db.pass` comes from the environment (or `.sqlpwd` is missing) and deletes it
afterwards.

## Passwords

```
passwordHash($pw)            {kid}:password_hash(base64(HMAC-SHA256($pw, key)), Argon2id | bcrypt)
passwordVerify($pw, $stored) checks in constant time; without a match against a dummy hash
passwordNeedsRehash($stored) old scheme, different parameters or a foreign key
```

* **Scheme:** `security.password_hash.algo` – `auto` = Argon2id if PHP provides it,
  otherwise bcrypt. Argon2id with 19 MiB and `time_cost` 2 (OWASP minimum), bcrypt
  `cost` 11.
* **Pepper:** the HMAC with `.camindo.key` in front makes a database dump without the
  key worthless and always gives bcrypt 44 characters.
* **Key identifier** (`{kid}:`) – required for key rotation.
* **Rehash on login:** after a successful check, `Admin::loginByName()` and
  `RecordUser` write a hash in the current format.
* **Timing:** without a match for the name, the check runs against a dummy hash
  (`var/cache/password_dummy.txt`) – an invented name does not answer faster.

Setting a password without the backend: `php Core/cron/camindo.php admin password NAME`
(`--pass-file`, `CAMINDO_ADMIN_PASS` or hidden input).

### Password policy

`PasswordPolicy` (`security.password_policy`): `min_length` 12, `min_special` 1,
`min_upper` 1 (`upper_not_first`: an upper-case letter in first position does not
count), `min_lower` 1, `min_digit` 0 – every rule can be switched off (0). Checked in
`Dataset::validateValue()` for every input into a field with `display_type password`
that carries the `Password` extension (`admin.password` always; password fields of
areas if selected in the field definition, with overrides in `extension_options`). The
extension shows strength and rules in the browser; only the server check counts. If you
add a rule, change `PasswordPolicy::count()` and the extension's script.

## Signatures

Against the DB writer, every security-relevant row carries in `sig` an HMAC over its
decisive columns (`Camindo\Core\Integrity`). A row with a wrong or missing signature is
treated as if it did not exist and noted in the `security` log.

| Table | Signature over | checked in |
|---|---|---|
| `admin` | `admin_id`, `name`, `password`, `group_id`, `status_id`, `second_factor` (if set) | login, `Session::load()` (every request) |
| `admin_session` | `session_id`, `admin_id` | `Session::load()` |
| `group` | `group_id`, `details` | building the group files |
| `rel_group_instance`, `rel_group_area`, `rel_group_language`, `rel_group_status` | all columns | building the group files |
| `admin_passkey` | `admin_id`, hash of the credential id, hash of the public key, `alg` (not the signature counter) | `Passkeys` on every read |

This detects: a changed `group_id`, swapped names or passwords, a reset status, an
injected session, self-granted rights. Rights are signed **per row** because rows can
disappear via `ON DELETE CASCADE` – deleting rows takes rights away, it never grants
any. `Session::load()` reads `group_id` and `status_id` fresh from the signed `admin`
row on every request.

**New write paths** to these tables must write the signature too
(`Integrity::*Signature()`), otherwise their rows count as tampered. Plain SQL on a
signed column invalidates the row – on purpose. Re-sign everything (after a key change,
for development data): `php Core/cron/camindo.php admin sign` – this deletes all
sessions.

## Sessions and cookies

* `Session::create()` **always** issues a new session id (against session fixation);
  from a session of the same user in the cookie, only expanded folders and open
  working copies are taken over.
* Cookie `sessid`: always HttpOnly, `Secure` according to `admin.cookie_secure`
  (default `true`), `SameSite` according to `admin.cookie_samesite` (default `Lax`), no
  expiry date; on the server, the session expires after `admin.max_inactive_sec`. Every
  request renews `access_date` – except background queries (polling of the message badge
  and of background processes: `cLoadContent({background: true})`, header
  `X-Camindo-Background: 1`); otherwise an open window would keep the session alive forever.
* HttpOnly prevents **reading** the session id via JavaScript, not requests on behalf of
  the session – the HTML filter and the script policy help against that
  ([14](14-security-web.md)).
* Working copies belong to the user; `admin_session_record` records which one belongs
  to which session. Logging out discards only this session's working copies; "log out
  on all devices" discards all working copies and sessions of the admin; "end all other
  sessions" (devices and sessions) discards the working copies of the other sessions
  (`Record::cancelOtherSessions()`) and those sessions.
* Every session belongs to one ***interface*** (`admin_session.interface`: `backend`,
  `cli`, `php`), which is part of its signature. The backend entry points only load
  backend sessions (`Session::load($id)`), the API only its own
  (`Session::load($id, ['cli', 'php'])`). Command-line and PHP API sessions expire after
  `api.max_inactive_sec` of inactivity and at the latest at `expires_at` (maximum
  lifetime, capped by `api.max_lifetime`). They have no cookie and no login device. The
  user group must be allowed to use the interface – checked at login and on every load
  ([11](11-permissions.md#interfaces)).
* Renewing the password and setting up TOTP only work in the backend login dialog; a
  login via the PHP API is refused until then (`admin.error_login_backend_first`).

## Login protection

| Protection | How |
|---|---|
| failed attempts per name | `LoginGuard` (table `admin_login_attempt`), `admin.bruteforce_max` within `admin.bruteforce_delay`; wrong second-step codes count as well; the counter is only cleared after a **complete** login |
| failed attempts per origin | `RateLimiter` bucket `admin_login_ip` (`admin.throttle.login_ip`, default 20 per 10 min) – against password spraying across many names |
| lock-out by others | a browser with a known device cookie of the account counts under its own key – knowing the name is not enough to lock the account |

### Login devices

`LoginDevices::register()` runs after every complete login (backend and frontend):
cookie `c9dev` (random value, HttpOnly, `security.device_days`), stored in the
`login_device` table only as an HMAC. A new device for an account that already has
devices triggers the `new_device` event. A known device **does not skip any factor** –
it is a recognition feature, not a login secret.

Notifications go to `LoginNotifierInterface::notify($event, $kind, $account,
$context)`; the class comes from `security.login_notifier` (default `LoginNotifier`:
e-mail to users; nothing for frontend accounts – the template sees the events
in `$set`). Events: `new_device`, `password_changed`, `totp_enabled`, `totp_disabled`,
`code_enabled`, `code_disabled`, `recovery_code_used`. A custom notifier (SMS, push) can
request the default one in its constructor and call it as a fallback.

Users see their devices and sessions under `Admin/devices.php` and can remove
them individually; `admin_session.device_id` makes sure a device's sessions end with it.

## Second factor in the backend

`Camindo\Core\Modules\Admin\SecondFactorInterface`:

```php
interface SecondFactorInterface {
    public function name(): string;                        // 'email' | 'totp' | …
    public function start(array $admin): bool;             // after the password
    public function verify(array $admin, string $code): bool;
}
```

| Implementation | Code |
|---|---|
| `EmailFactor` | 5 digits by e-mail, one attempt per code |
| `TotpFactor` | 6 digits from an authenticator app (RFC 6238, no library: `Totp`) or a recovery code |
| `WebAuthnFactor` | passkey or security key (WebAuthn); alternatively the app code (if TOTP is active) or a recovery code |

`Admin::secondFactor()` chooses per user; `security.admin_2fa` = `email` | `totp`
(default, optional) | `totp_required` (superusers must). The login has two stages:
after the password, a one-time `uid` in the form that exactly one verification attempt
may redeem.

* TOTP secret encrypted with AAD `totp|{admin_id}`; replay is blocked via
  `UPDATE … WHERE totp_step < ?`.
* Setup without intermediate storage: an encrypted setup token travels with the form
  (valid for 15 minutes); only a valid code writes the secret. The QR code is generated
  in the browser – no third-party service sees the secret.
* Ten recovery codes, stored as HMACs in `admin_recovery_code`; redeeming = `DELETE`.
* `admin.second_factor` is part of the signature: downgrading via SQL invalidates the
  row.

### WebAuthn

Passkeys as a second factor, without a library: `Camindo\Core\WebAuthn` builds the options
for `navigator.credentials.create()/get()` and verifies the responses (CBOR, COSE → PEM;
ES256, EdDSA via libsodium, RS256 from 2048 bits). Attestation is requested as `none` and
not evaluated – what counts is possession of the key, not where the device comes from.

* **Requirement:** a fixed backend domain ([14](14-security-web.md#fixed-backend-domain)).
  RP ID = `admin.backend_domain` without port, origin `https://{domain}`. If the domain
  changes, all passkeys become useless. Can be switched off with `security.webauthn = false`,
  independent of `security.admin_2fa`; a passkey also satisfies `totp_required`.
* **Which factor:** `WebAuthnFactor::applies()` comes before TOTP and e-mail – with
  `second_factor = 'webauthn'` (passkeys only) or whenever passkeys exist (also alongside
  TOTP; the form then accepts both). The `webauthn` marker is part of the user row's
  signature: deleted passkey rows do not downgrade the account, the recovery codes remain.
* **Sign-in:** the challenge is `Crypt::hash('webauthn-login|' . uid)` – the `uid` is random
  and good for exactly one attempt, and so is the challenge; no column is needed. The form
  sends the response as JSON in the `passkey` field (`cApp.ui.passkeyLogin()`),
  `loginByCode()` passes the `uid` on to `verify()`. Checked: type, challenge, origin,
  `crossOrigin`, `rpIdHash`, UP flag, signature over `authData ‖ SHA-256(clientDataJSON)`;
  the signature counter must increase (unless both are 0 – synced passkeys keep none). UV
  is not required, the passkey is the second factor after the password. Failed attempts
  count in `LoginGuard`.
* **Setup** (`AdminBackend::totp()`, steps `passkey_*`): name and current password first,
  then the gesture on the device – so a wrong password never leaves a passkey on the device
  that the server does not know. Challenge and name travel encrypted as a token in the form
  (AAD `webauthn-reg|{admin_id}`, 10 minutes). The first passkey sets
  `second_factor = 'webauthn'` (unless TOTP) and creates recovery codes if there are none.
  `user.id` in the passkey is an HMAC of the `admin_id`, not meaningful.
* **Storage:** `admin_passkey` (credential id, hash of the id as unique key, PEM, `alg`,
  counter, transports, name), each row signed (`Integrity::passkeySignature()`, also in
  `signAll()`), at most 20 per account (`Passkeys`).
* **Removing** requires the password; without passkeys and TOTP the own factor ends and the
  codes expire. "Reset second factor" by a superuser removes app and passkeys. Events
  `passkey_added`, `passkey_removed` go to the notifier and the audit log.
* **Self-test:** `php Core/cron/check_webauthn.php` – a software authenticator registers and
  signs in with all three algorithms, plus the failure cases.

Open: sign-in without a password (`residentKey: preferred` is already requested) and
passkeys for frontend profiles.

## Password obligations for users

* A password set by **another** user (creation, reset) must be changed at the next
  login (`admin.password_change`).
* `admin.password_retention_days` > 0 requires a change after that period.
* Only someone who knows the current password can change their own; every change ends
  the account's other sessions.
* The extra rights `admin_create` (create with an initial password) and `admin_change`
  (change, lock) only apply to the own user group; only a superuser sets the password of an
  existing other admin – and only with their **own** password in the field
  `password_current` (`AdminBackend::_editPasswordCurrentOk()`, failed attempts count in
  the `LoginGuard`). An unattended open session is thus not enough to take over either the
  own account or someone else's. An account without a password (created via CLI only)
  needs none.
* **Deleting** a user (soft delete, `Admin::hookOnSoftDelete()` → `removeCredentials()`)
  removes sessions, devices, passkeys, recovery codes and `history`; the audit log stays.
  The own account cannot be deleted.

## Key rotation

```
php Core/cron/camindo.php key status
php Core/cron/camindo.php key rotate [--invalidate-passwords] [--no-backup]
php Core/cron/camindo.php key retire KID [--force]
```

`Crypt` keeps a **key ring**: the current key in `.camindo.key` (or `CAMINDO_KEY`, see
above), retired ones in
`.camindo.key.retired`. `encrypt()`/`hash()` only use the current one, `decrypt()` then
tries the retired ones, `hashAll()` returns the HMAC under all of them (for recovery
codes and devices).

`KeyRotation::rotate()` (at most once per 24 hours): backup, maintenance mode, new key,
then re-encrypt everything (`data_enc` including `hash`, `record_cache`, user contact
data and TOTP secrets, messages, scheduler passwords, frontend TOTP, audit files),
rewrite all signatures (deletes all user sessions), rebuild the group cache. Password
hashes stay valid and are renewed at the next login; `--invalidate-passwords` empties
them instead. An abort loses nothing – `decrypt()` knows both keys, and `rotate` can be
repeated. `retire` only removes an old key once nothing depends on it any more.

File permissions (`KeyRotation::keyFileMode()`): the new key and `.camindo.key.retired`
take over the permissions of the previous `.camindo.key` (not every system allows
`0600`); without a file (key from `CAMINDO_KEY`) `permissions.key_file` applies (default
`0600`). `key status` warns if the file differs from `permissions.key_file`.

**Values an add-on module encrypts itself are not covered by the rotation** – there is
no hook for that yet. Where possible, encrypt in fields of type `encrypted`.

## Audit log

`Dataset` logs in `save()`, `softDelete()`, `restore()` and `purge()` – inside the
operation's transaction; a rollback takes the entry with it. Snapshots are compared
(`Dataset::auditSnapshot()`); a save without changes writes nothing. In addition:
failed logins and exports of personal data; successful logins and logouts only with
`audit.log_logins = true` (`AuditLog::login()`, off by default – they reveal working
hours but no changes).

* **Split:** `audit_log` holds the metadata (time, actor, origin, IP, action, object,
  names of changed fields); `AuditStore` writes the complete entries to
  `var/audit/{type}/{id/1000}/{id}[.{page}].jsonl` – after the commit
  (`Database::afterCommit()`).
* **HMAC chain:** every line carries the HMAC over its content and that of the previous
  line. `camindo.php audit verify` (daily via cron, result in the operation notices)
  checks the chain and reconciles it with the database – a changed or removed line is
  detected, as is a metadata row that is missing, added, or whose columns (time, action,
  actor, origin, object reference, label, field names, IP truncated to two octets) do
  not match the signed entry in the file (`AuditLog::verifyObject()`). Whoever has file system **and** key can still rewrite everything.
* **Dump restored:** `camindo.php audit restore` (`AuditLog::restoreToDatabase()`)
  removes entries with an `audit_id` above the database maximum
  (`AuditStore::cutAfter()`): lines before stay unchanged, followed by a `restore`
  checkpoint; the old files go to `var/audit-restore-{timestamp}/`.
* **Broken chain:** `camindo.php audit reseal TYPE ID [--reason=…]`
  (`AuditStore::reseal()`) relinks from the first break – after a `reseal` checkpoint,
  the lines before stay unchanged. Only if every line has a valid HMAC (the entries are
  genuine, only the linking is wrong); otherwise refused, and only done with `--force`
  (the checkpoint records `unverified`). Copy to `var/audit-reseal-{timestamp}/`, entry
  in the `security` log.
* **Manual deletion:** `camindo.php audit delete TYPE ID` or `--area=ID --force`
  (`AuditLog::deleteObjects()`, `recordIdsOfArea()`) removes the whole history –
  metadata rows and files, without a checkpoint. For records also their jobs
  (`PrivacyJobs::forget()`): completed ones always, pending ones only if the record no
  longer exists. Entry in the `security` log (counts only).
* **Values:** passwords only as `***`; encrypted values as ciphertext, decrypted only for
  display (`AuditDiff`).
* **Subclasses** control it with `$audit = false`, `$auditIgnore` and `auditSecret()`
  (`Template`: `styled_code`, derived from `code`; full-text fields never appear in a
  record snapshot).
* **Retention** `audit.retention_days_*`; `AuditLog::purge()` shortens files and sets a
  checkpoint in the chain. Logins/logouts and failed logins have their own period
  (`audit.retention_days_logins`, default 90 days); they sit in the middle of an
  account's history and are removed via `AuditStore::rewriteObject()` – if the callback
  returns `null`, the entry is dropped (checkpoint `retention`).
* **Delete policies** (`Modules\Privacy\DeletePolicy`, `PrivacyJobs`): sources `record`
  (every way of deleting, including child records – tree children with `$withChilds`
  and dependent link content get the blocks of their own area without
  `soft_delete`/`purge`), `profile` (always for all records with the `record.user_id` of
  a deleted profile, also when it is deleted together with its area) and `area` (`PrivacyJobs::scheduleArea()` from
  `Area::hookOnSoftDelete()`). `AuditLog::overwriteObject()` (action `audit_set_fields`)
  replaces values and the label, checkpoint `privacy`. The actors' `user_id` always
  stays: an entry becomes anonymous once the profile is deleted permanently or emptied.
* **Anonymizing deleted users** `Modules\Privacy\AdminAnonymizer` (module Privacy, tab
  “Deleted users”, extra right `admin_anonymize`): name → `admin_{id}` (reserved,
  `Admin::hookOnSave()`), personal fields, settings, password and second factor cleared,
  re-signed. In the audit log only the account's own history (label, before/after of the
  personal fields, its failed logins) and older failed logins under object 0 with their
  name – two files at most (`AuditStore::rewriteObject()`, checkpoint `privacy`). Where
  they acted they are stored as an id only (see below); after the rename the display shows
  `admin_{id}`. Every step can be repeated and the row is renamed last – an interrupted
  run is simply run again; `admin_{id}` marks completion. Entry `privacy` on the account,
  a line in the `security` log.
* **Actors as ids only:** `actor.admin_id` or `actor.user_id`, no name – neither in
  `audit_log.actor_name` nor in the file. `AuditDiff::actorLabel()` inserts the name from
  `admin` when displaying; the search looks up names via `admin` (`AuditLog::where()`), as
  does the list of actors (`actors()`). A failed login with the name of an existing account
  is stored under its `admin_id` (`AuditLog::login()`); only an unknown name is written
  (object 0). Existing entries: `camindo.php audit strip-names`
  (`AuditLog::stripActorNames()`, checkpoint `strip-names`).
* **IP addresses** are stored truncated with zeros in the database and files (valid
  addresses, `ClientIp::anonymize()` stays idempotent); `ClientIp::display()` shows the
  truncated parts as `xxx` (personal data export, audit display, device list) – all trailing
  zero octets (two at most), regardless of the current setting. If `privacy.ip_v4_octets`
  is raised, the next `IpAnonymizer` run re-truncates the database and log files completely,
  and the files under `var/audit` (`AuditLog::anonymizeFileIps()`); lowering it restores
  nothing.

Display: tab "Changes" of the detail views – records and every other dataset info
page (area, field, template, status, instance, language, converter, task, component,
user group; user for superusers only) via `Backend::auditTabAdd()`, `auditTabActive()`,
`auditHistory()`, or `infoAuditTabs()` for pages without tabs of their own –, module
Audit (superusers), tab
"Activity" of the user detail view (superusers, `AdminBackend::audit()` →
`AuditBackend::adminActivity()`: recently edited records via
`AuditLog::recentObjects()`, below them all entries of the user). Through the API:
`audit.search` and `audit.history` ([16](16-api.md)) – without superuser rights only
records in readable areas, IP addresses for superusers only.

## Operation notices and update check

`SystemHealth` checks for superusers (start page): cron heartbeat, backups, writable
directories, audit log chain, backend domain, available updates. `UpdateCheck` runs
`composer outdated` and `composer audit` daily (result in
`var/cache/update-check.json`); security advisories for installed packages appear as a
warning.
