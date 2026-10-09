# 15 – Konventionen

## Namen

### Dateien

| Art | Stil | Beispiel |
|---|---|---|
| Klassen, Interfaces, Traits | PascalCase | `AdminBackend.php`, `TreeManagerTrait.php` |
| View- und Konfigurationsscripte | snake_case | `admin_edit.php`, `settings.php` |
| HTML-Vorlagen | snake_case | `admin_table_row.html` |
| Assets (JS/CSS) | kebab-case | `backend-layers.js` |

### PHP (PSR-12)

| Art | Stil | Beispiel |
|---|---|---|
| Klasse, Interface, Namespace | PascalCase | `Camindo\Core\Modules\Admin` |
| Methode, Funktion, Variable, Property | camelCase | `sendEmail()`, `$isAuthorized` |
| Konstante | UPPER_CASE | `DEFAULT_PORT` |

Private Hilfsmethoden einer öffentlichen Methode tragen deren Namen mit Unterstrich als
Präfix: `edit()` → `_editSave()`, `_editForm()`.

### JavaScript

Klassen PascalCase, Funktionen und Variablen camelCase, globale Konstanten UPPER_CASE,
private Felder mit `#`. Globale Objekte nur `window.cApp` und `window.cConfig`.

### CSS

Ids und Klassen kebab-case, Varianten nach BEM mit `--`. Projekteigene Klassen tragen
das Präfix `c9-` (`c9-layer`, `c9-ctx-row`). Anpassungen an Bootstrap selbst gehören
nicht in `backend.css`, sondern in den Build der Bootstrap-Dateien.

### Datenbank

| Element | Konvention |
|---|---|
| Tabelle | Singular, snake_case: `admin`, `data_txt`, `rel_area_template` |
| Primärschlüssel | `{tabelle}_id` (Ausnahme: `bgproc.id`) |
| Kantentabelle | `rel_{a}_{b}` |
| Datentabelle | `data_{suffix}` nach `Cms::$dataSuffix` |
| Index / Fremdschlüssel | `idx_*` / `fk_*` |

### Snippet-Schlüssel

| Muster | Beispiel |
|---|---|
| `field_{tabelle}_{spalte}` | `field_admin_name` |
| `message_{tabelle}_{aktion}` | `message_admin_saved` |
| `alert_{tabelle}_{aktion}` | `alert_admin_delete` |
| `error_{was}` | `error_value_mandatory` |
| `button_{name}`, `headline_{was}`, `tooltip_field_{name}` | `button_back` |
| `speak_{was}` (Screenreader) | `speak_open_menu` |
| `access_{recht}` | `access_publish` |
| `module_label`, `module_description` | – |

Mit Präfix: `admin.field_admin_name`, `proc.imagegd.field_width`,
`ext.markdownvisual.label`. Schlüssel enthalten keine Punkte.

## Code-Stil

* Konsequent typisieren, auch Rückgabewerte; Constructor Property Promotion für
  Abhängigkeiten.
* Klammern: Klassen und Funktionen in der nächsten Zeile, Kontrollstrukturen in
  derselben.
* Bezeichner (Klassen, Methoden, Variablen, Spalten, Snippet-Schlüssel) und Kommentare
  englisch (US-Schreibweise). Verweise auf diese Dokumentation als
  `developer docs, chapter NN (Titel)`. Texte für Benutzer gehören in Snippets, nicht in
  den Code.
* Eingaben immer typisiert übernehmen (`(int)$_REQUEST['id']`); `bit` über
  `filter_var(…, FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE)`.
* Includes schützen: `if (!defined('CAMINDO_STARTED')) die('access denied.');`
* Kapselung mit einer IIFE, wo globaler Namensraum sauber bleiben muss (so bindet
  `Page` Templates ein).

| Konstante | gesetzt in |
|---|---|
| `CAMINDO_STARTED` | jedem Einstiegsscript |
| `CAMINDO_BACKEND` | Backend-Einstiegen (`index.php`, `module.php`, `file.php`, `preview.php`) |
| `CAMINDO_PREVIEW` | `backend/preview.php` |
| `CAMINDO_CLI` | Scripts unter `Core/cron/` |
| `CAMINDO_VERSION` | `Core/Core.php` |

## Ausgabe und Header

* **Das Frontend puffert nicht.** Header und Cookies gehören an den Anfang eines
  Templates, vor die erste Ausgabe – `CMS:HEADER`, `CMS:CSP`, `CMS:USER_*` und der erste
  `CMS:USER_TOKEN`.
