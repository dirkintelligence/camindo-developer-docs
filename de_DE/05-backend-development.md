# 05 – Backend-Entwicklung

Wie eine Backend-Seite entsteht – von der Datei bis zum HTML.

## Aufbau eines Moduls

```
Modules/Blog/                    (Core-Module: Core/Modules/{Name}/)
├── Blog.php                     Dataset-Klasse (Tabelle blog)
├── BlogBackend.php              Backend-Logik: list(), info(), edit(), search()
├── install/
│   ├── settings.php             Steckbrief des Moduls (Kapitel 08)
│   ├── views.php                Ansichten für Werkzeuge wie den KI-Assistenten (unten)
│   ├── schema.sql               Tabellen (CREATE TABLE IF NOT EXISTS)
│   └── migrations/              Versionsschritte
├── lang/{code}.ini              Snippets für Backend UND Frontend  (Präfix 'blog')
└── backend/
    ├── index.php                Einstieg der Modulkachel
    ├── blog_list.php            View-Scripte, je drei Zeilen
    ├── blog_info.php
    ├── blog_edit.php
    ├── blog_search.php
    ├── html/*.html              HTML-Vorlagen                     (Präfix 'blog')
    └── lang/{code}.ini          Snippets nur fürs Backend         (Präfix 'blog')
```

Namespaces: Core-Module `Camindo\Core\Modules\{Name}`, Zusatzmodule
`Camindo\Modules\{Name}`. Verzeichnisname = Klassenname = StudlyCase.

### Das Muster: View-Script → Backend-Klasse → Dataset

Die Scripte in `backend/` sind bewusst **drei Zeilen lang**: Backend-Klasse aus dem
Container holen, Methode aufrufen.

```php
<?php
// Modules/Blog/backend/blog_info.php
// @var Camindo\Core\ContainerInterface $app
use Camindo\Modules\Blog\BlogBackend;

$app->get(BlogBackend::class)->info();
```

| Schicht | Datei | Rolle |
|---|---|---|
| Routing | `backend/blog_edit.php` | Einstiegspunkt, kein Code |
| Controller + Darstellung | `BlogBackend.php` | liest den Request, orchestriert, gibt HTML aus |
| Modell | `Blog.php` (`extends Dataset`) | Schema, Validierung, Persistenz, Hooks |
| Vorlagen | `backend/html/*.html` | HTML mit `{platzhaltern}` |
| Texte | `lang/`, `backend/lang/` | Snippets |

Große Module teilen die Backend-Klasse auf – `Record` etwa in `RecordBackendEdit`,
`RecordBackendList`, `RecordBackendInfo`, `RecordBackendSearch` …

Abläufe, die mehrere Oberflächen brauchen, gehören nicht in die Backend-Klasse, sondern
in eine **Service-Klasse** ohne Request und ohne Ausgabe (Rückgabe `null`/`false`,
Meldungen in `Cms::$errors`). Beispiel `RecordService`:

| Methode | Aufgabe |
|---|---|
| `loadForAction($id, $action)` | produktiven Datensatz laden, Rechte der Benutzergruppe für `read`, `edit`, `delete`, `copy`, `move` |
| `loadForChange($id, $action, $areaId, $unlock)` | dazu Bereich, fremde Sperre und die eigene noch offene Maske |
| `lockInfo()`, `lockMessage()`, `ownOpenCopy()` | Bearbeitungssperren |
| `checkOut($data, $copy, $dependents)` | Arbeitskopie samt eingebetteter Inhalte, der Session zugeordnet |
| `saveFields()`, `checkIn($tmpId, $fields, $data)` | Check-in samt eingebetteter Inhalte in einer Transaktion |
| `updateOne(…)` | Ändern ohne Maske in einem Schritt (Stapelverarbeitung, API) |
| `delete($id, $withChilds)` | Soft-Delete, große Teilbäume im Hintergrund |
| `urlFilter($input)` | Suchfeld `_url` (Id, Slug, URL) |

