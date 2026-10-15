# 13 – Sicherheit I: Anmeldung, Schlüssel, Protokoll

Das Handbuch für Administratoren beschreibt die Sicherheitsfunktionen aus Sicht des
Betriebs. Dieses Kapitel beschreibt, wie sie gebaut sind und was neuer Code beachten
muss. Web-Themen (CSRF, HTML-Filter, Header, Script-Policy, Uploads) stehen in
[14](14-security-web.md).

## Bedrohungsmodell

| Angreifer | hat | hat nicht |
|---|---|---|
| **DB-Schreiber** | Schreibzugriff auf die Datenbank (geleaktes DB-Passwort, SQL-Injection in fremdem Code auf demselben Host, Datenbank-Werkzeug) | das Dateisystem, insbesondere `.camindo.key` |
| **Skript im Browser** | JavaScript, das im Backend oder Frontend läuft (gespeichertes oder reflektiertes XSS) | den Server |

Wer Dateisystem **und** Datenbank hat, ist Betreiber; dagegen hilft nichts in diesem
Kapitel. Das Geheimnis, auf dem fast alles beruht, ist `.camindo.key`.

## `Crypt` und der Schlüssel

| Methode | Zweck |
|---|---|
| `encrypt($text, $salt = '', $binary = false)` / `decrypt()` | AES-256-GCM mit zufälligem IV; `$salt` bindet den Geheimtext als AAD an einen Kontext – in eine andere Zeile kopiert, entschlüsselt er nicht |
| `hash($value, $binary = false)` | deterministischer HMAC-SHA256 mit dem Schlüssel – Suche in verschlüsselten Feldern, Signaturen, Token |
| `passwordHash()` / `passwordVerify()` / `passwordNeedsRehash()` | Passwörter, s. u. |

Nutzen Sie für neue Geheimnisse diese Methoden, nie eigene Konstruktionen, und binden
Sie Geheimtexte per AAD an ihre Zeile (`"zweck|{id}"`).

## Geheimnisse aus der Umgebung

Passwörter und der Schlüssel können statt in Dateien in Umgebungsvariablen stehen
(Docker, PHP-FPM `env[…]`, Secret-Verwaltung der Plattform). Eine `.env`-Datei wird
bewusst nicht gelesen, es gibt keine Bibliothek dafür.

| Variable | ersetzt | Vorrang |
|---|---|---|
| `CAMINDO_DB_PASS` | `db.pass` | Variable |
| `CAMINDO_SMTP_PASS` | `email.smtp.pass` | Variable |
| `CAMINDO_AI_API_KEY` | `ai.api_key` (Zusatzmodul Ai) | Variable |
| `CAMINDO_KEY` | Inhalt von `paths.key_file` | **Datei** |

`Config` setzt die Werte aus `Config::ENV_SECRETS` nach dem Zusammenführen mit
`.camindo.php` ein; `fromEnv()` nennt, welche aus der Umgebung kamen. Gelesen wird
ausschließlich über `Config::env($name)`: Der Wert wird je Prozess zwischengespeichert
und danach aus Prozessumgebung, `$_ENV` und `$_SERVER` entfernt – Kindprozesse
(`Execute`) erben ihn nicht, `phpinfo()` zeigt ihn nicht (außer Apache-`SetEnv` unter
„Apache Environment“). Eine Komponente mit eigenem Geheimnis liest ihre Variable
ebenfalls über `Config::env()` und lässt sie vor dem Wert aus `.camindo.php` gelten.

Beim Schlüssel ist der Vorrang umgekehrt: `Crypt` nimmt `paths.key_file`, wenn die
Datei existiert, sonst `CAMINDO_KEY` (`keySource()`: `file`|`env`). Existiert die Datei
(oder `.camindo.key.retired`), ist aber nicht lesbar – meist Eigentümer root nach einer
Installation als root –, wirft `Crypt`, statt auf die Variable auszuweichen: Die könnte
einen beliebigen anderen Schlüssel enthalten, unter dem dann neu verschlüsselt würde
(Eigentümer der Dateien: [09](09-installation-and-update.md)). So gilt nach
`key rotate` – das den neuen Schlüssel immer in die Datei schreibt – sofort der neue,
auch wenn die Variable noch den alten enthält. `envKeyState()` vergleicht die Variable
mit dem Schlüsselbund (`current`, `retired`, `other`); `key status`, `status` und
`SystemHealth` (Prüfung `keyenv`) melden eine veraltete Variable. Enthält die Variable
allein (ohne Datei) einen abgelösten Schlüssel, wirft `Crypt` beim Start: Mit ihm
weiterzuarbeiten hieße, Neues unter dem alten Schlüssel zu verschlüsseln und alle
Signaturen für manipuliert zu halten.