* Im Backend gibt eine Backend-Methode HTML aus; Widgets **geben es zurück**.
* Werte in HTML maskieren, die nicht aus `fetch()` kommen; in Attributen `"`
  maskieren, in JavaScript `Backend::jsString()` ([14](14-security-web.md#html-filter)).
* Keine Handler-Attribute im `echo`, kein `eval` ([14](14-security-web.md#script-policy-im-backend)).

## SQL

* **Immer Prepared Statements**: `Database::query($sql, $bindTypes, $values)`.
  Bezeichner vorher auf `[a-z0-9_]` bereinigen.
* Schreibende Mehrschritt-Vorgänge in einer Transaktion (`Dataset::transactional()`).
* Unterabfragen während einer Iteration mit `pushResult()`/`popResult()` klammern.
* Datums- und Zeitwerte in UTC speichern (`UTC_TIMESTAMP()`), für die Anzeige umrechnen.
* In Feldwerten nur SQL-Funktionen aus `Cms::$funcWhitelist`.
* Lesezugriffe auf Inhalte über `Cms::select()` – dann gelten Instanz-, Bereichs- und
  Löschbedingungen automatisch. Wer direkt abfragt, hängt sie selbst an.
* Tabellen mit Rechte-Signaturen nur über die Klassen schreiben, die signieren
  ([13](13-security-authentication.md#signaturen)).

## Sprache, Zeitzone, Schreibrichtung

| Kontext | Zeitzone | Snippet-Sprache |
|---|---|---|
| Backend | `admin.settings.timezone` | `admin.settings.backend_code` |
| Frontend | `language.timezone`, leer = `system.timezone` | `language.backend_code` |
| CLI, ohne Session | `system.timezone` | `en_US` |

Gespeichert wird in UTC, angezeigt in `Cms::$userTZ`. Datums- und Zahlenformate folgen
im Backend der Locale des Benutzers bzw. `system.locale`, im Frontend der Inhaltssprache
(`language.code`, sonst `language.backend_code`).

**Schreibrichtung (RTL).** Ist eine Backend-Sprache aus `Cms::RTL_LANGUAGES` eingestellt
(Arabisch, Persisch, Urdu, Hebräisch …), trägt das Backend `<html dir="rtl">`;
`Cms::isRtl()` / `Cms::textDirection()` liefern die Richtung für eigene Ausgaben (etwa
E-Mails). Damit Module und Extensions mitdrehen:

* logische CSS-Eigenschaften für Asymmetrisches: `margin-inline-start/-end`,
  `padding-inline-*`, `border-inline-*`, `inset-inline-*`, `text-align: start/end` –
  auch in Inline-Styles der Vorlagen
* Bootstrap-Utilities in BS5-Schreibweise (`ms-*`, `me-*`, `ps-*`, `pe-*`,
  `text-start`, `float-end`)
* Icons mit Richtung (Pfeile, Chevrons) bekommen die Klasse `icon-dir`
* JavaScript, das mit Pixelpositionen rechnet, fragt
  `document.documentElement.dir === 'rtl'`

## Dateirechte und schreibbare Verzeichnisse

Vom CMS erzeugte Dateien `0664`, Verzeichnisse `0775` (Vorgabe, konfigurierbar in
`permissions.file` / `permissions.dir`), Eigentümer der Webserver-Benutzer; `.camindo.php`, `.camindo.key`,
`.sqlpwd` mit `0600` (`key rotate` behält die Rechte der bisherigen `.camindo.key`, ohne
Datei `permissions.key_file`). Verzeichnisse legt `System::createDir()` mit den konfigurierten
Rechten an.

Schreibbar ist nur `var/` und `public/media/`:

```
var/cache/       Snippets, HTML-Vorlagen, Instanzen, Benutzergruppen, Trigger, Update-Prüfung
var/config/      Laufzeitlisten, Versionsstempel, ImageMagick-Policy
var/templates/   Quelltexte der Templates          var/php/      kompilierte Templates
var/custom/      Snippets je Instanz (snippets/), Zeitplaner: Scripte (php/), Ausgaben (html/)
var/extensions/  projekteigene Dateien der Extensions
var/audit/       Inhalte des Audit-Logs            var/log/      Protokolle
var/tmp/ var/upload/ var/import/ var/export/ var/download/ var/deleted/ var/dbbackup/
public/media/    Uploads und abgeleitete Medienversionen
```

`var/` liegt außerhalb des Docroots.

## Tests

camindo bringt keine Test-Suite mit, aber Selbsttests für heikle Stellen
(`Core/cron/check_*.php`) und den headless-Weg für Backend-Scripte
([05](05-backend-development.md#ohne-browser-testen)). Nach Änderungen an `Dataset`,
Bäumen oder dem Template-Compiler die Selbsttests laufen lassen.