Die Eingabemaske, das Verschieben, die Stapelverarbeitung, der Import und die API
([16](16-api.md)) nutzen ihn. Ebenso geteilt: `RecordPresenter` – Spalten und Werte der
Datensatzliste (`listFields()`, `listValue()`), dieselben in der Backend-Liste, der
Lookup-Vorschau und `record.search`.
Eine Arbeitskopie des Session-Benutzers sperrt ihn selbst nicht – `checkOut()` verwirft
sie aber. Wer einen Datensatz ohne Maske ändert, prüft deshalb mit `loadForChange()`,
ob derselbe Benutzer ihn noch in einer Maske offen hat.

### Die vier Standardseiten

| Methode | Script | Zweck |
|---|---|---|
| `list()` | `{modul}_list.php` | Trefferliste mit Sortierung und Paginierung |
| `info()` | `{modul}_info.php` | Detailansicht |
| `edit()` | `{modul}_edit.php` | Anlegen, Bearbeiten, Kopieren, Löschen |
| `search()` | `{modul}_search.php` | Suchmaske und Suche |

`edit()` ist ein kleiner Zustandsautomat, gesteuert über `$_REQUEST['function']`:

```php
public function edit(): void
{
    $this->id       = (int)($_REQUEST['id'] ?? 0);
    $this->function = (string)($_REQUEST['function'] ?? '') ?: 'edit';
    $this->data     = (array)($_REQUEST['data'] ?? []);

    $this->backend->setNaviLabel('#button_' . $this->function);
    echo $this->backend->template('container_header');

    if ($this->function === 'create')                           $this->_editCreate();
    if (in_array($this->function, ['edit', 'copy', 'delete']))  $this->_editLoad();
    echo $this->editHeadline();
    if ($this->function === 'save')                             $this->_editSave();
    if ($this->function === 'delete_confirm')                   $this->_editDeleteConfirm();
    if (in_array($this->function, ['create', 'edit', 'copy', 'errors'])) $this->_editForm();
    if ($this->function === 'delete')                           $this->_editDeleteForm();

    echo $this->backend->template('container_footer');
}
```

| `function` | Bedeutung |
|---|---|
| `create` | leeres Formular mit Vorgaben aus `init()` |
| `edit` (Vorgabe) | Datensatz laden und anzeigen |
| `copy` | laden, als neuen Datensatz behandeln |
| `save` | speichern; bei Fehlern weiter mit `errors` |
| `errors` | Formular mit Eingaben und Fehlern erneut anzeigen |
| `delete` / `delete_confirm` | Rückfrage / Löschen ausführen |

Private Hilfsmethoden tragen den Namen der öffentlichen Methode als Präfix:
`edit()` → `_editSave()`, `_editForm()`.

