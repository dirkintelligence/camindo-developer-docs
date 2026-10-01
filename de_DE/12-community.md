# 12 – Frontend-Community

„Frontend-User" ist in camindo der registrierte Website-Besucher mit Konto – nicht der
Backend-Benutzer, der heißt schlicht „Benutzer". Ein Frontend-User hat **keine eigene
Tabelle**: sein Abbild im CMS, der Profil-Datensatz, ist ein
Datensatz in einem Bereich vom Typ `profile`, seine Beiträge sind Datensätze in Bereichen
vom Typ `ugc`. Gelesen wird über `Cms::select()`, geschrieben über `Record` –
Arbeitskopie, Validierung, Uploads und Thumbnails inklusive.

| Klasse | Aufgabe |
|---|---|
| `Modules\Record\RecordUser` | Laufzeit der `CMS:USER_*`-Befehle |
| `UserSession` | Frontend-Sessions (Cookie `usess`), CSRF-Token (Cookie `ucsrf`) |
| `Modules\Record\UserSecondFactor` | zweiter Faktor für Profile |
| `MediaDelivery` | Auslieferung geschützter Dateien (`public/file.php`) |
| Modul `User` | Backend-Sicht: Gegenlesen neuer Beiträge (`record.needs_review`) |

Wie Template-Programmierer die Befehle einsetzen, beschreibt das Handbuch für
Template-Programmierer; hier steht, was dahinter passiert.

## Bereichstypen `profile` und `ugc`

`record`, `profile` und `ugc` sind gleichermaßen Datensatzbereiche
(`Area::CONTENT_TYPES`) – für Maske, Listen, Suche und `select()` dasselbe.

| Typ | Zusatzangaben |
|---|---|
| `profile` | `area.login_name_field_id` (Feld vom Typ `chars` oder `encrypted`), `area.login_password_field_id` (Feld vom Typ `password`), `details.second_factor` |
| `ugc` | `details.profile_area_ids` – Profil-Bereiche, deren Frontend-User hier schreiben dürfen |