`Backup::mysqldump()` schreibt bei `db.pass` aus der Umgebung (oder ohne `.sqlpwd`) je
Lauf eine Optionsdatei (`0600`) unter `paths.tmp_dir` und löscht sie danach.

## Passwörter

```
passwordHash($pw)            {kid}:password_hash(base64(HMAC-SHA256($pw, key)), Argon2id | bcrypt)
passwordVerify($pw, $stored) prüft in konstanter Zeit; ohne Treffer gegen einen Ersatz-Hash
passwordNeedsRehash($stored) altes Verfahren, andere Parameter oder fremder Schlüssel
```

* **Verfahren:** `security.password_hash.algo` – `auto` = Argon2id, wenn PHP es
  mitbringt, sonst bcrypt. Argon2id mit 19 MiB und `time_cost` 2 (OWASP-Minimum), bcrypt
  `cost` 11.
* **Pfeffer:** Der HMAC mit `.camindo.key` davor macht einen Datenbank-Dump ohne
  Schlüssel wertlos und gibt bcrypt immer 44 Zeichen.
* **Kennung des Schlüssels** (`{kid}:`) – nötig für die Schlüsselrotation.
* **Rehash bei der Anmeldung:** Nach erfolgreicher Prüfung schreiben
  `Admin::loginByName()` und `RecordUser` einen Hash im aktuellen Format.
* **Timing:** Ohne Treffer zum Namen wird gegen einen Ersatz-Hash gerechnet
  (`var/cache/password_dummy.txt`) – ein erfundener Name antwortet nicht schneller.

Passwort ohne Backend setzen: `php Core/cron/camindo.php admin password NAME`
(`--pass-file`, `CAMINDO_ADMIN_PASS` oder verdeckte Eingabe).

### Passwort-Richtlinie

`PasswordPolicy` (`security.password_policy`): `min_length` 12, `min_special` 1,
`min_upper` 1 (`upper_not_first`: ein Großbuchstabe an erster Stelle zählt nicht),
`min_lower` 1, `min_digit` 0 – jede Regel einzeln abschaltbar (0). Geprüft in
`Dataset::validateValue()` für jede Eingabe in ein Feld mit `display_type password`,
wenn es die Extension `Password` trägt (`admin.password` immer; Passwort-Felder von
Bereichen, wenn in der Felddefinition gewählt, mit Überschreibungen in
`extension_options`). Die Extension zeigt Stärke und Regeln im Browser; maßgeblich ist
allein die Serverprüfung. Wer eine Regel ergänzt, ändert `PasswordPolicy::count()` und
das Script der Extension.

## Signaturen

Gegen den DB-Schreiber trägt jede sicherheitsrelevante Zeile in `sig` einen HMAC über
ihre entscheidenden Spalten (`Camindo\Core\Integrity`). Eine Zeile mit falscher oder
fehlender Signatur wird behandelt, als gäbe es sie nicht, und im Log `security`
vermerkt.

| Tabelle | Signatur über | geprüft in |
|---|---|---|
| `admin` | `admin_id`, `name`, `password`, `group_id`, `status_id`, `second_factor` (wenn gesetzt) | Anmeldung, `Session::load()` (jeder Request) |
| `admin_session` | `session_id`, `admin_id` | `Session::load()` |
| `group` | `group_id`, `details` | Bau der Gruppendateien |
| `rel_group_instance`, `rel_group_area`, `rel_group_language`, `rel_group_status` | alle Spalten | Bau der Gruppendateien |
| `admin_passkey` | `admin_id`, Hash der Credential-Id, Hash des öffentlichen Schlüssels, `alg` (nicht der Signaturzähler) | `Passkeys` bei jedem Lesen |

