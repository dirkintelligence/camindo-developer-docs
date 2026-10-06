# 14 – Sicherheit II: Web

## CSRF

Ein angemeldeter Benutzer öffnet eine fremde Seite; die löst einen Request an das Backend
aus. Schickt der Browser das Cookie mit, liefe der Request unter der Session des Benutzers.
camindo setzt drei Hürden:

| Hürde | Wo |
|---|---|
| **Token** | `Session::csrfToken()` = HMAC über die Session-Id (nichts wird gespeichert). `cConfig.csrf_token`, von `cApp.system.csrfHeaders()` bei **jedem** Request als Header `X-Camindo-Token` mitgeschickt, ersatzweise POST-Feld `_token`, nie aus dem Query-String. `Backend::backendPath()` prüft es bei jeder Methode außer GET/HEAD – auch für Extension-Endpunkte. |
| **POST-Bindung** | Jeder Endpunkt mit Seiteneffekt verlangt POST: `if (!$backend->requirePost()) …`. Die Oberfläche löst ihn per `replacePost()`, `openPost()`, Formular oder Kontextmenü-Eintrag mit `data-method="post"` aus. |
| **Herkunft** | `Backend::originAllowed()`: bei POST muss `Origin` (ersatzweise `Referer`) zum Host des Backends oder zu `admin.allowed_origins` passen; fehlen beide, entscheidet das Token. |

Fehlschläge antworten mit `##accessdenied##` und landen im Log `security`. Dazu
`SameSite=Lax` am Session-Cookie.

**Anmeldedialog bei bestehender Session:** `Start/login.php` ist von der Token-Prüfung
ausgenommen, wenn schon eine gültige Session besteht (z. B. in einem zweiten Fenster
angemeldet, während hier der Dialog offen stand). `StartBackend::login()` meldet dann
nicht erneut an, sondern liefert nur das Token der bestehenden Session und schließt den
Dialog (`cApp.layers.loginComplete()`) – es ändert nichts, und die Antwort kann eine
fremde Seite nicht lesen. Die Herkunftsprüfung gilt weiter.

**Regeln für neuen Code:** Ein GET darf nichts verändern. Ein Neuladen einer per POST
geöffneten Seite wird zu einem GET, den der Server ablehnt – gewollt, eine schreibende
Aktion soll sich nicht durch F5 wiederholen. Was ein `<link>` oder Download-Fenster
braucht (Stylesheets der Editoren, Download einer Exportdatei, `file.php`,
`preview.php`), kommt ohne Token aus und darf deshalb nur lesen.

**Eingabemaske und Arbeitskopie:** Ein GET auf `record_edit.php` (Lesezeichen, F5, Link
von außen, Frontend-Editing) zeigt nur einen Platzhalter mit der Schaltfläche „Bearbeiten";
erst ein POST legt Arbeitskopie und Sperre an. Aus der Oberfläche geöffnete Layer tragen
ein Merkmal und laden die Maske sofort per POST – der Nutzer merkt nichts.

