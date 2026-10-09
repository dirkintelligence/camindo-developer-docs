# 08 – Komponenten und Extensions

## Begriffe

| Begriff | Bedeutung |
|---|---|
| ***Komponente*** | Oberbegriff für Modul, Prozessor und Extension |
| **Core-Komponente** | liegt unter `Core/Modules`, `Core/Processors`, `Core/Extensions`; wird nur mit dem Core als Ganzes ausgetauscht, ist immer installiert und aktiv |
| **Zusatzkomponente** | liegt unter `Modules/`, `Processors/`, `Extensions/`; einzeln installier-, aktivier-, aktualisier- und deinstallierbar |
| **Modul** | Backend-Funktionsblock mit Scripten, Dataset-Klassen, Snippets, Vorlagen; optional eigene `(CMS:…)`-Befehle |
| **Prozessor** | Medienumwandlung ([07](07-media-processing.md)) |
| ***Extension*** | JavaScript (+ CSS, + optionale Endpunkte) für ein Eingabefeld im Backend |
| **Präfix** | Snippet- und Vorlagen-Präfix: `{modul}`, `proc.{name}`, `ext.{name}` (klein) |

## Der Komponenten-Vertrag

Jede Komponente hat dasselbe Gerüst:

```
{Name}/
├── {Name}.php                 Modul: Dataset-Klasse(n); Prozessor: MediaProcessor; Extension: optional
├── install/
│   ├── settings.php           Steckbrief (Pflicht)
│   ├── Installer.php          optional: {Namespace}\Install\Installer
│   ├── schema.sql             optional: Tabellen – CREATE TABLE IF NOT EXISTS
│   ├── seed.sql               optional: Stammdaten
│   └── migrations/            {version}.sql und {version}.php
├── lang/  backend/lang/       Snippets
├── backend/                   Backend-Scripte, html/, lang/
├── public/                    nur Extension: js/, css/ (wird veröffentlicht)
└── docs/{Sprache}/            optional: Handbuch der Komponente (Markdown)
```

Namespaces: `Camindo\Modules\{Name}`, `Camindo\Processors\{Name}`,
`Camindo\Extensions\{Name}` (Core: `Camindo\Core\…`). Verzeichnisname = Klassenname =
StudlyCase (`^[A-Z][A-Za-z0-9]*$`). Ein Zusatzmodul darf nicht so heißen wie ein
Core-Modul; ein Zusatzprozessor überschreibt dagegen einen gleichnamigen Core-Prozessor.

### `install/settings.php`

```php
<?php
return [
    'version'     => '1.2.0',                  // SemVer – steuert die Migrationen
    'type'        => 'blog',
    'label'       => '#blog.module_label',
    'description' => '#blog.module_description',
    'icon'        => 'article',
    'listed'      => true,                     // Modul: in Rechteverwaltung und Startseite
    'position'    => 50,                       // Sortierung (höher = weiter unten)
    'requires'    => [
        'core'       => '^9.0',                // Constraint gegen CAMINDO_VERSION
        'php'        => ['gd', 'intl'],        // PHP-Erweiterungen
        'bin'        => ['ffmpeg'],            // paths.* in .camindo.php, ausführbar
        'components' => ['Record' => '^9.0'],  // andere Komponenten, müssen aktiv sein
    ],
    'tables'      => ['blog', 'blog_tag'],     // Tabellen der Komponente (für --purge)
];
```

Nur Schlüssel aus einer festen Liste werden übernommen
(`ComponentManager::SETTINGS_KEYS`):