**Schreibende Aktionen nur per POST.** Jeder Endpunkt mit Seiteneffekt prüft
`if (!$this->backend->requirePost()) { echo $access->deniedHtml(); return; }`. Das
CSRF-Token prüft `Backend::backendPath()` für jeden POST automatisch
([14](14-security-web.md#csrf)).

## Rechte prüfen

Den Modulzugriff prüft `Backend::backendPath()` für jedes Script. Darüber hinaus fragt
die Backend-Klasse `Backend::access()`:

```php
$access = $this->backend->access();
if (!$access->areaRead($areaId)) {                 // Leserecht der Benutzergruppe auf den Bereich
    echo $access->deniedHtml();
    echo $this->backend->template('container_footer');
    return;
}
if ($access->areaWrite($areaId, $ownerId)) { … }   // Bearbeiten-Schaltfläche nur mit Recht
if ($access->moduleRight('Blog', 'publish')) { … } // Zusatzrecht des Moduls
```

Die Datenebene prüft unabhängig davon (`Dataset::accessCheck()`), die Oberfläche soll
nur nicht anbieten, was scheitern würde. Kontextmenüs blenden Einträge je Zeile über
`data-ctx-hide="edit,delete"` (Zeile) und `data-ctx="edit"` (Eintrag) aus.
Einzelheiten: [11](11-permissions.md).

## Snippets – mehrsprachige Texte

### Ablage

| Verzeichnis | Präfix | gilt für |
|---|---|---|
| `Core/lang/{code}.ini` | – | Backend und Frontend |
| `Core/backend/lang/{code}.ini` | – | nur Backend |
| `{Modul}/lang/{code}.ini` | `{modul}` (klein) | Backend und Frontend |
| `{Modul}/backend/lang/{code}.ini` | `{modul}` | nur Backend |
| `{Prozessor}/lang/…`, `…/backend/lang/…` | `proc.{name}` | dito |
| `{Extension}/lang/…`, `…/backend/lang/…` | `ext.{name}` | dito |
| `var/translation/translated/*.ini` | im Schlüssel | Übersetzungspakete (s. u.) |
| `var/custom/snippets/{instance_id}/lang/{code}.ini` | `custom.{instance_id}` | Snippets einer Instanz (Modul Snippets) |

`{code}` ist ein Backend-Sprachcode im Format `de_DE`, `en_US` (12 Sprachen im Core).
`en_US.ini` ist Pflicht und dient als Rückfall. In `lang/` gehört nur, was auch im
Frontend erscheinen kann (Fehlermeldungen, Beschriftungen von Metafeldern); ein
Frontend-Request lädt nie eine `backend/lang`-Datei. Andere Dateien im
`lang`-Verzeichnis werden übergangen.

```ini
; Modules/Blog/backend/lang/en_US.ini
field_blog_title   = "Title"
message_blog_saved = "The entry has been saved."
alert_blog_delete  = "Really delete the entry?"
count_entries      = "{n, plural, one{# entry} other{# entries}}"
```

Das Format liest und schreibt `SnippetIni` – bewusst nicht `parse_ini_file()`, damit
eine Datei von außen nichts auslöst (keine Umgebungsvariablen, keine Konstanten,
`yes`/`no` als Schlüssel erlaubt). Werte stehen in doppelten Anführungszeichen und dürfen
über mehrere Zeilen gehen; maskiert werden nur `\"` und `\\`, `\n` bleibt Text.
Kommentare beginnen mit `;` oder `#`.

**Übersetzungspakete.** Eine weitere Backend-Sprache braucht keine Änderung an den
Komponenten: Eine Datei `var/translation/translated/{code}.ini` (oder
`{code}_{beliebig}.ini`) enthält die Texte beliebig vieler Komponenten, Schlüssel mit
Präfix (`ext.datepicker.option_mode`, ohne Punkt = Core), getrennt in die Abschnitte
`[frontend]` (wie `lang/`) und `[backend]`. Jede Sprache mit einer solchen Datei wird
Backend-Sprache; ihr Name in der eigenen Sprache steht in `en_US_{code}.ini`
(`backend_language_{code} = "…"`). Pakete überschreiben die Dateien der Komponenten, die
Snippet-Sets der Instanzen überschreiben die Pakete. Jeder vollständige Bau schreibt
Vorlagen nach `var/translation/blueprint/` (`en_US_core.ini`: alle englischen Texte des
Cores mit Abschnitten und Präfix).

ICU-Formate (`plural`, `select`) werden nur ausgewertet, wenn das **ganze** Snippet damit
beginnt.

### Verwendung

```php
$cms->snippet('blog.field_blog_title');                 // Modul Blog
$cms->snippet('proc.ffmpegvideo.headline_output');      // Prozessor
$cms->snippet('button_back');                           // Core
$cms->snippet('blog.count_entries', ['n' => 3]);        // mit Parametern
```

Aufgelöst wird in der aktuellen Sprache; fehlt ein Schlüssel, wurde beim Bauen des
Caches schon der englische Text eingesetzt; fehlt er auch dort, erscheint der Schlüssel
selbst – beim Entwickeln fällt so jede Lücke auf. ICU-Snippets werden beim Bauen zu
Closures kompiliert.

**Beschriftungen mit `#`.** In Schema-Definitionen steht `'label' =>
'#blog.field_blog_title'`. `Cms::textOrSnippet()` löst ein führendes `#` als Snippet
auf, alles andere bleibt Klartext. Eingabefelder für Beschriftungen tragen im Schema
`'extension' => 'Snippets'` – eine Suche über alle Snippets. Für übliche Feld- und
Bereichsnamen liefert der Core `label_*`-Snippets mit (`Core/backend/lang`, etwa
`#label_customers`, `#label_images`, `#label_url`) – Redakteure brauchen dafür kein
eigenes Set.

**Snippets je Instanz.** Das Modul Snippets pflegt ein Set je Instanz
(`custom.{instance_id}`), als Dateien im selben Format, geschrieben ausschließlich über
`SnippetIni::quote()` – eingegebener Text bleibt immer ein Wert. Im Frontend
genügt die Kurzform `custom.{schlüssel}` – die Instanz ergänzt `Cms::snippet()` selbst;
im Backend ist die ausdrückliche Form `custom.31.welcome` nötig. Das Frontend wählt die
Snippet-Sprache über `language.backend_code`.

## HTML-Vorlagen

| Verzeichnis | Präfix |
|---|---|
| `Core/backend/html/*.html` | – |
| `{Modul}/backend/html/*.html` | `{modul}` |
| `{Prozessor}/backend/html/*.html` | `proc.{name}` |
| `{Extension}/backend/html/*.html` | `ext.{name}` |

```
<!-- Modules/Blog/backend/html/blog_table_row.html -->
<tr class="c9-ctx-row" data-id="{data_id}" role="button" tabindex="0"
    aria-label="{#blog.speak_info_blog}">
  <td class="text-primary fw-bold">{data_title}</td>
  <td>{data_date}</td>
</tr>
```

```php
echo $backend->template('blog.blog_table_row', [
    '{data_id}'    => $id,
    '{data_title}' => htmlspecialchars($title),
]);
```

* Name = `{präfix}.{datei}` ohne `.html`; ohne Punkt die präfixlose Core-Vorlage.
* `{einfach}` ersetzt `strtr()` aus dem Array; `{#name}` wird automatisch durch das
  gleichnamige Snippet ersetzt.
* Werte für Platzhalter, die in einem Attributwert `"…"` stehen, bekommen `"` als
  `&quot;` (`attributePlaceholders()`). Text-Inhalte maskiert der Aufrufer selbst.
* `onclick="…"`, andere `on…`-Attribute, `href="javascript:…"` und `<script>` in einer
  Vorlage werden für die Script-Policy markiert und funktionieren wie gewohnt
  ([14](14-security-web.md#script-policy-im-backend)).

## Caches – nach jeder Änderung neu bauen

Snippets und HTML-Vorlagen kommen ausschließlich aus Caches
(`var/cache/lang/{locale}/{präfix}[.backend].php`, `var/cache/html/cache[.{präfix}].php`).
Ohne Neubau erscheint weiter der alte Stand:

```
php Core/cron/camindo.php cache all                     Snippets, HTML, Templates, Handbücher
php Core/cron/camindo.php cache snippets --prefix=blog  nur eine Komponente
php Core/cron/camindo.php cache html --prefix=blog
```

Im Backend: Werkzeuge → Entwickler. Die Builder sind `SnippetCache` und `HtmlCache`.
Nach Änderungen an `install/settings.php`: `php Core/cron/camindo.php refresh` bzw.
„Refresh" im Modul Component.

## Die Klasse `Backend`

`Camindo\Core\Backend` liefert alle UI-Bausteine. Sie ist zustandsbehaftet: Formular,
Reiter, Buttons und Info-Blöcke werden schrittweise aufgebaut und am Ende abgerufen.

### Seitengerüst, Reiter, Buttons

```php
$backend->setNaviLabel('#button_edit');           // Beschriftung im Pfad
echo $backend->template('container_header');
echo $backend->headline('#blog.headline');

$backend->tabBarReset();
$backend->tabAdd('#tab_data', "cApp.layers.replace('Blog/blog_edit.php?id=$id')", true);
echo $backend->tabBar();

$backend->buttonBarReset();
$backend->buttonAdd('button_save', 'cApp.forms.submit()', '#button_save');
$backend->buttonAdd('button_back', 'cApp.layers.close()', '', alignRight: true);
echo $backend->buttonBar();

echo $backend->template('container_footer');
```

### Formulare

```php
$backend->formInit('blog.blog_edit_form');         // Vorlage mit {data_*}-Platzhaltern
$backend->formHidden('function', 'save');
$backend->formField($blog, [
    'name'  => 'title',
    'value' => $data['title'],
    'id'    => $id,
    // optional: colDef, varName, display_type, options, template, placeholder,
    //           append, selectPlease, lookupOptions, attr, fullWidth, langId, langSuffix,
    //           textLang, dir
]);
$backend->formStatic($blog, 'created_at', $html);
echo $backend->form('edit_form_buttons', [], removePlaceholders: true);
```

`formField()` ersetzt den Platzhalter `{data_{name}}`; mit `'varName' => 'filter'`
entsteht `{filter_{name}}` – so werden Suchmasken gebaut. Mehrsprachige Felder bekommen
`'langId'` und `'langSuffix'`: der Feldname wird `data[name][{language_id}]`.

***Schreibrichtung:*** `'textLang'` (Sprachcode bzw. Locale des Inhalts, Vorgabe bei
`langId`: `langSuffix`) setzt `dir` nach `Cms::isRtl()` – ein `ar_SA`-Feld steht auch im
deutschen Backend rechts nach links. Das gilt für Texte (`text`, `chars`, `encrypted` als
`text`/`textarea`, auch mit Editor-Extension); E-Mail, Telefon, Passwort und Felder mit
Flag `name` stehen immer `ltr`, alles andere erbt die Richtung des Backends (`<html dir>`).
`'dir' => 'ltr'|'rtl'|'auto'` erzwingt eine Richtung. Dieselbe Regel liefert
`Backend::fieldDirection($colDef, $lang)` für eigene Ausgaben (Info-Ansicht, nur lesbare
Sprachen). Editor-Extensions übernehmen die Richtung des Feldes
(`getComputedStyle(el).direction`).

`formField()` prüft streng gegen zwei Listen in `Backend`: `$displayTypes[$typ]`
(welcher `display_type` zu welchem Feldtyp passt) und `$validationFlags[$typ]`. Passt
etwas nicht, erscheint statt des Felds eine Fehlermeldung – Schema-Fehler fallen beim
Entwickeln sofort auf.

| Typ | erlaubte `display_type` |
|---|---|
| `chars`, `encrypted` | `text`, `textarea`, `email`, `tel`, `select`, `radio` |
| `text` | `text`, `textarea`, `select`, `radio` |
| `json`, `reltable` | `checkbox`, `multiselect`, `sortable` |
| `bit` | `switch`, `checkbox`, `radio` |
| `integer`, `currency`, `float`, `decimal`, `datetime`, `time` | `text`, `select`, `radio` |
| `timestampl`, `fulltext` | `text` |
| `id` | `select`, `radio`, `lookup` |
| `relation`, `pointer` | `lookup`, `select`, `multiselect` |
| `relation1`, `pointer1` | `lookup`, `select` |
| `link` | `lookup`, `embed` |
| `file` | `upload`, `thumbnail` |
| `password` | `password` |

`sortable` ist die sortierbare Form von `checkbox` (Pfeil-Buttons,
`backend-smartrows.js`).

**Leere Felder.** `serializeFiles()` lässt leere Werte weg, und ein ausgeschalteter
Switch steht gar nicht im POST. Die Datensatz-Maske führt deshalb das versteckte Feld
`shown` mit allen gerenderten Feldern; was dort steht, aber in `data[]` fehlt, gilt als
geleert. Eigene Formulare, die „leer" vom „nicht angezeigt" unterscheiden müssen,
brauchen denselben Mechanismus. Aus demselben Grund überträgt ein
`<button type="submit" name="x" value="y">` nichts – Aktionen gehören in ein
`<input type="hidden">`, das ein Handler setzt.

### Info-Ansichten, Listen, Verweise

```php
$backend->infoReset();
$backend->infoAdd($blog, 'title', $data['title']);
$backend->infoStatic('Beschriftung', $html);
echo $backend->infoTable();

echo $backend->paginationTop('#blog.headline_list');
echo $backend->tableHeadColumn($colDef, $f, 'title');   // sortierbare Spaltenüberschrift
echo $backend->paginationBottom();

echo $backend->teaser('admin', $adminId, linked: true);  // kompakter Verweis mit Icon
```

### Weitere Helfer

```php
$backend->balloon('success', $cms->snippet('blog.message_blog_saved'));   // Meldung im Browser
$backend->script("$('#x').focus();");               // Script in der Ajax-Antwort (mit Nonce)
echo '<a href="#"' . $backend->on('js', "cApp.layers.add('Blog/blog_info.php?id=$id')") . '>…</a>';
echo '<script' . $backend->nonceAttr() . '>…</script>';
$backend->jsString($wert);                          // sicheres JS-String-Literal
$backend->sanitizePath($pfad);
$backend->backendValue($value, $colDef, 'short');   // Wert für die Anzeige aufbereiten
$backend->isSuperuser();
```

## JavaScript-API (`window.cApp`)

| Namespace | Aufgaben |
|---|---|
| `cApp.system` | Spinner, Meldungen, CSRF-Header |
| `cApp.forms` | Änderungsverfolgung (`markChanged(el)`), Ajax-Submit, `updateElement()` |
| `cApp.ui` | Lookups, Reiter, QR-Codes, Kopieren |
| `cApp.layers` | Layer-Stapel und Hash-Routing ([01](01-architecture.md#layer-und-hash-routing-im-backend)) |
| `cApp.aside` | Seitenleiste (`open`, `openPost`, `close`) |
| `cApp.ctxMenu` | Kontextmenüs (`.c9-ctx-row`; Einträge mit `data-method="post"` per POST) |
| `cApp.polling` | Hintergrundprozesse |
| `cApp.extensions` | Extensions der Eingabefelder ([08](08-components-and-extensions.md#extensions)) |
| `cApp.handlers` | Binden markierter Event-Handler |

Alle Ajax-Aufrufe laufen über `$(el).cLoadContent(options)`:

```js
$('#aside-content').cLoadContent({
    url: 'Blog/blog_edit.php?id=42',
    method: 'POST',
    params: $('#form').serializeFiles(),
    callback: () => { /* … */ }
});
```

`serializeFiles()` baut ein `FormData`, lässt leere Textfelder und ungenutzte
Datei-Felder weg und prüft die Gesamtgröße gegen `cConfig.max_upload_bytes`.

**Regeln der Script-Policy** (ausführlich in [14](14-security-web.md#script-policy-im-backend)):

* Handler (`onclick`) nur in HTML-Vorlagen oder über `$backend->on()`, nie roh im `echo`.
* Eigene `<script>`-Blöcke nur mit `$backend->nonceAttr()` bzw. `$backend->script()`.
* Kein `eval`, `new Function`, `setTimeout('string')`; kein HTML mit Handlern per
  `innerHTML` – Listener gehören an Elemente (`$(document).on('click', '.klasse', …)`).

## Startseiten-Widgets

Die Startseite ist ein Raster aus vier Plätzen; Platz 1 hält die Modulkacheln, die
Plätze 2–4 füllt jeder Benutzer in seinen Einstellungen (`admin.settings.widgets`, eine
Liste aus `module` und `position`). Ein Modul bietet ein Widget in seiner
`install/settings.php` an:

```php
'widget' => [
    'class'  => 'BlogBackend',        // relativ zum Modul-Namespace
    'method' => 'widget',             // Vorgabe 'widget'
    'label'  => '#blog.widget_label', // Vorgabe '#{modul}.module_label'
    'icon'   => 'article',            // Material-Symbol, optional
],
```

Der Klassenname ist bewusst relativ – eine `settings.php` kann nur Klassen des eigenen
Moduls instanziieren. Die Methode hat keine Parameter und **gibt das HTML zurück**; die
Karte drumherum liefert `Widgets`. Eine Ausnahme landet in
`var/log/{Datum}_widget-errors.log`, die Startseite läuft weiter. Ein Widget sieht nur,
wer auch das Modul darf.

## Baupläne für neue Bereiche

`Core/Modules/Area/blueprints/{record|profile|ugc}/*.xml` und die eigenen Baupläne einer
Installation in `var/area_blueprints/*.xml` erscheinen beim Anlegen eines Bereichs als
Vorlagen (`AreaService::blueprints()`). Ein Bauplan ist eine XML-Exportdatei mit genau
einem `<area>` samt `<fields>` und ohne Datensätze; `RecordImport::importBlueprint()`
legt die Felder an. Das Kopieren eines Bereichs nutzt denselben Weg.

- Der Dateiname ist der Name des Bauplans; der Typ kommt aus dem `<area>` der Datei
  (in `var/area_blueprints/` gibt es keine Unterordner je Typ). Ein eigener Bauplan
  ersetzt einen mitgelieferten gleichen Namens und Typs.
- `<label>` und `<note>` des Bereichs sind Überschrift und Erklärung in der Auswahl –
  Text oder Snippet-Verweis (`#key`, `#modul.key`). Die mitgelieferten Baupläne
  verwenden die allgemeinen Beschriftungen des Cores (`#label_area_articles`,
  `#label_title` …).

## Import und Export

Das Modul Record exportiert Datensätze als **XML** (Sicherungsformat: Bereiche, Felder,
Datensätze in allen Sprachen, Verknüpfungen, Dateien, Templates, Konverter), **CSV**
(eine Zeile je Datensatz, Unterfelder in Punkt-Schreibweise) und **DOCX** (für
Übersetzungen) und importiert dieselben Formate. Größere Mengen laufen als
Hintergrundprozess (`Core/cron/record_export.php`, `record_import.php`); das Ergebnis
kommt als interne Nachricht.

Für **Template-Sätze**, die in eine fremde Datenbank übertragen werden sollen, markiert
der Export (Option „Templates für Import in Fremddatenbank optimieren") Verweise in den
Parametern der `(CMS:…)`-Befehle:

```
(CMS:INCLUDE <!-- CMS:REPLACE template_name=header -->)
(CMS:SELECT filter[area]=<!-- CMS:REPLACE area_name=artikel --> …)
```

Welche Parameter Verweise tragen, sagt `commands.php` je Parameter mit `'ref'`
(`template`, `area`, `record`). Der Import ersetzt die Markierungen durch die Namen und
Ids der Zieldatenbank. Daraus folgt eine Regel für portable Templates: **Verweise
gehören als Literal in die Parameter der Befehle** (`bereich.template`,
`filter[area]=name`) – Namen im PHP-Code eines Templates schreibt der Import nicht um.

Custom-Snippets reisen mit: `<snippets>` enthält die Snippets, auf die Beschriftungen
verweisen (`#custom.{id}.{key}`), und – mit Inhalten – die, die die exportierten
Templates ansprechen (`Snippets::templateRefs()`: `custom.{key}`, auch zusammengesetzt
wie `"custom.status_{$x}"` → alle Schlüssel mit diesem Anfang). Der Import ergänzt sie
im Set der Zielinstanz (`Snippets::merge()`, Vorhandenes bleibt); der Template-Code
bleibt unverändert, weil die Kurzform das Set der laufenden Instanz meint.

Die Stapelverarbeitung der Datensatzliste (`RecordBatch`) bearbeitet, verschiebt oder
löscht viele Datensätze – je Datensatz über denselben Weg wie die Maske
(`RecordService::updateOne()`: Check-out, `save()`, Check-in), optional als Simulation.

## Neues Modul – Checkliste

1. Verzeichnis `Modules/{Name}/` (Zusatzmodul) mit `install/settings.php`:
   ```php
   <?php
   return [
       'version'     => '1.0.0',
       'type'        => 'blog',
       'label'       => '#blog.module_label',
       'description' => '#blog.module_description',
       'icon'        => 'article',
       'listed'      => true,
       'requires'    => ['core' => '^9.0'],
       // optional: position, access, shared, personal, script_access, entries,
       //           button, superuser, widget, api, assets, commands, services, tables
       //           (Kapitel 08)
   ];
   ```
2. Tabelle in `install/schema.sql` (`CREATE TABLE IF NOT EXISTS`, Spalte `{tabelle}_id`).
3. Dataset-Klasse mit `hookDefinition()` ([03](03-dataset-api.md)).
4. `{Name}Backend.php` mit `list()`, `info()`, `edit()`, `search()`.
5. View-Scripte in `backend/`, darunter `index.php`.
6. Snippets `lang/en_US.ini` (Pflicht) und weitere Sprachen; `module_label`,
   `module_description`.
7. HTML-Vorlagen in `backend/html/` – Handler nur dort oder per `$backend->on()`.
   Dazu `install/views.php`: die Ansichten, die ein Werkzeug öffnen darf (unten).
8. `php Core/cron/camindo.php component install {Name} --activate` – legt die Tabellen
   an, baut die Caches der Komponente und schreibt die Laufzeitlisten.
9. Den Benutzergruppen das Modul (und ggf. Zusatzrechte) zuweisen.

## Ansichten-Katalog (`install/views.php`)

Welche View-Scripte ein Werkzeug dem Benutzer als Layer öffnen darf – etwa der Assistent
des Zusatzmoduls Ai ([16](16-api.md#assistent-zusatzmodul-camindomodule-ai)) –,
beschreibt jede Komponente in `install/views.php`. `Camindo\Core\BackendViews` liest
die Kataloge der aktiven Module, zeigt nur, was der Benutzer öffnen darf
(`Access::script()`: Modulzugriff, `script_access`, persönliche Scripte), und baut die
Adresse nur aus den erlaubten Parametern.

```php
<?php
return [
    'blog_list.php' => [
        'description' => 'List of all blog posts with status; starting point to create a post.',
    ],
    'blog_edit.php' => [
        'description' => 'Edit form of a post. function=create: new post; copy / delete with id.',
        'params'      => ['id' => 'int', 'function' => ['create', 'copy', 'delete']],
    ],
];
```

* Nur Scripte, die beim Öffnen nichts ändern: Listen, Detailansichten, Formulare, die
  erst beim Absenden wirken. Keine Aktionen (`*_run.php`, Cache-Neubau …), keine
  ajax-Scripte.
* Die Beschreibung ist englisch und für ein Sprachmodell geschrieben: was die Ansicht
  zeigt, wofür man sie braucht, was die Parameter bewirken.
* Parametertypen: `'int'` (positive Ganzzahl), `'string'` (bis 200 Zeichen) oder eine
  Liste erlaubter Werte; andere Parameter verwirft `BackendViews::url()`.
* Kein `refresh` nötig – die Datei wird bei Bedarf gelesen.

## Ohne Browser testen

Backend-Scripte lassen sich auf der Kommandozeile ausführen – ohne Anmeldung, mit den
Rechten eines Benutzers:

```php
<?php
define('CAMINDO_STARTED', true);
define('CAMINDO_BACKEND', true);
require '/pfad/zum/projekt/load_camindo.php';

$app->get(\Camindo\Core\Session::class)->setCliAdmin($adminId);   // withAccess: true = Gruppenrechte laden
$_REQUEST = $_POST = ['id' => 5, 'function' => 'save', 'data' => [/* … */]];
require '/pfad/zum/projekt/Modules/Blog/backend/blog_edit.php';
```

`setCliAdmin()` funktioniert nur auf der Kommandozeile. Das Script wird direkt
eingebunden, nicht über `Backend::backendPath()` (das verlangt eine Session). Für
HTTP-Tests legt `Session::create($adminId)` eine echte Session an (Cookie `sessid`, das
CSRF-Token liefert `Session::csrfToken()`); hinterher die Session wieder entfernen.
Wer so eine Datensatz-Maske rendert, erzeugt eine Arbeitskopie – mit `Record::cancel()`
verwerfen.
