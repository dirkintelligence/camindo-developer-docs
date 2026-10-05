# 01 – Architektur

## Verzeichnisse

```
projekt/                     Projektwurzel (Skelett camindo/cms)
├── composer.json            PSR-4: Camindo\ => ./ ; der Core bringt Camindo\Core\ mit
├── load_camindo.php         lädt Konfiguration und Bootstrap
├── .camindo.php             Konfiguration der Installation   – vom Installer, nie im Git
├── .camindo.key             Schlüssel für Verschlüsselung, Signaturen, Pfeffer
├── .sqlpwd                  Zugangsdaten für mysqldump
│
├── public/                  DOCUMENT_ROOT
│   ├── index.php file.php csp_report.php     Stubs → Core/public_master/…
│   ├── .htaccess            Rewrite-Regeln, teilweise generiert
│   ├── media/               Medienablage {instance_id}/{field_id}/{record_id}.{ext}
│   └── backend/
│       ├── index.php module.php preview.php file.php     Stubs
│       ├── assets → ../../Core/public_master/backend/assets   (Symlink oder Kopie)
│       └── ext/{name}/      veröffentlichte Assets der Extensions
│
├── Core/                    Paket camindo/core – wird bei Updates als Ganzes ersetzt
│   ├── Core.php             Bootstrap
│   ├── src/                 Kernklassen (Namespace Camindo\Core)
│   ├── Modules/ Processors/ Extensions/      Core-Komponenten
│   ├── lang/ backend/       Core-Snippets und -Vorlagen
│   ├── cron/                Einstiegspunkte der Kommandozeile
│   ├── install/             Datenbankschema, Migrationen, Vorlagen, Skelett
│   ├── docs/                Handbücher (Markdown, je Sprache)
│   └── public_master/       Einstiegsscripte und Backend-Assets, auf die die Stubs zeigen
│
├── Modules/ Processors/ Extensions/          Zusatzkomponenten (Composer-Ziel)
├── var/                     alles Schreibbare
└── vendor/                  Composer
```

`Core/public_master/` heißt bewusst nicht `public`: beim Einrichten des Webservers soll
niemand den Docroot auf ein Verzeichnis im Core legen. Composer schreibt nur in
`Core/`, `Modules/`, `Processors/`, `Extensions/` und `vendor/`; alles, was eine
Installation individuell macht (`public/`, `var/`, die Punkt-Dateien), liegt außerhalb.

## Bootstrap

Es gibt genau einen Weg ins System:

```
Einstiegsscript
  └── define('CAMINDO_STARTED', true)       Schutz gegen Direktaufruf der Includes
      require load_camindo.php
        ├── require .camindo.php            Konfiguration → Array $_cfg (fehlt sie: [])
        └── require Core/Core.php
              ├── PHP-Versionsprüfung, define('CAMINDO_VERSION', …)
              ├── require vendor/autoload.php
              ├── new Container(); Config aus $_cfg registrieren; unset($_cfg)
              ├── $GLOBALS['app'] = $container
              ├── BackendDomain::enforce()     nur Backend-Einstiege
              └── SecurityHeaders::send()      vor jeder Ausgabe
```

`load_camindo.php` liegt bewusst außerhalb von `Core/`: es gehört dem Projekt und bleibt
bei Core-Updates erhalten. `.camindo.php` prüft selbst, ob `CAMINDO_STARTED` definiert
ist. In `.camindo.php` dürfen **keine `$_SERVER`-Variablen** stehen – die Datei wird auch
auf der Kommandozeile geladen.

Fehlt `.camindo.php`, läuft der Bootstrap mit leerer Konfiguration weiter; Frontend und
Backend zeigen dann den Hinweis, dass die Installation aussteht (`SystemState`).

### Konfiguration

`Config` bekommt das `$_cfg`-Array, mischt es rekursiv über eigene Vorgaben und leitet
Pfade ab (`paths.core_dir`, `paths.camindo_dir`, `paths.var_dir`, `paths.media_dir`,
`paths.backend_dir`). Zugriff in Punkt-Schreibweise:

```php
$config->get('paths.media_dir');
$config->get('admin.max_inactive_sec');
```

Jeder Schlüssel hat eine Vorgabe in `Config`; `.camindo.php` enthält nur, was davon
abweicht. Die kommentierte Vorlage aller wichtigen Schlüssel ist
`Core/install/templates/camindo.php.dist`. Der Konstruktor stellt außerdem Locale,
Zeitzone und das PHP-Fehlerprotokoll (`var/log/{Datum}_php_errors.log`) ein.

## Dependency-Injection-Container

`Container` implementiert `ContainerInterface` (`get()`, `has()`, `set()`) und löst
Klassen per **Autowiring** auf:

1. schon erzeugte Instanz → zurückgeben
2. registrierte Factory (`set()`) → aufrufen, Ergebnis merken
3. sonst Konstruktor-Parameter per Reflection rekursiv auflösen

Aufgelöst werden nur Klassen-Typen. Skalare Konstruktor-Parameter führen zu einer
Exception – solche Klassen müssen per `set()` registriert werden (im Core nur
`Config`). Jede Klasse ist ein **Shared Service**: `$app->get(Admin::class)` liefert
innerhalb eines Requests immer dieselbe Instanz, auch bei Dataset-Klassen. Zyklische
Abhängigkeiten werden nicht erkannt; halten Sie Konstruktoren zyklenfrei oder laden Sie
selten gebrauchte Dienste nach (`LazyServicesTrait`).

```php
$app = $GLOBALS['app'];                       // in jedem Einstiegsscript vorhanden
$adminBackend = $app->get(AdminBackend::class);
```

`ProcessorFactory` löst Prozessoren über ihren Namen auf – über die Laufzeitliste der
aktiven Komponenten, ein Zusatzprozessor gleichen Namens hat Vorrang vor dem Core.

## Die vier Request-Wege

### 1. Frontend

```
Browser → public/.htaccess → public/index.php → Core/public_master/index.php
        → Page::resolveRequest()
```

Der generierte Block in `public/.htaccess` (zwischen `## CAMINDO SYNTAX START` und
`END`, mit Prüfsumme – nicht von Hand ändern, geschrieben von `HtaccessWriter`) enthält:

| Regel | Ziel |
|---|---|
| `^backendModules/(.*)$` | `/backend/module.php?_path=$1` |
| `^backendExtensions/(.*)$` | `/backend/module.php?_ext=$1` |
| `^([a-zA-Z]{2}(?:[-_][a-zA-Z]{2})?)/(.*)$` | `/index.php?lang=$1&request=$2` |
| `^(.*)$` (keine Datei, nicht `/backend`, nicht `/media`) | `/index.php?request=$1` |

`Page::resolveRequest()` arbeitet in dieser Reihenfolge:

1. **Instanz** aus dem Host: `primary_domain`, dann `rel_instance_domain` (wörtlich oder
   als regulärer Ausdruck, in der Reihenfolge der Instanzen), sonst die Standard-Instanz.
   Die Reihenfolge der Instanzen ist systemweit `Instance::orderSql()`: nach `position`,
   die Standard-Instanz immer zuletzt. Quelle ist der Dateicache
   `var/cache/instances.php` + `instance-{id}.php` (`InstanceCache`), keine Datenbank.
   Daraus `Cms::setInstanceAccess()`: eigene Instanz plus die Instanzen, die sie lesen darf.
2. **Sprache**: Das Kürzel aus der URL muss eine Sprache der Instanz sein. Fehlt es und
   führt die Instanz mehrere Sprachen, entscheidet `Accept-Language`
   (`InstanceCache::negotiateLanguage()`), sonst gilt die Hauptsprache.
   `Cms::setLanguage()` stellt damit Inhaltssprache, Zeitzone (`language.timezone`),
   Snippet-Sprache (`language.backend_code`) und die Locale der Datumsformate ein
   (`language.code`, sonst `backend_code`; es genügt, dass ICU die Sprache kennt).