| Schlüssel | Für | Bedeutung |
|---|---|---|
| `version`, `type`, `label`, `description`, `icon`, `position`, `requires`, `tables` | alle | siehe oben |
| `listed` | Modul | unterliegt der Rechteverwaltung, bekommt eine Kachel |
| `access` | Modul | Zusatzrechte `name => Snippet der Beschriftung` ([11](11-permissions.md#zusatzrechte)) |
| `script_access` | Modul | `script.php => zusatzrecht` |
| `shared` | Modul | Scripte, die andere Module ohne Modulzugriff als Lookup aufrufen dürfen |
| `personal` | Modul | Scripte, die jeder Benutzer ohne Modulzugriff aufrufen darf (eigene Einstellungen) |
| `entries` | Modul | Einstiegsscripte der Kachel; der erste erlaubte gewinnt |
| `button` | Modul | `false`: keine Kachel (etwa nur Widget) |
| `superuser` | Modul | `true`: nur für Superuser, unabhängig von den Gruppenrechten |
| `widget` | Modul | Startseiten-Widget ([05](05-backend-development.md#startseiten-widgets)) |
| `api` | Modul | API-Klassen relativ zum Modul-Namespace, z. B. `['BlogApi']` ([16](16-api.md#eigene-api-methoden)) |
| `commands`, `services` | Modul | eigene `(CMS:…)`-Befehle ([06](06-template-compiler.md#eigene-befehle-aus-zusatzmodulen)) |
| `assets` | Modul, Extension | JS- und CSS-Dateien ([Assets](#assets-und-endpunkte)); bei Modulen auf jeder Backend-Seite |
| `display_types`, `field_types`, `options_provider` | Extension | siehe unten |

`settings.php` wird per `include` ausgewertet – eine Komponente ist Code, den der
Betreiber installiert.

### `install/Installer.php`

Wer beim Installieren mehr braucht als `schema.sql`/`seed.sql`, implementiert
`Camindo\Core\Install\ComponentInstallerInterface` in der Klasse
`{Namespace}\Install\Installer` (sie wird per `require_once` geladen, nicht über den
Autoloader):

```php
interface ComponentInstallerInterface {
    public function install(InstallContext $ctx): void;                     // nach schema.sql/seed.sql
    public function migrate(string $from, string $to, InstallContext $ctx): void;
    public function activate(InstallContext $ctx): void;
    public function deactivate(InstallContext $ctx): void;
    public function uninstall(InstallContext $ctx, bool $purge): void;     // purge: Daten löschen
}
```

`InstallContext` reicht `db`, `config`, `system`, Name und Pfad der Komponente durch und
bietet `tableExists()`, `columnExists()`, `indexExists()`, `runSqlFile()` und `log()`
(Konsole bzw. Backend).

## Lebenszyklus

```
composer require      component install      component activate
 ──────────▶ entdeckt ───────────▶ inaktiv ◀────────────────▶ aktiv
                                      ▲                         │ composer update
                                      │ component update        ▼
                                      └─────────────────── Update wartet
```

| Zustand | Erkennbar an | Bedeutung |
|---|---|---|
| **entdeckt** | Verzeichnis da, keine Zeile in `component` (oder `version = ''`) | nichts installiert, wird nicht geladen |
| **inaktiv** | `status = 0` | installiert, aber nicht geladen |
| **aktiv** | `status = 1` | Normalbetrieb |
| **Update wartet** | Dateiversion > installierte Version | gilt zur Laufzeit als inaktiv, bis `component update` lief |
| **verwaist** | Zeile, aber Verzeichnis fehlt | nach `composer remove` ohne `uninstall`; aufräumbar |
| **fehlerhaft** | `requires` nicht erfüllt, Namenskonflikt | nicht installier- oder aktivierbar, Grund in der Liste |

`Camindo\Core\ComponentManager` bündelt alles ohne Backend-Abhängigkeit – die
Kommandozeile und das Modul Component rufen dieselben Methoden:

| Methode | Tut |
|---|---|
| `discover()` / `status()` | Verzeichnisse lesen, `requires` prüfen, mit `component` abgleichen – liest nur |
| `refresh()` | neue Komponenten eintragen, Dateiversionen nachziehen, Laufzeitlisten schreiben |
| `install(name)` | Sicherung, `schema.sql`, `seed.sql`, `Installer::install()`, Zeile anlegen, Caches der Komponente, Assets |
| `activate` / `deactivate` | Status setzen, `Installer::(de)activate()`, Laufzeitlisten, Gruppen-Cache |
| `update(name)` | Sicherung, Migrationen von der installierten zur Dateiversion, `Installer::migrate()`, Caches |
| `uninstall(name, purge)` | deaktivieren, `Installer::uninstall()`, mit `purge` Tabellen aus `tables` löschen, Zeile, Caches und Assets entfernen |

Abläufe und Befehle: [09](09-installation-and-update.md#zusatzkomponenten).

### Laufzeitlisten

`var/config/components/{modules,processors,extensions}.php` enthalten **nur die zur
Laufzeit aktiven** Komponenten (Core + aktiv) mit `path`, `namespace`, `version` und den
übernommenen Schlüsseln der `settings.php`. Was fehlt, gibt es zur Laufzeit nicht. Die
Listen werden je Request per `include` gelesen, nie aus der Datenbank.
`Cms::componentPath($art, $name)` löst darüber auf; `Backend::backendPath()`,
`ProcessorFactory`, `Access` und die Startseite nutzen sie. Nach jedem Schreiben der
Listen baut `ComponentManager` den Suchindex der Handbücher neu auf – so kommen und gehen
die Handbücher der Komponenten mit ihnen.

### Handbuch einer Komponente

Jede Komponente darf ein eigenes Handbuch mitbringen. Das Modul Docs nimmt es in die
Handbücher des Backends auf (Auswahl, Suche, Stichwortverzeichnis, Druckansicht,
Zurück/Weiter), sobald die Komponente aktiv ist:

```
{Name}/docs/{Sprache}/01-{art}-{name}.md     Startseite (Pflicht), z. B. Modules/Blog/docs/de_DE/01-module-blog.md
{Name}/docs/{Sprache}/*.md                   weitere Kapitel, auch in Unterverzeichnissen
{Name}/docs/{Sprache}/_/*.svg                Icons (Material Symbols), wie in Core/docs
```

| Regel | |
|---|---|
| `{art}` | `module`, `extension` oder `processor` |
| `{name}` | Komponentenname in Kleinbuchstaben (`TinyMce` → `tinymce`) |
| Sprachen | dieselben Codes wie `Core/docs/` (`de_DE`, `en_US` …); fehlt die Startseite in einer Sprache, gibt es das Handbuch dort nicht |
| Titel | die erste Überschrift der Startseite – in Auswahl und Übersicht |
| Reihenfolge | Startseite, dann ihre Links in Listen und Tabellen, übrige Seiten nach Dateinamen – nummerieren Sie die Kapitel (`02-…`, `03-…`) |
| Links | relativ innerhalb des Handbuchs (`./02-befehle.md#abschnitt`); Links in die Handbücher des Core oder anderer Komponenten bleiben Text – nennen Sie Handbuch und Kapitel im Text |
| Format | wie die Handbücher des Core: Sie-Form, Stichwörter als `***fett+kursiv***` |
| Codebeispiele | Blöcke mit CMS-Befehlen als ```` ```cms ```` – das Modul Docs färbt sie wie den Template-Editor ein; Syntaxbeschreibungen, die kein gültiger Befehl sind (`(CMS:TEXT field [parameter=wert …])`), als ```` ```cmscode ```` – ganz lila, ungeparst |

Intern heißt das Handbuch `{art}-{name}` (z. B. `module-blog`); unter diesem Namen
erscheinen seine Seiten auch in der API (`docs.search`, `docs.read`, Parameter `book`).
Die README der Komponente bleibt kurz: Zweck, Installation, Lizenz und je Sprache ein
Link auf die Startseite in `docs/` (siehe [10](10-publishing-components.md#ein-repository-je-komponente)).

## Extensions

Eine Extension erweitert ein Eingabefeld eines bestimmten Display-Typs im Backend um
JavaScript-Verhalten. Sie wird **einem Feld** zugewiesen (in dessen Definition), nicht
global einem Display-Typ – ein Bereich kann eine `textarea` mit TinyMCE und eine andere
mit Markdown haben. Ein Feld hat höchstens eine Extension.

| Core-Extension | Display-/Feldtyp | Zweck |
|---|---|---|
| `Calendar` | `text` / `datetime`, `timestampl`, `time` | native Datums-/Zeitauswahl |
| `Datepicker` | `text` / `datetime`, `timestampl` | Kalender in der Locale des Benutzers |
| `MarkdownVisual` | `textarea` / `text`, `chars` | Markdown-Editor, bearbeitet wird die formatierte Ansicht |
| `HTMLVisual` | `textarea` / `text`, `chars` | HTML-Editor mit derselben Oberfläche, Quelltext über die Schaltfläche `<>`; Ausgabe mit `(CMS:TEXT feld entitydecode=false lbrconvert=false)` |
| `IconPicker` | `text` | Auswahl eines Material-Symbols |
| `Snippets` | `text` | Snippet-Suche für Beschriftungsfelder |
| `Password` | `password` | Stärkeanzeige und Passwort-Richtlinie |

Zusatz-Extension: `TinyMce` (WYSIWYG-Editor, eigenes Paket, GPL wie TinyMCE).

### Steckbrief

```
Extensions/TinyMce/
├── composer.json                 type camindo-extension, require tinymce/tinymce
├── install/settings.php
├── lang/  backend/lang/          Präfix ext.tinymce
├── backend/
│   ├── form.php                  Optionen in der Feld-Maske
│   ├── info.php                  Optionen in der Feld-Detailansicht
│   └── *.php                     optionale Endpunkte
└── public/
    ├── js/tinymce.ext.js         Registrierung bei cApp.extensions
    └── css/tinymce.ext.css
```

```php
return [
    'version'       => '1.0.0',
    'type'          => 'editor',
    'label'         => '#ext.tinymce.label',
    'description'   => '#ext.tinymce.description',
    'display_types' => ['textarea'],             // wo die Extension wählbar ist
    'field_types'   => ['text', 'chars'],        // optional: zusätzlich nach Feldtyp
    'assets' => [
        'js'   => ['vendor:tinymce/tinymce/tinymce.min.js', 'js/tinymce.ext.js'],
        'css'  => ['css/tinymce.ext.css'],
        'dirs' => ['vendor:tinymce/tinymce/skins'],   // ganze Verzeichnisse, rekursiv
    ],
    'options_provider' => 'TinyMceOptions',      // optional, siehe unten
    'requires' => ['core' => '^9.0'],
];
```

`vendor:`-Pfade zeigen in `vendor/`, alle anderen in `public/` der Extension.

### Zuweisung im Feld

`field.details` bekommt `extension` und `extension_options`:

```json
{ "extension": "TinyMce", "extension_options": { "toolbar": "basic", "height": "400" } }
```

Die Feld-Maske bietet nur aktive Extensions an, die zu Display- und Feldtyp passen, und
bindet den Optionsblock aus `{Extension}/backend/form.php` ein (mit `$backend`, `$cms`,
`$options` und `$name` = `data[details][extension_options]`). `Field::hookOnSave()`
weist eine unpassende oder inaktive Extension ab. Metafelder bekommen eine Extension
über `'extension' => 'Name'` in der Spaltendefinition.

### Rendering und JavaScript-API

`Backend::formField()` versieht das Eingabeelement in der Eingabemaske (nicht in der
Suchmaske) mit Attributen – kein Inline-Script je Feld:

```
<textarea … class="… c9-ext" data-c9-ext="tinymce" data-c9-ext-type="text"
          data-c9-ext-options='{"toolbar":"basic","height":"400"}'></textarea>
```

Ein `options_provider` (Klasse mit `clientOptions(array $options): array`) kann die
Optionen vorher serverseitig anreichern – `Password` liefert so die wirksamen Regeln und
übersetzten Texte.

`backend/index.php` gibt die veröffentlichten Assets aller aktiven Extensions als
Manifest `cConfig.extensions` aus; geladen werden sie erst, wenn ein Feld sie braucht.
Die Extension registriert sich mit drei Hooks:

```js
cApp.extensions.register('tinymce', {
    init(el, options)  { /* Editor auf el erzeugen; bei Änderung cApp.forms.markChanged(el) */ },
    beforeSubmit(el)   { /* Inhalt in el.value zurückschreiben */ },
    destroy(el)        { /* Instanz entfernen – Layer wird ersetzt oder geschlossen */ }
});
```

`cApp.extensions` ruft `init()` nach jedem `cLoadContent`, `beforeSubmit()` vor dem
Absenden eines Formulars und `destroy()` vor dem Ersetzen oder Schließen eines Layers.
`cApp.forms.markChanged(el)` meldet Änderungen an die Änderungsverfolgung – Editoren
feuern `input`/`change` auf der versteckten Textarea nicht. Serverseitig bleibt der
Wert ein normales Text-Feld.

### Assets und Endpunkte

`ComponentManager` **kopiert** die in `assets` genannten Dateien nach
`public/backend/ext/{name}/` (Kopie statt Symlink, damit `vendor/` nicht vom Docroot
aus erreichbar wird) und trägt sie mit `?v={version}` ins Manifest ein.

Auch ein **Modul** darf `assets` nennen (`['js' => […], 'css' => […]]`, relativ zu
`public/` des Moduls). Sie landen in `public/backend/mod/{name}/`, und das Backend
bindet sie auf **jeder** Seite ein – Stylesheets im Kopf, Scripte vor
`backend-init.js`, unabhängig vom Modulrecht des Benutzers (die Rechte prüft der Server).
Ein solches Script initialisiert sich selbst (`$(document).ready(…)`) und darf ohne
Modulrecht nichts kaputt machen. Gedacht ist das für Dinge, die über das Modul hinaus
wirken, etwa ein Tastaturkürzel. `camindo.php refresh` publiziert die Assets aller
laufenden Extensions und Module neu – nach einer Änderung an den Dateien genügt das.

Endpunkte einer Extension liegen in `backend/` und sind über
`/backendExtensions/{Name}/{script}` erreichbar. Sie verlangen nur eine Session –
Extensions haben keine Modulrechte. Ein Endpunkt darf deshalb nichts ausliefern, was
eine Bereichsrechte-Prüfung bräuchte.

### Projekteigene Dateien

Editoren brauchen projektspezifische Stylesheets. Sie liegen in `var/extensions/{name}/`
(`Backend::extensionVarDir()`), nicht in der Extension, die bei Updates ersetzt wird.
`form.php` bietet die Dateien zur Auswahl an (`Backend::extensionVarFiles()`), ein
eigener Endpunkt liefert sie aus, weil `var/` nicht im Docroot liegt.

### Im Frontend

Eine Extension, deren Script ohne `cApp` auskommt, lässt sich auch im Frontend nutzen.
`Password` zeigt das Muster: das Script stattet `input[data-c9-password]` beim Laden
selbst aus; den Attributwert liefert `(CMS:USER_PASSWORD_RULES)`.

### Markdown-Syntax: Editor und Renderer

`MarkdownVisual` bringt keine Bibliothek mit. Das Script (`markdownvisual.ext.js`)
enthält einen eigenen Markdown-Parser, das Rendering in den Editor und einen
Serializer DOM → Markdown. Eingefügtes Markdown und HTML laufen durch dieselbe
Blockliste; was sie nicht abbildet (Bilder, Durchstreichen, Farben), entfällt. Der Text
im Editor ist maskiert, Link-Ziele sind auf `http(s)`, `mailto`, `tel`, `ftp` und
relative Adressen beschränkt.

Zum üblichen Markdown kommen `++unterstrichen++`, Ausrichtungscontainer
`::: left|center|right … :::`, `<br>` in Tabellenzellen und die Schreibrichtung als
Unicode-Isolate: RLI `U+2067` (rechts nach links) bzw. LRI `U+2066` (links nach rechts)
bis PDI `U+2069`, im Editor `bdi[dir]` (Schalter `dir`, auch `&#x2067;` … `&#x2069;` wird
erkannt; ein Zeichen ohne Gegenstück bleibt Text). Hervorhebungen reichen nicht über die
Grenze eines Isolats. Der Renderer reicht die Zeichen unverändert durch – Browser setzen
sie selbst um. Im Frontend rendert
`Camindo\Core\Markdown` (Parsedown im Safe Mode, erweitert um genau diese Syntax, dazu
Entities und die Escapes `\:`, `\&`, `\=`, `\<`) – als Template-Befehl
`(CMS:TEXT feld markdown=true)`. Script und Klasse müssen dieselbe Syntax kennen; wer
eine ändert, ändert beide. Den Selbsttest dafür enthält
`cron/check_template_params.php`.

Die Oberflächentexte liefert der `options_provider` `Camindo\Core\MarkdownVisualOptions`
aus den Snippets `ext.markdownvisual.ui_*`; fehlende Texte ersetzt das Script durch
englische Vorgaben.