| Element | Zweck |
|---|---|
| `record.user_id` | Besitzer im Frontend = Profil-Datensatz; Grundlage der Besitzprüfung |
| `record.owner_id` | Besitzer im Backend (Benutzer), siehe [11](11-permissions.md#owner) |
| `record.status_id = 2` | „unbestätigt" – nach der Registrierung, bis zur Bestätigung |
| `record.needs_review` | 1 bei allem, was ein Frontend-User anlegt oder ändert |
| `user_session`, `user_basket`, `user_token`, `user_login_attempt` | Sessions, Warenkorb, Einmal-Tokens, Fehlversuche |
| `user_second_factor`, `user_recovery_code` | zweiter Faktor |

Passwörter liegen als Hash in `data_pwd` und **nie** im `record_cache`; `fetch()`
liefert Passwort-Felder immer leer.

## Sessions

| | |
|---|---|
| Cookie | `usess` (Name über `cookiename`), HttpOnly, `user.cookie_secure`, `user.cookie_samesite` (`Lax`), Browser-Session |
| Verfall | `user.max_inactive_sec` (86400) ohne Zugriff; verfallene Sessions räumt `create()` ab |
| anonym | `record_id` NULL – `USER_BASKET` legt solche Sessions für einen Warenkorb ohne Anmeldung an |
| Anmeldung | `USER_LOGIN` vergibt **immer eine neue Session-Id** (gegen Session-Fixation); Warenkorb und Details einer anonymen oder eigenen Session ziehen um, die alte Zeile wird gelöscht |
| Ende | `USER_LOGOUT` löscht Zeile und Cookie; `UserSession::closeAllOf($recordId, $exceptId)` beendet alle Sessions eines Profils (Löschen des Profils, Passwortwechsel) |

Cookies müssen vor der ersten Ausgabe gesetzt werden – das Frontend puffert bewusst
nicht. Die `USER_*`-Befehle gehören deshalb an den Anfang eines Templates.

## Schreibschutz: Token, Drosselung, Formularschutz

**CSRF-Token.** Alle schreibenden Befehle verlangen ein Token aus `CMS:USER_TOKEN` im
Request (POST-Feld `_token` oder Header `X-Camindo-Token`, nie aus dem Query-String):
`USER_LOGIN`, `USER_REGISTER`, `USER_PW`, `USER_INPUT` (sobald `submit` gefüllt ist),
`USER_DELETE`, `USER_LOGOUT`, `USER_BASKET` außer `load`. Das Token ist ein HMAC über das
Cookie `ucsrf` (signiertes Double-Submit), nicht an die Session gebunden – Anmeldung,
Registrierung und Reset laufen ohne Session. Der erste `CMS:USER_TOKEN` je Request muss
vor der Ausgabe stehen, weil er das Cookie setzt. Abschaltbar je Befehl (`csrf=false`)
oder global (`user.csrf`). Schreibende Formulare senden per POST;
`$_POST['submit']` ist die verbindliche Schreibweise für `submit`.

**Drosselung** (`RateLimiter`, Tabelle `rate_limit`, Schlüssel nur als HMAC gespeichert):
zählt **jede** Anfrage in einem gleitenden Fenster, Grenzen in
`user.throttle.{bucket}` = `[höchstens, Sekunden]`.

| Bucket | Vorgabe | wofür |
|---|---|---|
| `pw_value`, `pw_ip` | 3 / h, 10 / h | „Passwort vergessen" je eingegebenem Wert bzw. Herkunft |
| `register_ip`, `register_value` | 5 / h, 3 / Tag | Registrierung je Herkunft bzw. E-Mail |
| `mail_instance` | 200 / h | Mails aus `USER_PW` + `USER_REGISTER` je Instanz |
| `login_ip` | 30 / 10 min | Fehlanmeldungen je Herkunft über alle Namen |
| `login_code` | 3 / 10 min | „Neuen Code senden" je Profil |
| `email_ip` | 5 / h | `CMS:EMAIL throttle=true` |
| `zip_ip` | 30 / h | neu erzeugte Archive von `CMS:ZIP` je Herkunft (Treffer im Zwischenspeicher zählen nicht) |

`pw_value` zählt den **eingegebenen** Wert vor der Suche – sonst würde nur eine
existierende Adresse gesperrt, und daran ließe sich ablesen, welche Adressen ein Konto
haben. Die Herkunft bestimmt `ClientIp` (`X-Forwarded-For` nur von Adressen aus
`security.trusted_proxies`, IPv6 auf das /64 gekürzt).

**Brute-Force-Schutz der Anmeldung:** `LoginGuard::forUsers()` – Tabelle
`user_login_attempt`, `user.bruteforce_max` (5) Fehlversuche je Bereich und Name in
`user.bruteforce_delay` (600 s).

**Formularschutz** (`FormGuard`): `(CMS:USER_TOKEN formguard=true)` gibt einen signierten
Zeitstempel `_ft` und ein unsichtbares Köderfeld aus; `formguard=true` an
`USER_REGISTER`, `USER_PW` und `CMS:EMAIL` weist ab, wenn der Köder gefüllt ist oder das
Formular zu schnell (`user.formguard_min_seconds`) bzw. zu spät abgeschickt wurde.
Erweiterungspunkt für einen fremden Dienst: `security.form_challenge` = Klasse mit
`FormChallengeInterface::verify($_POST)`.

Unbestätigte Profile löscht der Cron nach `user.unconfirmed_days` (7) endgültig.

## Die Befehle

Ergebnisse landen in `$set`; nach `USER_LOGIN`, `USER_REGISTER`,
`USER_REGISTER_CONFIRM` und `USER_PW` ist das Profil außerdem der laufende Datensatz
`$cms` des Erfolgstemplates. Fehler gehen an `Cms::error()` und erscheinen über
`CMS:MESSAGE`; `message[index]` ersetzt einen Standardtext (Snippets `user.error_*`).
Die Standardtexte folgen `language.backend_code` der Seite – für Sprachen ohne eigene
Backend-Sprache setzt das Template `message[…]` selbst.

Formulardaten heißen `data[feldname]` (mehrsprachig `data[feld][language_id]`),
Uploads unter demselben Namen in `$_FILES` – dieselbe Form wie die Backend-Maske, sodass
`Record::save()` sie ohne Umweg versteht.

| Befehl | Wirkung |
|---|---|
| `USER_AUTH` / `USER_PROTECT` | Session aufnehmen; `PROTECT` bindet ohne Anmeldung `errortemplate` ein und endet |
| `USER_LOGIN` | Anmeldung mit Name + Passwort oder mit dem Reset-Token; verlangt ggf. den zweiten Faktor |
| `USER_LOGIN_CODE` | zweiter Schritt der Anmeldung (App-, E-Mail- oder Wiederherstellungscode), `resend` |
| `USER_2FA` | zweiten Faktor einrichten, wechseln, abschalten, neue Wiederherstellungscodes |
| `USER_LOGOUT` | Session beenden |
| `USER_REGISTER` / `USER_REGISTER_CONFIRM` | Profil anlegen (Status 2) / Token einlösen (Status 0) |
| `USER_PW` | Reset-Token für „Passwort vergessen" in `$set['hash']` |
| `USER_INPUT` | Datensatz anlegen oder ändern (s. u.) |
| `USER_DELETE` | eigenen Datensatz oder das eigene Profil in den Papierkorb |
| `USER_BASKET` | Warenkorb (s. u.) |
| `USER_TOKEN` | CSRF-Token, optional mit Formularschutz |
| `USER_PASSWORD_RULES` | Passwort-Richtlinie als JSON für die Extension `Password` im Frontend |

**Kein `USER_*`-Befehl verschickt selbst E-Mails.** Freischalt-Link, Reset-Link,
Anmeldecode und Hinweise („neues Gerät", „Passwort geändert") versendet das
Erfolgstemplate mit `CMS:EMAIL` – die Befehle legen Token, Codes und Ereignisse in
`$set` ab. Wer den Code per SMS will, ruft im Template seinen SMS-Dienst statt
`CMS:EMAIL`.

### `USER_INPUT` – wer darf was schreiben

| Bereich | ohne `ignoreuser` |
|---|---|
| `ugc` | angemeldet, Profil-Bereich des Users in `details.profile_area_ids`; ändern nur eigene Datensätze (`user_id`) |
| `profile` | nur das eigene Profil; kein Anlegen (dafür `USER_REGISTER`) |
| `record` | nie |

`ignoreuser=true` hebt **alle** Prüfungen auf (Kontaktformular ohne Anmeldung) – das
Template trägt dann die Verantwortung. Ablauf: Berechtigung → eigenen Datensatz
auschecken bzw. `tmpInit()` → nur benutzerdefinierte Felder übernehmen (eingeschränkt
durch `fields`, Metafelder nur über `overwrite`) → `user_id`, `needs_review = 1` →
`Record::save()` mit allen Prüfungen des Bereichs. Scheitert das Speichern, wird die
Arbeitskopie samt Uploads verworfen.

**Passwortwechsel** in einem Profil verlangt das bisherige Passwort
(`data[current_password]`) – außer die Session wurde über den Reset-Link angemeldet und
die Freigabe gilt noch. Danach enden alle anderen Sessions des Profils,
`$set['password_changed']` = 1.

**Uploads aus dem Frontend** prüft `Record` strenger als das Backend
(`VirtualFieldsTrait::setUntrustedUploads()`): aktiver Inhalt (HTML, SVG, XML/XSL,
JavaScript – an Endung **oder** erkanntem Mime-Typ, `MediaAccess::isActiveContent()`)
wird immer abgelehnt, auch wenn das Feld ihn erlaubt.

**Owner:** `USER_REGISTER` und `USER_INPUT` (beim Anlegen) setzen `owner_id` und
`created_by` aus dem Parameter `owner` bzw. `user.default_owner_id`; gibt es den Benutzer
nicht, wird nichts gespeichert. `changed_by` bleibt NULL.

### `USER_BASKET`

Warenkorb je Session, `category` trennt mehrere. `function`: `load` (Artikel als
Ergebnismenge für `CMS:RESULT_START` – ersetzt die laufende Ergebnismenge ohne Sicherung),
`add`, `update` (Menge 0 entfernt), `update_addon`. In den Warenkorb darf nur, was
veröffentlicht und für den Request sichtbar ist
(`RecordUser::basketAllowedIds()`), ohne `container` nur aus Bereichen vom Typ `record`
oder `ugc`, mit `container` nur aus den genannten Bereichen. Ein Shop sollte `container`
immer setzen. Dieselbe Prüfung gilt beim Lesen: ein inzwischen offline genommener
Artikel zählt nicht mehr.

## Zweiter Faktor

`area.details.second_factor` je Profil-Bereich (`UserSecondFactor::needed()`):

| Wert | Wirkung |
|---|---|
| `none` (Vorgabe) | nie |
| `optional` | nur mit selbst eingerichtetem Faktor (`USER_2FA`) |
| `new_device` | eigener Faktor, sonst Code per E-Mail auf unbekanntem Gerät (`LoginDevices::isKnown()`) – auch das erste Gerät ist unbekannt |
| `required` | App-Code, wenn eingerichtet, sonst Code per E-Mail |

Ablauf: `USER_LOGIN` prüft das Passwort, legt bei Bedarf eine anonyme Session mit der
wartenden Anmeldung an (`details.mfa_pending`) und bindet `codetemplate` ein, das den
Code aus `$set['login_code']` verschickt. `USER_LOGIN_CODE` prüft den Code (falsche Codes
zählen im `LoginGuard` auf denselben Namen, nach drei verfällt die wartende Anmeldung)
und meldet dann vollständig an. Der Zähler des `LoginGuard` wird erst nach vollständiger
Anmeldung geleert. TOTP-Secret verschlüsselt (`Crypt::encrypt`, AAD `utotp|{id}`),
Replay-Sperre über den letzten Zeitschritt, zehn Wiederherstellungscodes als HMAC.
Zurücksetzen im Backend: Detailansicht des Profils → „Zweiter Faktor".

## Geschützte Medien

Upload- und Thumbnail-Felder haben die Einstellung `details.user_auth`:

| Wert | Wirkung |
|---|---|
| 0 | `CMS:MEDIA` verlinkt direkt ins Medienverzeichnis |
| 1 | `MediaAccess::protectField()` sperrt `media/{instance_id}/{field_id}/` (`Require all denied`); `CMS:MEDIA` verlinkt auf `/file.php?file={field_id}/{record_id}[/{language_id}]`; `MediaDelivery` liefert nur an einen angemeldeten Frontend-User aus einem der Profil-Bereiche in `details.profile_area_ids` |

`file.php` bedient auch `CMS:MEDIA auth=true` und `download="name"`. Jeder Fehlschlag ist
ein 404. Aktiver Inhalt geht immer als `attachment` mit
`Content-Security-Policy: sandbox` hinaus ([14](14-security-web.md#uploads-und-aktiver-inhalt)).

Die Prüfung selbst steckt in `MediaDelivery::resolveFile()` (Instanz, Veröffentlichung,
Feld gehört zum Bereich, `user_auth` mit Profil-Bereich). `CMS:ZIP` (`CmsZip`) nutzt
sie für jede Datei eines Archivs – ins ZIP kommt nur, was auch einzeln über `file.php`
abrufbar wäre.

## Konfiguration (`user.*`)

```
max_inactive_sec 86400   basket_days 30       token_expiration 86400
password_token_expiration 3600                login_code_expiration 600
bruteforce_max 5         bruteforce_delay 600 bruteforce_ip false
cookie_secure true       cookie_samesite Lax  csrf true
throttle […]             formguard_min_seconds 3   formguard_max_seconds 3600
unconfirmed_days 7       default_owner_id 0
```