3. **Datensatz**: `request` wird an Kommata zerlegt – `{slug|id},{template},…`. Eine
   reine Ziffernfolge ist eine `record_id`, alles andere ein Pfad
   `[slug-prefix/]slug` (`Cms::recordIdsBySlugPath()`). Leer = Startseite der Instanz.
4. **Template**, in dieser Rangfolge: aus der URL (nur mit Berechtigung, siehe
   [06](06-template-compiler.md#template-zuweisung)), `record.template_id`,
   `area.default_template_id`.
5. **Kanonische URL**: bei `instance.canonical = 1` wird jede abweichende Adresse mit
   301 auf die eine gültige umgeleitet (Startseite ausgenommen).
6. Der Datensatz muss **veröffentlicht** sein (`Cms::publishedSql()`: Status 0,
   `online_at` erreicht); ein abgelaufener bleibt unter seiner Adresse erreichbar.
7. Die kompilierte Datei `var/php/{area_id}/{name}.php` wird innerhalb einer IIFE
   eingebunden; `$cms` (aktueller Datensatz) und `$set` sind darin global verfügbar.

Weicht der Versionsstempel `var/config/core.php` von `CAMINDO_VERSION` ab oder ist der
Wartungsmodus aktiv, antwortet das Frontend mit 503, bevor die Datenbank angesprochen
wird ([09](09-installation-and-update.md#wartungsmodus)).

**Vorschau aus dem Backend:** `backend/preview.php` → `Page::resolvePreview()`, derselbe
Ablauf mit Benutzer-Session, der Instanz als Parameter, ohne Veröffentlichungsprüfung des
aufgerufenen Datensatzes und ohne kanonische Umleitung. Mit fester Backend-Domain wird
die Vorschau per signiertem Token an die Website-Domain übergeben
([14](14-security-web.md#feste-backend-domain)).

### 2. Backend-Oberfläche

`backend/index.php` liefert das statische Gerüst (Kopfzeile, Layer-Container, Aside,
Hauptmenü, Anmeldedialog), setzt `window.cConfig` (Pfade, CSRF-Token, Nonce,
Extension-Manifest, Upload-Grenze) und lädt die Scripts:

```
backend.js            Namespace window.cApp, jQuery-Erweiterungen (cLoadContent, serializeFiles)
backend-handlers.js   Nonce-Header, Binden markierter Event-Handler (Script-Policy)
backend-system.js     cApp.system    Spinner, Meldungen, CSRF-Header
backend-forms.js      cApp.forms     Änderungsverfolgung, Ajax-Submit
backend-ui.js         cApp.ui        Lookups, Reiter, QR-Codes …
backend-layers.js     cApp.layers    Layer-Stapel und Hash-Routing
backend-aside.js      cApp.aside     Seitenleiste
backend-ctx-menu.js   cApp.ctxMenu   Kontextmenüs
backend-polling.js    cApp.polling   Hintergrundprozesse
backend-smartrows.js, backend-embedded.js, backend-extensions.js, backend-mail.js
backend-init.js       ruft alle init() auf
```

### 3. Backend-Ajax

Alle dynamischen Inhalte laufen über **eine** jQuery-Erweiterung:

```js
$('#ziel').cLoadContent({ url: 'Admin/admin_list.php?id=5', method: 'POST', params: fd });
```

`cLoadContent()` baut die URL `/backendModules/{pfad}`, bricht einen laufenden Request
für dasselbe Element ab, schickt CSRF-Token und Nonce als Header mit, zeigt einen
Spinner, erkennt die Antwort `##accessdenied##` (öffnet den Anmeldedialog) und setzt
danach den Fokus.

Serverseitig:

```
/backendModules/{Modul}/{script}
  → /backend/module.php?_path={Modul}/{script}
      → Backend::backendPath()
          ├── Pfad bereinigen, Komponente über die Laufzeitliste auflösen
          ├── Session::load()             ohne Session: ##accessdenied## (Ausnahme Start/login.php)
          ├── Backend::originAllowed()    fremde Origin bei POST → ##accessdenied##
          ├── Session::verifyCsrfRequest() fehlendes Token bei POST → ##accessdenied##
          └── Access::script()            Modulrecht, Zusatzrecht des Scripts
      → require {Komponente}/backend/{script}
```

`/backendExtensions/{Extension}/{script}` läuft genauso, verlangt aber nur eine Session
(Extensions haben keine Modulrechte). Weitere Einstiege: `backend/file.php` für **jede**
Mediendatei im Backend, `public/file.php` (`MediaDelivery`) für geschützte Dateien und
Downloads im Frontend.

### 4. Kommandozeile und Cron

```
php Core/cron/camindo.php <befehl>      Installation, Update, Caches, Komponenten, Schlüssel …
php Core/cron/cron.php                  minütlich per Cron: TriggerCron (Hintergrundarbeit)
php Core/cron/trash.php                 nachts: Papierkorb leeren
```

Alle Scripts unter `Core/cron/` verweigern den Dienst, wenn `PHP_SAPI !== 'cli'`, und
setzen `CAMINDO_CLI`. Einzelheiten: [07](07-media-processing.md#cron) und
[09](09-installation-and-update.md#die-kommandozeile).

## Layer und Hash-Routing im Backend

Der URL-Hash bildet den Zustand des Backends ab; mehrere mit `~` verkettete Pfade sind
übereinanderliegende Layer:

```
#Record/record_list.php?area=8~Record/record_edit.php?id=42
 └── Layer 0 ──────────────┘ └── Layer 1 ─────────────────┘
```

| Methode | Wirkung |
|---|---|
| `cApp.layers.add(path)` | neuen Layer oben auflegen |
| `cApp.layers.replace(path)` / `replacePost(path, params)` | obersten Layer ersetzen (GET bzw. POST) |
| `cApp.layers.close()` | obersten Layer schließen |
| `cApp.layers.refresh()` | obersten Layer neu laden |
| `cApp.layers.invalidate([callback], [levels])` | den Layer darunter (bzw. `levels` Layer) als veraltet markieren – er lädt neu, sobald er wieder oben liegt |
| `cApp.layers.removeReferer()` | den Layer darunter entfernen |
| `cApp.aside.open(path)` / `openPost(path)` | Seitenleiste laden |

Nach Speichern, Löschen oder Import gilt einheitlich:
`cApp.layers.invalidate(); cApp.layers.close();` (bzw. `replace()` statt `close()`).

## Kernklassen (`Core/src/`)

| Klasse | Aufgabe |
|---|---|
| `Cms` | Herzstück: Feldtypen, Schema von `record`, `select()`/`fetch()`, Snippets, Formatierer, Laufzeit der `(CMS:…)`-Befehle |
| `Dataset` (abstrakt) | Tabellen-Abstraktion: `init/load/save/delete/restore`, Validierung, Hooks, Audit |
| `VirtualFieldsTrait` | virtuelle Felder, Uploads, Thumbnails, Relationen, Volltext |
| `TreeManagerTrait` | Nested Set + Adjacency List, ein Baum je Inhaltsbereich |
| `TemporaryTrait` | Arbeitskopien: `checkOut`, `checkIn`, `cancel`, Bearbeitungssperren |
| `LinkTrait`, `MoveTrait` | fest verknüpfte Inhalte; Verschieben zwischen Bereichen |
| `Backend` | UI-Bausteine: Vorlagen, Formularfelder, Reiter, Buttons, Listen, Script-Policy |
| `Page` | Frontend-Request auflösen, Vorschau, Template ausführen |
| `Database` | mysqli-Wrapper: Prepared Statements, Result-Stack, Transaktionen, `afterCommit()` |
| `Config`, `Container`, `System` | Konfiguration, DI, Logging/Dateien/Verzeichnisse |
| `Session`, `Access`, `Integrity` | Benutzer-Session, Rechte der Benutzergruppen, Signaturen – [11](11-permissions.md), [13](13-security-authentication.md) |
| `UserSession`, `MediaDelivery`, `MediaAccess` | Frontend-Community und geschützte Medien – [12](12-community.md) |
| `InstanceCache` | Dateicache für Instanzen, Domains, Sprachen, Bereiche und Gruppenrechte |
| `Crypt`, `KeyRotation`, `PasswordPolicy`, `Totp`, `LoginGuard`, `LoginDevices`, `RateLimiter`, `FormGuard`, `ClientIp` | Sicherheit – [13](13-security-authentication.md) |
| `SecurityHeaders`, `HtmlSanitizer`, `BackendDomain`, `FrontendHandover`, `CspReport` | Web-Sicherheit – [14](14-security-web.md) |
| `AuditLog`, `AuditStore`, `AuditDiff` | Audit-Log |
| `MediaProcessor`, `ProcessorFactory`, `Execute`, `BackgroundProcess`, `MassDeletion` | Medien und Hintergrundprozesse – [07](07-media-processing.md) |
| `ComponentManager`, `SnippetCache`, `HtmlCache`, `HtaccessWriter`, `SystemState`, `Backup`, `UpdateCheck`, `SystemHealth` | Komponenten, Caches, Betrieb – [08](08-components-and-extensions.md), [09](09-installation-and-update.md) |
| `Pagination`, `Email`, `History`, `Widgets`, `TriggerTemplates` | Blätterleisten, E-Mail-Versand, „zuletzt bearbeitet", Startseiten-Widgets, Trigger-Templates |

### `Database` – der Result-Stack

`Cms::select()` wird in Templates verschachtelt aufgerufen. `Database` hält deshalb
einen expliziten Stapel von Ergebnissen:

```php
$this->db->pushResult();   // aktuelles Ergebnis sichern
$this->db->query(…);       // Unterabfrage
$this->db->popResult();    // vorheriges Ergebnis wiederherstellen
```

Der Template-Compiler setzt das bei `CMS:SELECT` automatisch. Wer im PHP-Code während
einer Iteration eine zweite Abfrage stellt, klammert sie selbst.

Helfer: `fetch()`, `fetchAll()` (einspaltige Ergebnisse werden zu einer flachen Liste),
`fetchNum()`, `fetchColumn()`, `fetchPairs()`, `fetchAndIndex()`,
`beginTransaction()`/`commit()`/`rollback()`/`inTransaction()` und `afterCommit()` für
Rückrufe nach dem äußersten Commit.

## Core-Module

| Modul | Aufgabe |
|---|---|
| `Admin` | Benutzer, Benutzergruppen, persönliche Einstellungen, zweiter Faktor, Geräte |
| `Area` | Inhaltsbereiche und Eingabefelder (`Area`, `Field`), Baupläne |
| `Audit` | Audit-Log recherchieren (nur Superuser) |
| `Component` | Komponenten verwalten (Oberfläche zum `ComponentManager`) |
| `Converter` | Konverter = konfigurierte Prozessoren |
| `Docs` | Handbücher im Backend, Volltext-Index |
| `Instance`, `Language` | Websites und Sprachen |
| `Logfiles` | Protokolle unter `var/log` (nur Superuser) |
| `Messaging` | interne Nachrichten zwischen Benutzern |
| `Record` | Datensätze: Maske, Liste, Suche, Import/Export, Stapelverarbeitung; `RecordUser` = Laufzeit der `CMS:USER_*`-Befehle |
| `Scheduler` | Zeitgesteuerte Aufgaben |
| `Snippets` | eigene Textbausteine je Instanz |
| `Start` | Startseite, Anmeldung, Hauptmenü, Hintergrundprozesse |
| `Status` | Datensatz-Status |
| `Template` | Templates, Parser/Compiler, Befehlsdefinitionen |
| `Tools` | Caches, Datenbankpflege, Server-Informationen, Entwicklerwerkzeuge |
| `Trash` | Papierkorb |
| `User` | Backend-Sicht auf die Frontend-Community (Gegenlesen) |

Core-Prozessoren: `ImageGd`, `Magick`, `FFmpegVideo`, `FFmpegAudio`, `FFmpegImage`.
Core-Extensions: `Calendar`, `Datepicker`, `IconPicker`, `Markdown`, `Password`,
`Snippets`.