Damit fallen auf: eine geänderte `group_id`, getauschte Namen oder Passwörter, ein
zurückgesetzter Status, eine eingeschleuste Session, selbst vergebene Rechte. Die
Rechte sind **je Zeile** signiert, weil Zeilen per `ON DELETE CASCADE` verschwinden
können – Zeilen löschen nimmt Rechte, es gibt keine. `Session::load()` liest `group_id`
und `status_id` je Request frisch aus der signierten `admin`-Zeile.

**Neue Schreibpfade** auf diese Tabellen müssen die Signatur mitschreiben
(`Integrity::*Signature()`), sonst gelten ihre Zeilen als manipuliert. Direktes SQL auf
eine signierte Spalte macht die Zeile ungültig – absichtlich. Alles neu signieren (nach
einem Schlüsselwechsel, für Entwicklungsdaten):
`php Core/cron/camindo.php admin sign` – löscht dabei alle Sessions.

## Sessions und Cookies

* `Session::create()` erzeugt **immer** eine neue Session-Id (gegen Session-Fixation);
  aus einer Session desselben Benutzers im Cookie werden nur aufgeklappte Ordner und
  offene Arbeitskopien übernommen.
* Cookie `sessid`: HttpOnly immer, `Secure` nach `admin.cookie_secure` (Vorgabe `true`),
  `SameSite` nach `admin.cookie_samesite` (Vorgabe `Lax`), kein Ablaufdatum;
  serverseitig verfällt die Session nach `admin.max_inactive_sec`. Jeder Request setzt
  `access_date` neu – außer Abfragen im Hintergrund (Polling von Nachrichten-Abzeichen und
  Hintergrundprozessen: `cLoadContent({background: true})`, Header `X-Camindo-Background: 1`),
  sonst hielte ein offenes Fenster die Session endlos am Leben.
* HttpOnly verhindert das **Auslesen** der Session-Id per JavaScript, nicht Requests im
  Namen der Session – dagegen helfen HTML-Filter und Script-Policy ([14](14-security-web.md)).
* Arbeitskopien hängen am Benutzer; `admin_session_record` hält fest, welche zu welcher
  Session gehört. Abmelden verwirft nur die Arbeitskopien dieser Session; „auf allen
  Geräten abmelden" alle Arbeitskopien und Sessions des Benutzers; „Alle anderen
  Sitzungen beenden" (Geräte und Sitzungen) die Arbeitskopien der übrigen Sessions
  (`Record::cancelOtherSessions()`) und diese Sessions.