**Frontend:** Token aus `CMS:USER_TOKEN` (HMAC über das Cookie `ucsrf`), Pflicht für alle
schreibenden `CMS:USER_*`-Befehle ([12](12-community.md#schreibschutz-token-drosselung-formularschutz)).

Ein Token schützt nicht vor Skript, das bereits im Backend läuft – es steht in
`cConfig` und ist für eingeschleustes JavaScript lesbar. Die eigentliche Abwehr ist, dass
kein Skript hineinkommt: HTML-Filter und Script-Policy.

## HTML-Filter

**Grundsatz:** HTML in Eingaben ist erlaubt (Rich-Text), aktiver Inhalt nicht. Kein
pauschales `htmlspecialchars()` beim Lesen – Werte gehen auch in URLs, E-Mails, JSON und
zurück in Masken. Stattdessen:

**A – Schreibfilter.** `Dataset::validateValue()` schickt jeden String-Wert (und jedes
String-Element von `json`/`reltable`) ohne `strip_tags` durch
`HtmlSanitizer::clean()`. Ausnahmen: `password` und die Tabelle `template`
(`$sanitizeHtml = false`) – im Template ist Script Inhalt, und aus der Datenbank gelangt
ohnehin nie Code ins Dateisystem.

**B – Lesefilter.** Derselbe Filter auf jeden String-Wert in `Cms::fetch()` (also
`CMS:SELECT`, alle Masken und Listen), in den Felddefinitionen
(`VirtualFieldsTrait::virtualSchema()`) und in den Bezeichnungen des `InstanceCache`.
Damit ist auch geschützt, was an A vorbei in die Datenbank kam (Altdaten, direktes SQL).

**Was `HtmlSanitizer` entfernt:**

| | |
|---|---|
| samt Inhalt | `script`, `iframe`, `object`, `applet`, `frameset`, `noembed`, `noframes` |
| nur das Tag | `embed`, `base`, `meta`, `frame`, SVG-Animationen und -Handler |
| Attribute | `on…`, `srcdoc`, `formaction`, `xlink:href` |
| URL-Werte | `javascript:`/`vbscript:`, `data:` außer Rasterbildern – nach Auflösen von Entities und Entfernen von Steuerzeichen |

Unvollständige Tags werden entschärft, der Filter läuft, bis sich nichts mehr ändert.
`system.html_allowed_tags` schränkt optional auf eine Liste erlaubter Tags ein. Sauberer
Text passiert ohne Kosten.

**C – Kontexte im Backend.** Ein Wert wie `" onmouseover="…` enthält kein `<` – den sieht
kein HTML-Filter:

* `Backend::template()` maskiert `"` in Werten für Platzhalter, die in einem
  Attributwert stehen.
* `Backend::jsString($wert)` erzeugt ein sicheres JS-String-Literal (für `<script>` und
  Handler).
* Text-Inhalte, die nicht aus `fetch()` kommen (Request-Werte, Dateinamen, Meldungen
  mit Nutzereingaben), maskiert der Aufrufer mit `htmlspecialchars()`.

**Frontend:** `CMS:TEXT` gibt Werte absichtlich roh aus (HTML ist Inhalt); dort greift
der Lesefilter. In Attributen maskiert das Template mit `(CMS:TEXT feld
specialchars=true)`.

## Sicherheits-Header

`SecurityHeaders::send()` läuft am Ende des Bootstraps für jeden Web-Einstieg:

| Kontext | Header |
|---|---|
| Backend | `X-Frame-Options: DENY`, `Content-Security-Policy: frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; script-src …`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin` |
| Frontend, Vorschau | `X-Frame-Options: SAMEORIGIN`, `Content-Security-Policy: frame-ancestors 'self'`, `nosniff`, `Referrer-Policy: strict-origin-when-cross-origin` |
| beide, auf Wunsch | `Strict-Transport-Security` über HTTPS, `security.hsts_max_age` (Vorgabe 0 = aus) |

Im Frontend sind das Voreinstellungen: `(CMS:HEADER "Name: Wert")` am Template-Anfang
ersetzt einen Header, `(CMS:HEADER "Name:")` ohne Wert entfernt ihn. Wegen `nosniff` muss
ein Template, das CSS, JavaScript oder JSON ausliefert, seinen `Content-Type` setzen.

## Script-Policy im Backend

```
Content-Security-Policy: … script-src 'self' 'nonce-{nonce}' 'report-sample'
```

Scripts laufen nur aus eigenen Dateien oder als `<script>` mit der Nonce der Seite. Kein
`'unsafe-inline'`, kein `'unsafe-eval'`, kein `'strict-dynamic'` (jQuery würde sonst
jedes eingeschleuste `<script>` eines nachgeladenen Stücks ausführen).

**Eine Nonce je Seite.** Das Backend ist eine Seite, in die `cLoadContent` Stücke lädt;
es gilt die Policy aus dem Header von `index.php`. `backend-handlers.js` schickt die
Nonce der Seite als Header `X-Camindo-Nonce` mit, `SecurityHeaders::nonce()` übernimmt
sie für die Ajax-Antwort – jedes Stück trägt so die Nonce der Seite.

**Handler markieren statt ausführen.** `Backend::template()` schreibt Handler in
Vorlagen um, bevor Platzhalter ersetzt werden:

```
onclick="CODE"          →  data-c9h-{nonce}-click="CODE"
href="javascript:CODE"  →  href="#" data-c9h-{nonce}-js="CODE"
<script>                →  <script nonce="{nonce}">
```

`backend-handlers.js` sammelt die Attribute mit der Nonce der Seite, erzeugt **ein**
Script-Element mit Nonce, das je Handler eine Funktion definiert, und bindet sie per
`addEventListener` (ein `MutationObserver` erfasst jedes eingefügte HTML). Code wird nur,
was die Nonce im **Attributnamen** trägt – und die steht nur im Code des Servers, nie in
Inhalten aus der Datenbank. Ein eingeschleustes `onclick` bleibt ein wirkungsloses
Attribut.

**Regeln** (Kurzfassung in [05](05-backend-development.md#javascript-api-windowcapp)):

```php
// Vorlage – wird markiert
<button type="button" onclick="{onclick}">…</button>

// HTML aus PHP
echo '<a href="#"' . $backend->on('js', "cApp.layers.replace('X/y.php?id={$id}')") . '>…</a>';
echo '<select' . $backend->on('change', 'cApp.forms.updateElement(this)') . '>';
'attr' => ['onchange' => '…']                      // formField(): wird markiert

// Script in der Ajax-Antwort
$backend->script("$('#x').focus();");
echo '<script' . $backend->nonceAttr() . '>…</script>';
```

Nicht möglich: `onclick` oder `javascript:` roh im `echo` bzw. als ganzer Platzhalterwert,
`eval`/`new Function`/`setTimeout('string')`, HTML mit Handlern per `innerHTML`.

`security.backend_csp`: `enforce` (Vorgabe) | `report` (alles läuft, Verstöße werden nur
gemeldet – für Komponenten im Umbau) | `off`. Verstöße meldet `backend-handlers.js` an
`Start/csp_report.php`; Protokoll `var/log/{Datum}_csp.log`.

## Script-Policy im Frontend

Je Instanz zuschaltbar (Instanz-Maske, `instance.details`; Vorgabe `security.frontend_csp
= off`): `csp` = `off` | `report` | `enforce`, weitere Script-Quellen,
`'strict-dynamic'`, `form-action`. Quellen prüft `SecurityHeaders::cleanSources()` – nur
Host-Angaben, keine Schlüsselwörter, keine reinen Schemata.

```
script-src 'self' 'nonce-N' [Quellen] ['strict-dynamic']; object-src 'none'; base-uri 'self';
frame-ancestors 'self'[; form-action 'self' …]; report-uri /csp_report.php
```

* **Die Nonce setzt der Template-Programmierer selbst:** `<script(CMS:NONCE)>`. Der
  Compiler setzt nichts ein; Inhalte aus der Datenbank tragen nie eine Nonce – genau die
  Trennung, um die es geht. Der Parser meldet `<script>` ohne Nonce, `on…`-Attribute und
  `javascript:` als Hinweis.
* `(CMS:CSP script=… form=… mode=…)` ergänzt Quellen bzw. den Modus für eine Seite (vor
  der ersten Ausgabe).
* Nur Scripts werden begrenzt; wer mehr will, schreibt die ganze Policy mit
  `CMS:HEADER` und holt die Nonce mit `(CMS:NONCE format=value)`.
* Meldungen an `public/csp_report.php` (`CspReport`: nur POST, klein, gedrosselt).

## Uploads und aktiver Inhalt

Eine SVG- oder HTML-Datei unter `/media` liegt im Ursprung der Website; direkt aufgerufen
führt der Browser eingebettetes Script aus. Drei Schichten:

| Schicht | Wirkung |
|---|---|
| Upload-Prüfung | gesperrte Endungen (`php`, `phar`, `htaccess` …) immer; Endung muss zum erkannten Mime-Typ passen (`error_file_mismatch`); die Gruppe `image/*` passt nicht auf SVG |
| Frontend-Uploads | aktiver Inhalt (HTML, SVG, XML/XSL, JavaScript – Endung **oder** Mime-Typ) wird immer abgelehnt, unabhängig von den Feldeinstellungen |
| Auslieferung | `nosniff` für alle Dateien; aktiver Inhalt zusätzlich mit `Content-Security-Policy: sandbox` und `Content-Disposition: attachment` (Instanz-`.htaccess`, `MediaDelivery`, `backend/file.php`) |

## ImageMagick

ImageMagick erkennt das Format am Inhalt, nicht an der Endung – eine `bild.jpg` mit
Skript-Inhalt würde als Skriptformat verarbeitet. Der Prozessor `Magick` sichert ab:

* **Coder-Präfix:** Quelle und Ziel werden mit dem Coder des serverseitig erkannten
  Mime-Typs übergeben (`jpeg:/pfad/datei[0]`); ein unbekannter Typ ist `unsupported`.
* **Eigene Policy, immer aktiv:** `MagickPolicy` schreibt `var/config/magick/policy.xml`
  und setzt `MAGICK_CONFIGURE_PATH` für alle Aufrufe. Sie kann die Policy des Systems
  nur verschärfen: Grenzen für Speicher, Zeit, Maße und Frames; gesperrt sind
  Skript- und Pseudo-Formate (MSL, TEXT, URL, FILE …), die PostScript-Familie und alle
  Delegates.
* **SVG und PDF nur auf Wunsch:** `security.magick_allow = ['svg']` bzw. `['pdf']`
  (PDF braucht Ghostscript). Die Detailansicht des Konverters zeigt per Probelauf, was
  tatsächlich läuft.
* **Versionswarnung** unter ImageMagick 7.1.1 in der Konverter-Ansicht.

Die Policy greift für die Programme, nicht für die PHP-Erweiterung Imagick – ein
Zusatzmodul, das Imagick nutzt, arbeitet unter der Policy des Systems. Wer ImageMagick
selbst aufruft, ruft vorher `MagickPolicy::apply()`.

## Feste Backend-Domain

`admin.backend_domain` (z. B. `cms.example.com`) bindet das Backend an eine Domain –
Voraussetzung für Passkeys, und eine Domain, die keine Website verrät, hält
Anmeldeversuche fern.

| Request | Verhalten (`BackendDomain`) |
|---|---|
| Backend-Einstieg unter anderer Domain | 404 (`enforce()` im Bootstrap), kein Hinweis auf die richtige Domain |
| Frontend unter der Backend-Domain | 302 auf das Backend, wenn die Domain keiner Instanz gehört |
| `/backend/assets`, `/backend/ext` | unter jeder Domain (statische Dateien) |

`BackendDomain::url($pfad)` baut absolute Backend-Adressen; ohne feste Domain bleibt es
bei relativen.

**Vorschau über die Website-Domain** (`FrontendHandover`): `preview.php` prüft Session
und Instanzrecht und leitet mit einem signierten Token auf die Hauptdomain der Instanz
um (`?c9preview=…`) – mit fester Backend-Domain immer, sonst sobald das Backend nicht
unter einer Domain der Instanz läuft (die Medien-`.htaccess` ließe unter fremder Domain
keine Bilder zu). Das Token enthält Instanz, Pfad, Sprache, Benutzer, ein Kennzeichen der
Session (nicht die Session-Id) und einen Ablauf; gültig nur, solange die Backend-Session
lebt und die Zeile des Benutzers in `admin` gültig signiert ist, und nur unter einer Domain der Instanz.

### Frontend-Editing

Das Cookie `fed` trägt einen signierten Wert `{Modus}.{admin_id}.{Ablauf}.{HMAC}`;
`Cms::fedActive()` prüft ihn ohne Datenbankabfrage. Unter der Website-Domain setzt es die
Vorschau; `cFED()` öffnet das Backend über `BackendDomain::url()`.

Außerdem je Website eine Schaltfläche in den persönlichen Einstellungen
(`AdminBackend::_settingsFedSites()`): ein Link `https://{domain}/index.php?c9fed={token}`
(`FrontendHandover::fedUrl()`, Modus = gespeicherte Einstellung, eine Stunde gültig, an die
Session gebunden wie die Vorschau). `Page::switchFedOn()` prüft Token und Domain, setzt das
Cookie und leitet auf `/` weiter; ungültig → 403. Die Knöpfe gibt es auch ohne feste
Backend-Domain, für alle Hauptdomains außer der, unter der das Backend läuft.

**Abgleich mit allen Websites.** Ein Cookie für eine fremde Domain lässt sich weder per
Ajax noch per iframe zuverlässig setzen (Drittanbieter-Cookies werden gesperrt bzw.
partitioniert). Nach dem Speichern einer geänderten Einstellung – und über die Schaltfläche
„Auf allen Websites übernehmen" – läuft der Browser deshalb als normale Navigation durch
die Websites: `FrontendHandover::fedSyncData()` signiert die Liste der Domains, den Modus
(auch 0 = Cookie löschen) und die Rücksprung-Adresse ins Backend; der Parameter `c9hops`
wählt nur Indizes aus dieser Liste (kein offener Redirect). `cApp.ui.fedSync()` prüft
jede Domain zuvor mit einem Bild (`c9probe=1`, 1×1-GIF nur bei passendem Token und
Domain), nicht erreichbare oder fremde Domains fallen aus der Kette. `Page::fedSyncHop()`
setzt das Cookie und leitet zur nächsten Website bzw. zurück; der Hash der Backend-URL
bleibt über die Weiterleitungen erhalten.