* Jede Session gehört zu einer ***Oberfläche*** (`admin_session.interface`: `backend`,
  `cli`, `php`), die in ihrer Signatur steckt. Die Backend-Einstiege laden nur
  Backend-Sessions (`Session::load($id)`), die API nur ihre eigenen
  (`Session::load($id, ['cli', 'php'])`). Sessions der Kommandozeile und der PHP-API
  verfallen nach `api.max_inactive_sec` Untätigkeit und spätestens zu `expires_at`
  (Höchstlaufzeit, begrenzt durch `api.max_lifetime`). Sie haben kein Cookie und kein
  Anmeldegerät. Die Benutzergruppe muss die Oberfläche benutzen dürfen – geprüft bei der
  Anmeldung und bei jedem Laden ([11](11-permissions.md#oberflächen)).
* Passwort erneuern und TOTP einrichten gehen nur im Anmeldedialog des Backends; eine
  Anmeldung über die PHP-API wird bis dahin abgewiesen (`admin.error_login_backend_first`).

## Anmeldeschutz

| Schutz | Wie |
|---|---|
| Fehlversuche je Name | `LoginGuard` (Tabelle `admin_login_attempt`), `admin.bruteforce_max` in `admin.bruteforce_delay`; gezählt werden auch falsche Codes der zweiten Stufe; der Zähler wird erst nach **vollständiger** Anmeldung geleert |
| Fehlversuche je Herkunft | `RateLimiter` Bucket `admin_login_ip` (`admin.throttle.login_ip`, Vorgabe 20 je 10 min) – gegen Password-Spraying über viele Namen |
| Sperre durch Fremde | Ein Browser mit bekanntem Geräte-Cookie des Kontos zählt unter einem eigenen Schlüssel – wer den Namen kennt, kann das Konto nicht aussperren |

### Anmeldegeräte

`LoginDevices::register()` läuft nach jeder vollständigen Anmeldung (Backend und
Frontend): Cookie `c9dev` (Zufallswert, HttpOnly, `security.device_days`), in der
Tabelle `login_device` nur als HMAC. Ein neues Gerät eines Kontos, das schon Geräte hat,
löst das Ereignis `new_device` aus. Ein bekanntes Gerät **überspringt keinen Faktor** – es
ist ein Wiedererkennungsmerkmal, kein Anmeldegeheimnis.

Benachrichtigungen gehen an `LoginNotifierInterface::notify($event, $kind, $account,
$context)`; die Klasse kommt aus `security.login_notifier` (Vorgabe `LoginNotifier`:
E-Mail an Benutzer; für Frontend-Konten nichts – das Template sieht die
Ereignisse in `$set`). Ereignisse: `new_device`, `password_changed`, `totp_enabled`,
`totp_disabled`, `code_enabled`, `code_disabled`, `recovery_code_used`. Ein eigener
Notifier (SMS, Push) kann den Standard im Konstruktor verlangen und als Rückfall rufen.

Benutzer sehen ihre Geräte und Sessions unter `Admin/devices.php` und können sie
einzeln entfernen; `admin_session.device_id` sorgt dafür, dass mit einem Gerät auch
seine Sessions enden.

## Zweiter Faktor im Backend

`Camindo\Core\Modules\Admin\SecondFactorInterface`:

```php
interface SecondFactorInterface {
    public function name(): string;                        // 'email' | 'totp' | …
    public function start(array $admin): bool;             // nach dem Passwort
    public function verify(array $admin, string $code): bool;
}
```

| Implementierung | Code |
|---|---|
| `EmailFactor` | 5 Ziffern per E-Mail, ein Versuch je Code |
| `TotpFactor` | 6 Ziffern aus einer Authenticator-App (RFC 6238, ohne Bibliothek: `Totp`) oder ein Wiederherstellungscode |
| `WebAuthnFactor` | Passkey bzw. Sicherheitsschlüssel (WebAuthn); daneben App-Code (wenn TOTP aktiv) oder Wiederherstellungscode |

`Admin::secondFactor()` wählt je Benutzer; `security.admin_2fa` = `email` | `totp`
(Vorgabe, freiwillig) | `totp_required` (Superuser müssen). Die Anmeldung ist
zweistufig: nach dem Passwort eine einmalige `uid` im Formular, die genau ein
Prüfversuch einlösen darf.

* TOTP-Secret verschlüsselt mit AAD `totp|{admin_id}`; Replay-Sperre über
  `UPDATE … WHERE totp_step < ?`.
* Einrichten ohne Zwischenstand: ein verschlüsseltes Setup-Token reist im Formular mit
  (15 Minuten gültig); erst ein gültiger Code schreibt das Secret. Der QR-Code entsteht
  im Browser – kein fremder Dienst sieht das Secret.
* Zehn Wiederherstellungscodes, gespeichert als HMAC in `admin_recovery_code`,
  Einlösen = `DELETE`.
* `admin.second_factor` ist Teil der Signatur: ein Herabstufen per SQL macht die Zeile
  ungültig.

### WebAuthn

Passkeys als zweiter Faktor, ohne Bibliothek: `Camindo\Core\WebAuthn` baut die Optionen
für `navigator.credentials.create()/get()` und prüft die Antworten (CBOR, COSE → PEM;
ES256, EdDSA über libsodium, RS256 ab 2048 Bit). Attestation wird als `none` angefordert
und nicht ausgewertet – es zählt der Besitz des Schlüssels, nicht die Herkunft des Geräts.

* **Voraussetzung:** feste Backend-Domain ([14](14-security-web.md#feste-backend-domain)).
  RP ID = `admin.backend_domain` ohne Port, Origin `https://{domain}`. Ändert sich die
  Domain, sind alle Passkeys wertlos. Abschaltbar mit `security.webauthn = false`,
  unabhängig von `security.admin_2fa`; ein Passkey erfüllt auch `totp_required`.
* **Welcher Faktor:** `WebAuthnFactor::applies()` vor TOTP und E-Mail – bei
  `second_factor = 'webauthn'` (nur Passkeys) oder wenn Passkeys vorhanden sind (auch neben
  TOTP; das Formular nimmt dann beides). Die Kennung `webauthn` steht in der Signatur der
  Zeile in `admin`: gelöschte Passkey-Zeilen stufen nicht herunter, es bleiben die
  Wiederherstellungscodes.
* **Anmeldung:** Die Challenge ist `Crypt::hash('webauthn-login|' . uid)` – die `uid` ist
  zufällig und genau einen Versuch wert, also auch die Challenge; eine Spalte braucht es
  nicht. Das Formular schickt die Antwort als JSON im Feld `passkey`
  (`cApp.ui.passkeyLogin()`), `loginByCode()` reicht die `uid` an `verify()` durch.
  Geprüft: Typ, Challenge, Origin, `crossOrigin`, `rpIdHash`, Flag UP, Signatur über
  `authData ‖ SHA-256(clientDataJSON)`; der Signaturzähler muss steigen (außer beide 0 –
  synchronisierte Passkeys führen keinen). UV wird nicht verlangt, der Passkey ist der
  zweite Faktor nach dem Passwort. Fehlversuche zählen im `LoginGuard`.
* **Einrichten** (`AdminBackend::totp()`, Schritte `passkey_*`): erst Bezeichnung und
  bisheriges Passwort, dann die Geste am Gerät – so entsteht bei falschem Passwort kein
  Passkey, den der Server nie kennt. Challenge und Bezeichnung reisen verschlüsselt als
  Token im Formular (AAD `webauthn-reg|{admin_id}`, 10 Minuten). Der erste Passkey setzt
  `second_factor = 'webauthn'` (außer bei TOTP) und erzeugt Wiederherstellungscodes, wenn
  keine da sind. `user.id` im Passkey ist ein HMAC der `admin_id`, nicht sprechend.
* **Speicherung:** `admin_passkey` (Credential-Id, Hash der Id als eindeutiger Schlüssel,
  PEM, `alg`, Zähler, Transportwege, Bezeichnung), je Zeile signiert (`Integrity::
  passkeySignature()`, auch in `signAll()`), höchstens 20 je Konto (`Passkeys`).
* **Entfernen** mit dem Passwort; ohne Passkeys und TOTP endet der eigene Faktor und die
  Codes verfallen. „Zweiten Faktor zurücksetzen" durch einen Superuser entfernt App und
  Passkeys. Ereignisse `passkey_added`, `passkey_removed` gehen an den Notifier und ins
  Audit-Log.
* **Selbsttest:** `php Core/cron/check_webauthn.php` – ein Software-Authenticator
  registriert und meldet sich mit allen drei Verfahren an, dazu die Fehlerfälle.

Offen: Anmeldung ohne Passwort (`residentKey: preferred` ist schon angefordert) und
Passkeys für Frontend-Profile.

## Passwort-Pflichten für Benutzer

* Ein Passwort, das ein **anderer** Benutzer gesetzt hat (Anlegen, Zurücksetzen), muss bei
  der nächsten Anmeldung geändert werden (`admin.password_change`).
* `admin.password_retention_days` > 0 verlangt eine Änderung nach Ablauf.
* Das eigene Passwort ändert nur, wer das bisherige kennt; jede Änderung beendet die
  anderen Sessions des Kontos.
* Zusatzrechte `admin_create` (anlegen mit Startpasswort) und `admin_change` (ändern,
  sperren) gelten nur für die eigene Benutzergruppe; das Passwort eines bestehenden anderen
  Benutzers setzt nur ein Superuser – und nur mit seinem **eigenen** Passwort im Feld
  `password_current` (`AdminBackend::_editPasswordCurrentOk()`, Fehlversuche zählen im
  `LoginGuard`). Eine offen stehende Sitzung reicht so weder für das eigene noch für ein
  fremdes Konto. Ein Konto ohne Passwort (nur per CLI angelegt) braucht keins.
* **Löschen** eines Benutzers (Soft-Delete, `Admin::hookOnSoftDelete()` →
  `removeCredentials()`): Sessions, Geräte, Passkeys, Wiederherstellungscodes und
  `history` werden entfernt, das Audit-Log bleibt. Das eigene Konto lässt sich nicht
  löschen.

## Schlüsselrotation

```
php Core/cron/camindo.php key status
php Core/cron/camindo.php key rotate [--invalidate-passwords] [--no-backup]
php Core/cron/camindo.php key retire KID [--force]
```

`Crypt` führt einen **Schlüsselbund**: den aktuellen Schlüssel in `.camindo.key` (oder
`CAMINDO_KEY`, s. o.),
abgelöste in `.camindo.key.retired`. `encrypt()`/`hash()` nutzen nur den aktuellen,
`decrypt()` versucht danach die abgelösten, `hashAll()` liefert den HMAC unter allen
(für Wiederherstellungscodes und Geräte).

`KeyRotation::rotate()` (höchstens einmal je 24 Stunden): Sicherung, Wartungsmodus,
neuer Schlüssel, dann alles neu verschlüsseln (`data_enc` samt `hash`, `record_cache`,
Kontaktdaten der Benutzer und TOTP-Secrets, Nachrichten, Zeitplaner-Passwörter,
Frontend-TOTP, Audit-Dateien), alle Signaturen neu schreiben (löscht alle
Benutzer-Sessions), Gruppen-Cache neu. Passwort-Hashes bleiben gültig und werden bei der
nächsten Anmeldung erneuert; `--invalidate-passwords` leert sie stattdessen. Ein
Abbruch verliert nichts – `decrypt()` kennt beide Schlüssel, `rotate` lässt sich
wiederholen. `retire` entfernt einen alten Schlüssel erst, wenn nichts mehr von ihm
abhängt.

Dateirechte (`KeyRotation::keyFileMode()`): Neuer Schlüssel und `.camindo.key.retired`
übernehmen die Rechte der bisherigen `.camindo.key` (nicht jedes System erlaubt `0600`);
ohne Datei (Schlüssel aus `CAMINDO_KEY`) gilt `permissions.key_file` (Vorgabe `0600`).
`key status` warnt bei Abweichung von `permissions.key_file`. Als root übernehmen beide
Dateien auch den Eigentümer der bisherigen (`SecretFile::copyOwner()`), ohne Datei den des
Projektverzeichnisses, falls nicht root.

**Werte, die ein Zusatzmodul selbst verschlüsselt, erfasst die Rotation nicht** – es
gibt dafür noch keinen Einhängepunkt. Verschlüsseln Sie nach Möglichkeit in Feldern vom
Typ `encrypted`.

## Audit-Log

`Dataset` protokolliert in `save()`, `softDelete()`, `restore()` und `purge()` – in der
Transaktion des Vorgangs, ein Rollback nimmt den Eintrag mit. Verglichen werden
Schnappschüsse (`Dataset::auditSnapshot()`); ein Speichern ohne Änderung schreibt
nichts. Dazu Fehlanmeldungen und Exporte personenbezogener Daten; erfolgreiche An- und
Abmeldungen nur mit `audit.log_logins = true` (`AuditLog::login()`, Vorgabe aus – sie
verraten Arbeitszeiten, aber keine Änderungen).

* **Aufteilung:** `audit_log` hält die Metadaten (Zeit, Handelnder, Herkunft, IP,
  Aktion, Objekt, Namen der geänderten Felder); die vollständigen Einträge schreibt
  `AuditStore` nach `var/audit/{typ}/{id/1000}/{id}[.{seite}].jsonl` – nach dem Commit
  (`Database::afterCommit()`).
* **HMAC-Kette:** jede Zeile trägt den HMAC über ihren Inhalt und den der Vorzeile.
  `camindo.php audit verify` (täglich als Zeitplaner-Aufgabe `auditverify`, Ergebnis in den Betriebshinweisen) prüft
  die Kette und gleicht mit der Datenbank ab – eine geänderte oder entfernte Zeile fällt
  auf, ebenso eine Metazeile, die fehlt, hinzukommt oder deren Spalten (Zeit, Aktion,
  Handelnder, Herkunft, Objekt-Bezug, Bezeichnung, Feldnamen, IP auf zwei Oktette
  gekürzt) nicht zum signierten Eintrag der Datei passen (`AuditLog::verifyObject()`). Wer Dateisystem **und** Schlüssel hat, kann trotzdem alles neu schreiben.
* **Dump zurückgespielt:** `camindo.php audit restore` (`AuditLog::restoreToDatabase()`)
  entfernt Einträge mit `audit_id` über dem Maximum der Datenbank
  (`AuditStore::cutAfter()`): Zeilen davor bleiben unverändert, danach folgt ein
  Kontrollpunkt `restore`; die alten Dateien landen in `var/audit-restore-{Zeitpunkt}/`.
* **Kette gebrochen:** `camindo.php audit reseal TYP ID [--reason=…]`
  (`AuditStore::reseal()`) verkettet ab der ersten Bruchstelle neu – hinter einem
  Kontrollpunkt `reseal`, die Zeilen davor bleiben unverändert. Nur wenn jede Zeile
  einen gültigen HMAC hat (die Einträge sind echt, nur die Verkettung stimmt nicht);
  sonst verweigert und erst mit `--force` (der Kontrollpunkt vermerkt `unverified`).
  Kopie nach `var/audit-reseal-{Zeitpunkt}/`, Eintrag im Protokoll `security`.
* **Von Hand löschen:** `camindo.php audit delete TYP ID` bzw. `--area=ID --force`
  (`AuditLog::deleteObjects()`, `recordIdsOfArea()`) entfernt den ganzen Verlauf –
  Metazeilen und Dateien, ohne Kontrollpunkt. Bei Datensätzen dazu die Aufträge
  (`PrivacyJobs::forget()`): erledigte immer, offene nur, wenn es den Datensatz nicht
  mehr gibt. Eintrag im Protokoll `security` (nur Anzahlen).
* **Werte:** Passwörter nur als `***`; verschlüsselte Werte als Geheimtext, entschlüsselt
  erst in der Anzeige (`AuditDiff`).
* **Kindklassen** steuern mit `$audit = false`, `$auditIgnore` und `auditSecret()`
  (`Template`: `styled_code`, abgeleitet aus `code`; Volltextfelder stehen nie im
  Schnappschuss eines Datensatzes).
* **Aufbewahrung** `audit.retention_days_*`; `AuditLog::purge()` kürzt Dateien und
  setzt einen Kontrollpunkt in die Kette. An-/Abmeldungen und Fehlversuche haben eine
  eigene Frist (`audit.retention_days_logins`, Vorgabe 90 Tage); sie liegen mitten im
  Verlauf eines Kontos und werden über `AuditStore::rewriteObject()` entfernt – gibt
  die Funktion `null` zurück, fällt der Eintrag weg (Kontrollpunkt `retention`).
* **Lösch-Policies** (`Modules\Privacy\DeletePolicy`, `PrivacyJobs`): Auslöser
  `record` (jeder Löschweg, auch Unterdatensätze – Kinder bei `$withChilds` und fest
  verknüpfte Inhalte bekommen die Blöcke ihres Bereichs ohne `soft_delete`/`purge`),
  `profile` (immer für alle Datensätze mit `record.user_id` eines gelöschten Profils, auch wenn es mit seinem Bereich gelöscht wird)
  und `area` (`PrivacyJobs::scheduleArea()` aus `Area::hookOnSoftDelete()`).
  `AuditLog::overwriteObject()` (Aktion `audit_set_fields`) ersetzt Werte und
  Bezeichnung, Kontrollpunkt `privacy`. Die `user_id` der Handelnden bleibt immer
  stehen: Anonym wird ein Eintrag, indem das Profil endgültig gelöscht oder geleert
  wird.
* **Gelöschte Benutzer anonymisieren** `Modules\Privacy\AdminAnonymizer` (Modul Privacy,
  Reiter „Gelöschte Benutzer“, Zusatzrecht `admin_anonymize`): Name → `admin_{id}`
  (reserviert, `Admin::hookOnSave()`), persönliche Felder, Einstellungen, Passwort und
  zweiter Faktor geleert, neu signiert. Im Audit-Log nur der Verlauf seines Kontos
  (Bezeichnung, Vorher/Nachher der persönlichen Felder, seine Fehlversuche) und ältere
  Fehlversuche unter Objekt 0 mit seinem Namen – höchstens zwei Dateien
  (`AuditStore::rewriteObject()`, Kontrollpunkt `privacy`). Wo er gehandelt hat, steht er
  nur als Id (siehe unten), die Anzeige zeigt nach dem Umbenennen `admin_{id}`. Jeder
  Schritt ist wiederholbar, die Zeile wird zuletzt umbenannt – ein abgebrochener Lauf
  wird einfach wiederholt; `admin_{id}` kennzeichnet den Abschluss. Eintrag `privacy` am
  Konto, Zeile im Log `security`.
* **Handelnde nur als Id:** `actor.admin_id` bzw. `actor.user_id`, kein Name – weder in
  `audit_log.actor_name` noch in der Datei. `AuditDiff::actorLabel()` setzt den Namen
  beim Anzeigen aus `admin` ein; die Recherche sucht Namen über `admin`
  (`AuditLog::where()`), die Auswahlliste der Handelnden ebenso (`actors()`). Ein
  Fehlversuch mit dem Namen eines bestehenden Kontos steht unter dessen `admin_id`
  (`AuditLog::login()`), nur ein unbekannter Name wird mitgeschrieben (Objekt 0).
  Altbestand: `camindo.php audit strip-names` (`AuditLog::stripActorNames()`,
  Kontrollpunkt `strip-names`).
* **IP-Adressen** stehen gekürzt mit Nullen in Datenbank und Dateien (gültige Adressen,
  `ClientIp::anonymize()` bleibt idempotent); `ClientIp::display()` zeigt die gekürzten
  Stellen als `xxx` (Datenauskunft, Audit-Anzeige, Geräteliste) – alle Null-Oktette am
  Ende (höchstens zwei), unabhängig von der heutigen Einstellung. Wird
  `privacy.ip_v4_octets` erhöht, kürzt der nächste Lauf von `IpAnonymizer` Datenbank und
  Logdateien vollständig nach und die Dateien unter `var/audit`
  (`AuditLog::anonymizeFileIps()`); gesenkt bringt nichts zurück.

Anzeige: Reiter „Änderungen" der Detailansichten – Datensatz und jede andere
Dataset-Info-Seite (Bereich, Feld, Template, Status, Instanz, Sprache, Konverter,
Aufgabe, Komponente, Benutzergruppe; Benutzer nur Superuser) über `Backend::auditTabAdd()`,
`auditTabActive()`, `auditHistory()` bzw. `infoAuditTabs()` für Seiten ohne eigene
Reiter –, Modul Audit (Superuser),
Reiter „Aktivitäten" der Detailansicht eines Benutzers (Superuser, `AdminBackend::audit()` →
`AuditBackend::adminActivity()`: zuletzt bearbeitete Datensätze über
`AuditLog::recentObjects()`, darunter alle Einträge des Benutzers). Über die API:
`audit.search` und `audit.history` ([16](16-api.md)) – ohne Superuser nur Datensätze
lesbarer Bereiche, IP-Adressen nur für Superuser.

## Betriebshinweise und Update-Prüfung

`SystemHealth` prüft für Superuser (Startseite): Cron-Herzschlag, Sicherungen,
schreibbare Verzeichnisse, Kette des Audit-Logs, Backend-Domain, verfügbare Updates.
`UpdateCheck` ruft täglich `composer outdated` und `composer audit` auf (Ergebnis in
`var/cache/update-check.json`); Sicherheitsmeldungen zu installierten Paketen erscheinen
als Warnung.
