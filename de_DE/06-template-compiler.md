# 06 – Template-Compiler

Frontend-Templates werden in der Befehlssprache `(CMS:…)` geschrieben, beim Speichern
**zu PHP kompiliert** und als Datei abgelegt. Zur Laufzeit läuft reines PHP – kein
Parser, keine Template-Engine im Request. Die Bedeutung der einzelnen Befehle für
Template-Programmierer beschreibt das Handbuch für Template-Programmierer; dieses
Kapitel beschreibt, was dahinter liegt und wie man eigene Befehle ergänzt.

## Die Pipeline

```
Quelltext  var/templates/{instance_id}/{bereich}/{name}.cms.php      ← führt
   │
   ├── Backend: Maske lädt die Datei, Speichern schreibt sie zurück
   └── Datei von Hand geändert/angelegt/gelöscht: TemplateFiles::sync() (minütlich per Cron)
          │
          └── Template::save()
                 ├── hookOnSave()  → TemplateParserTrait::parse($code, $areaId)
                 │        ├── phpCode    → nur als Datei, NIE in die Datenbank
                 │        ├── styledCode → template.styled_code  (Anzeige)
                 │        ├── errors     → template.error_text   (Fehler und Hinweise)
                 │        └── code       → template.code         (Kopie, nur für die Suche)
                 └── hookOnSaved() → var/php/{area_id}/{name}.php

Frontend-Request → Page → include var/php/{area_id}/{name}.php   (in einer IIFE)
```

| Ort | Inhalt | Rolle |
|---|---|---|
| `var/templates/{instance_id}/{bereich}/{name}.cms.php` | Quelltext | **führt** – das, was bearbeitet wird |
| `var/php/{area_id}/{name}.php` | kompiliertes PHP | wird ausgeführt |
| `template.code` / `styled_code` / `error_text` | Kopie, eingefärbte Fassung, Meldungen | Suche und Anzeige im Backend |

**Aus der Datenbank gelangt nie Code ins Dateisystem.** Kompiliert wird nur aus den
Dateien; fehlt eine Datei, lädt das Template mit leerem Quelltext. Wer in die Datenbank
schreiben kann, kann damit keinen PHP-Code ausführen.

### Dateien und Datenbank abgleichen

`TemplateFiles::sync()` gleicht das Dateisystem mit der Datenbank ab: neues Verzeichnis
unter einem Mandanten → neuer Template-Bereich; neue Datei → neues Template; geänderte
Datei → neu speichern und kompilieren; gelöschte Datei → Template löschen; gelöschtes
Verzeichnis → Bereich in den Papierkorb. Namen müssen `^[a-z][a-z0-9_]+$` genügen.
Fehlt ein ganzes Mandanten-Verzeichnis, löscht `sync()` dort nichts. Umbenennen im
Dateisystem ist Löschen + Anlegen – Zuweisungen gehen verloren; umbenennen also besser
im Backend. Umgekehrt legt das Backend Dateien und Verzeichnisse an, benennt sie um und
verschiebt sie.

`php Core/cron/camindo.php cache templates` kompiliert alle Dateien neu
(`Template::rebuildFiles()`) und gleicht danach ab.

Syntaxfehler blockieren das Speichern **nicht** – sie stehen in `error_text` und werden
im Backend mit Zeilennummer angezeigt.

### Templates ansprechen

`template.name` ist nur **innerhalb seines Template-Bereichs** eindeutig. Überall, wo ein
Template genannt wird (`CMS:INCLUDE`, `resulttemplate`, `successtemplate` …, Template-Teil
der URL), gilt:

| Schreibweise | Bedeutung |
|---|---|
| `mein_template` | das Template dieses Namens – es darf im Zugriffsbereich der Instanz nur eines geben |
| `bereich.mein_template` | das Template im Template-Bereich `bereich` |

`Cms::templateFile()` löst auf (einmal je Referenz und Request) und bricht bei einem
mehrdeutigen oder unbekannten Namen mit einer `RuntimeException` ab.

## Syntax

```
(CMS:BEFEHL wert key=value key[sub]=value key[]=value …)
```

* Befehlsname ohne Rücksicht auf Groß-/Kleinschreibung.
* Parameter durch Leerraum getrennt, Zeilenumbrüche erlaubt.
* Werte in `"…"` oder `'…'` werden übernommen, inklusive PHP-Interpolation (`$var`,
  `{$ausdruck}`); alles andere wird in doppelte Anführungszeichen gesetzt.
* Unterschlüssel in eckigen Klammern; derselbe Schlüsselpfad mehrfach wird zum Array.
* `<?php … ?>`-Blöcke bleiben unverändert. Befehle werden auch in PHP-Kommentaren
  erkannt – `(CMS:…)` gehört dort nicht hinein.

```
(CMS:SELECT filter[area]=artikel resulttemplate=artikel_liste)
```

wird zu

```php
<?php /* --- CMS:SELECT --- */ $_a = [];
$_a["filter"]["area"] = "artikel";
$_a["resulttemplate"] = "artikel_liste";
$this->db->pushResult(); $this->cms->pushRow($cms ?? null);
$_r = $this->cms->cmsSelect($_a);
if (!empty($_r)) {
    $this->cms->includeDepth++;
    require $this->cms->templatePath . "/{$_r}.php";
    $this->cms->includeDepth--;
}
$this->db->popResult(); $cms = $this->cms->popRow(); ?>
```

Der Generator hält die **Zeilen treu**: Zeilennummern im kompilierten PHP entsprechen
denen im Quelltext.

## Befehlsdefinitionen (`Core/Modules/Template/commands.php`)

Jeder Befehl ist ein Array-Eintrag:

```php
'select' => [
    'method'     => 'cms.cmsSelect',            // {objekt}.{methode}; ohne Punkt: cms
    'php_prefix' => 'push_result',              // Schnipsel vor dem Aufruf
    'php_suffix' => ['include', 'pop_result'],  // … und danach
    'buffer_set' => true,                       // set=… fängt die ganze Ausgabe ab
    'note'       => 'Durchsucht die Datenbank …',
    'do'         => 'resulttemplate',           // Parameter für einen Wert ohne Schlüssel
    'oneof'      => 'filter,relation',          // mindestens einer davon
    'params'     => [
        'resulttemplate' => [
            'mandatory'   => true,
            'subparam'    => 0,                 // 0 = kein Unterschlüssel, 1, 2
            'default'     => '',
            'value'       => 'Templatename',
            'description' => 'Listentemplate …',
            'ref'         => 'template',        // Verweis: template | area | record
            'php_value'   => false,             // true: $var ungequotet typtreu übergeben
        ],
    ],
],
```

Aus `method` wird `$_r = $this->{objekt}->{methode}($_a);`; `$this` ist die
`Page`-Instanz. `'no_action' => true` erzeugt keinen Aufruf – so arbeiten
`CMS:RESULT_START`/`RESULT_END` allein über ihre Schnipsel. `'ref'` dient dem
Export/Import (Verweise umschreiben, [05](05-backend-development.md#import-und-export))
und dem Editor (Links auf das verwiesene Template).

Parameterwerte werden sonst immer zu Strings (`cacheparams=$p` → `"$p"`). Mit
`'php_value' => true` übernimmt der Parser einen ungequoteten Wert, der nur aus einer
Variablen besteht – `$p`, `$cms['id']`, `$set["a"]["b"]`, `$obj->name` oder dasselbe in
`{…}` –, als PHP-Ausdruck (`TemplateParserTrait::phpValueToken()`); Arrays und Zahlen
kommen so unverändert an. Gequotete Werte bleiben Strings. Gesetzt bei
`CMS:INCLUDE cacheparams`.

`'buffer_set' => true` gibt dem Befehl den Parameter `set` für seine ***gesamte***
Ausgabe samt Schnipseln (Listentemplate, eingebundenes Template): Der Parser setzt
`Cms::setBufferStart($_a['set'])` davor und `setBufferEnd()` dahinter – ein Stapel in
`Cms`, weil eingebundene Templates `$_a` überschreiben. Gesetzt bei `CMS:SELECT`,
`CMS:SELECT_LANGVERSION` und `CMS:INCLUDE`.

| Objekt | Klasse | geladen |
|---|---|---|
| `cms` | `Cms` | immer |
| `db` | `Database` | immer |
| `pagination` | `Pagination` | beim ersten Zugriff |
| `recordUser` | `Modules\Record\RecordUser` | beim ersten Zugriff |
| `email` | `Email` | beim ersten Zugriff |

Die Schnipsel liegen in `Core/Modules/Template/php/`:

| Datei | Inhalt |
|---|---|
| `push_result.php` / `pop_result.php` | Datenbankergebnis, Zeilenzähler und `$cms` sichern bzw. zurückholen |
| `include.php` | Ergebnis-Template einbinden, `includeDepth` zählen |
| `include_cached.php` | wie `include.php`, danach `Cms::includeEnd()` – nur `CMS:INCLUDE` ([Ausgabe-Cache](#ausgabe-cache-von-cmsinclude)) |
| `include_and_stop.php` | dito, danach `exit()` (Erfolgstemplates) |
| `loop_start.php` / `loop_end.php` | Ergebnisschleife |

`admin.max_include_depth` (20) begrenzt die Verschachtelung.

### Eigene Befehle aus Zusatzmodulen

Ein Zusatzmodul bringt eigene Befehle mit, indem seine `install/settings.php` zwei
Schlüssel setzt:

```php
'commands' => 'commands.php',                                // relativ zum Modul
'services' => ['pdf' => 'Camindo\\Modules\\Pdf\\CmsPdf'],     // Objekt im Template-Kontext => Klasse
```

* `commands.php` hat denselben Aufbau wie die Core-Datei; die Befehlsnamen tragen
  sinnvollerweise den Modulnamen als Präfix (`'pdf_page'` → `CMS:PDF_PAGE`). Ein Name,
  den der Core schon vergibt, bleibt beim Core.
* `services` erweitert die nachladbaren Dienste von `Page`: `'method' =>
  'pdf.cmsPdfPage'` wird zu `$this->pdf->cmsPdfPage($_a)`, und die Klasse kommt beim
  ersten Zugriff aus dem Container (Autowiring). Die Namen `cms`, `db`, `pagination`,
  `recordUser`, `email` sind reserviert.
* Die Methode bekommt das Parameter-Array `$_a` und gibt entweder Text aus bzw. zurück
  oder – bei Befehlen mit `include` – den Namen des einzubindenden Templates.

Ist das Modul inaktiv, kennt der Parser seine Befehle nicht („Unknown command"), und
ein bereits kompiliertes Template scheitert am Zugriff auf den Dienst. Beispiel:
das Modul `Pdf`.

## Die Befehle im Überblick

| Befehl | Methode | Zweck |
|---|---|---|
| `CMS:SELECT` | `cms.cmsSelect` | Abfrage, bindet `resulttemplate` ein |
| `CMS:RESULT_START` / `RESULT_END` | – | Ergebnisschleife |
| `CMS:TEXT`, `CMS:NUMBER`, `CMS:DATE`, `CMS:DATETIME`, `CMS:TIME`, `CMS:RAW` | `cms.cms…` | Werte formatiert ausgeben |
| `CMS:MEDIA` | `cms.cmsMedia` | Datei bzw. Bildversion |
| `CMS:RELATION` | `cms.cmsRelation` | verknüpfte Datensätze |
| `CMS:URL` | `cms.cmsUrl` | URL eines Datensatzes |
| `CMS:INCLUDE` | `cms.cmsInclude` | anderes Template einbinden, wahlweise mit [Ausgabe-Cache](#ausgabe-cache-von-cmsinclude) |
| `CMS:SELECT_LANGVERSION` | `cms.cmsSelectLanguages` | Sprachen der Instanz als Ergebnismenge |
| `CMS:HEADER` | `cms.cmsHeader` | HTTP-Header setzen oder (ohne Wert) entfernen |
| `CMS:SNIPPET` | `cms.cmsSnippet` | Snippet (Textbaustein) in der Sprache des Aufrufs |
| `CMS:MESSAGE`, `CMS:CONTINUE`, `CMS:SAVEDATA`, `CMS:XML` | `cms.cms…` | Meldungen, Ablauf, Daten, XML |
| `CMS:CSP`, `CMS:NONCE` | `cms.cmsCsp`, `cms.cmsNonce` | Script-Policy der Seite ([14](14-security-web.md#script-policy-im-frontend)) |
| `CMS:FED`, `CMS:FED_SCRIPT`, `CMS:FED_CODE` | `cms.cmsFed…` | Frontend-Editing |
| `CMS:JUMP_LIST`, `JUMP_PREV`, `JUMP_NEXT` | `pagination.cmsJump…` | Blätterleisten |
| `CMS:EMAIL` | `email.cmsEmail` | E-Mail aus Templates versenden |
| `CMS:ZIP` | `zip.cmsZip` | Dateien mehrerer Datensätze als ZIP-Download; `CmsZip` ist ein nachgeladener Dienst (`Page::$lazyServices`) |
| `CMS:USER_*` | `recordUser.cms…` | Frontend-Community ([12](12-community.md)) |

## Laufzeit von `CMS:SELECT`

1. `includeDepth` gegen `admin.max_include_depth` prüfen.
2. `filter` oder `relation` muss gesetzt sein.
3. Parameternamen übersetzen (`filter_op` → `filterop`, `filter_mode` → `filtermode`,
   `hits` → `result`); `relation` ohne `src_id` bekommt den laufenden Datensatz. Bleibt `src_id` kleiner als 1,
   liefert `select()` keine Treffer (statt den Relationsfilter wegzulassen).
4. **Veröffentlichungsbedingung** anhängen (`Cms::publishedSql()`): Status 0,
   `online_at` erreicht, `offline_at` nicht erreicht. `future=true`, `archive=true` und
   `state=N|any` lockern das je Abfrage. Das gilt auch in der Vorschau – dort darf nur
   der aufgerufene Datensatz selbst unveröffentlicht sein.
5. `Cms::select()` ausführen ([04](04-select-and-filter.md)).
6. Treffer → Name des `resulttemplate` zurückgeben; keine Treffer → `noresult` ausgeben
   bzw. `noresulttemplate` zurückgeben; der `include`-Schnipsel bindet ein.

Die Parameter `lang` und `collate` sind definiert, haben aber keine Wirkung.

**Verschachtelung.** Ein `CMS:SELECT` darf in einer Ergebnisschleife stehen, und sein
`resulttemplate` darf wieder eine Schleife enthalten. Die Schnipsel `push_result` /
`pop_result` sichern dafür Datenbankergebnis, den gemeinsamen Zähler `Cms::$row_no` und
`$cms` und stellen sie danach wieder her. Selbsttest:
`php Core/cron/check_nested_select.php`. Ausnahme: `(CMS:USER_BASKET function=load)`
ersetzt die laufende Ergebnismenge ohne Sicherung und gehört deshalb nicht in eine
Ergebnisschleife.

Im Template stehen bereit:

| Variable | Inhalt |
|---|---|
| `$cms` | die aktuelle Ergebniszeile bzw. der Seitendatensatz |
| `$set` | mit `set=` abgelegte Werte |
| `$this->cms`, `$this->db` | `Cms`, `Database` |
| `$this->cms->row_no` | laufende Zeilennummer |
| `$this->isPreview()` | true in der Vorschau aus dem Backend |

## Ausgabe-Cache von `CMS:INCLUDE`

`(CMS:INCLUDE teaser caching=5m cacheparams=$params)` legt die ***Ausgabe*** des
eingebundenen Templates ab und liefert sie innerhalb der Frist aus, ohne das Template
auszuführen. Umgesetzt in `Camindo\Core\IncludeCache`, angebunden in
`Cms::cmsInclude()` und dem Schnipsel `include_cached.php`:

```php
$_r = $this->cms->cmsInclude($_a);   // Treffer: gibt die Datei aus, liefert ''
if (!empty($_r)) { … require …; }    // sonst: Template läuft in einen Ausgabepuffer
$this->cms->includeEnd();            // Aufzeichnung abschließen, ablegen, ausgeben
```

`cmsInclude()` legt je Aufruf einen Eintrag auf einen Stapel (verschachtelte Includes
überschreiben `$_r`/`$_a`); `includeEnd()` nimmt ihn wieder herunter.

* ***Frist*** `caching`: Sekunden, `30s`, `5m`, `1h`, `1d`, `infinite`; leer oder `0`
  schaltet ab, alles andere ist ein Template-Fehler (`IncludeCache::ttl()`).
* ***Schlüssel***: sha256 aus Instanz, Inhaltssprache und `cacheparams` als Text
  (`IncludeCache::paramString()`: Skalare per `(string)`, Arrays als JSON). Der
  Parameter trägt `'php_value' => true`, eine ungequotete Variable kommt deshalb
  typtreu an.
* ***Ablage***: `var/cache/include/{area_id}/{name}/{hash}.html`, erste Zeile die
  Ablaufzeit (0 = unbegrenzt). Gültig ist eine Datei nur, solange sie nicht abgelaufen,
  jünger als die Frist des Aufrufs und jünger als die kompilierte Template-Datei ist –
  Speichern oder `cache templates` machen den Cache eines Templates ungültig.
* ***Nonce***: Die Nonce der Seite wird beim Ablegen durch einen Platzhalter ersetzt und
  beim Ausliefern durch die aktuelle – gecachte Scripts laufen unter der Script-Policy.
* ***Gleichzeitige Aufrufe***: Wer rendert, legt `{hash}.html.lock` exklusiv an
  (`fopen(…, 'x')`); die anderen warten bis `admin.include_cache_lock_wait` (30 s) und
  lesen dann die frische Datei, danach rendern sie ohne abzulegen. Kein `flock()` – es
  sperrt nicht auf jedem Dateisystem prozessübergreifend. Eine Shutdown-Funktion gibt
  Sperren nach `exit`/Fatal Error frei; verwaiste Sperren (älter als das Doppelte der
  Wartezeit, mindestens 120 s) werden übernommen. Geschrieben wird temporär und per
  `rename()`.
* ***Nicht abgelegt*** wird, wenn während des Renderns `Cms::error()` lief. ***Kein
  Cache*** in der Vorschau (`Cms::$includeCacheOff`, gesetzt von `Page::resolve()`), bei
  eingeschaltetem Frontend-Editing und mit `admin.include_cache = false`.
* ***Nur die Ausgabe***: `$set`, Header, Cookies und andere Seiteneffekte des
  eingebundenen Templates fallen bei einem Treffer weg.
* ***Aufräumen***: Zeitplaner-Aufgabe `cachecleanup` (`IncludeCache::cleanup()`:
  abgelaufene Dateien, Dateien zu gelöschten oder geänderten Templates, alte Sperr- und
  temporäre Dateien, leere Verzeichnisse); `camindo.php cache include
  [--template={area_id}/{name}]` leert. Die Startseite weist auf eine fehlende Aufgabe
  hin, sobald der Cache Dateien enthält.

Selbsttest: `php Core/cron/check_include_cache.php`.

## Der Parameter `id`

Ausgabebefehle (`CMS:TEXT`, `NUMBER`, `DATE`, `TIME`, `DATETIME`, `RAW`, `MEDIA`, `URL`,
`SAVEDATA`) lesen normalerweise aus dem laufenden Datensatz. Mit `id` lesen sie aus
einem anderen – aufgelöst wie in der URL (`Cms::_record()`): reine Ziffern = `record_id`,
alles andere ein Pfad `[slug-prefix/]slug`. Jeder Verweis wird je Request einmal geladen
und gemerkt; ein unbekannter Verweis gilt als leerer Wert (`ifempty` greift).

## URL und Slug

`CMS:URL` erzeugt genau das Format, das `Page` wieder zerlegt (`Cms::recordUrl()`):

```
[/{lang}]/{slug|id}[,{template}]
```

* Das Sprachsegment steht immer in der URL, sobald die Instanz mehr als eine Sprache
  führt; `lang=de_DE` zeigt auf eine andere Sprachversion – mit dem Slug **dieser**
  Sprache (`Cms::languageSlug()`).
* Hat der Datensatz ein Feld `slug`, gewinnt es. `slug` ist ein optionales virtuelles
  Feld; `Record::hookOnSave()` füllt es aus dem Default-Feld, wenn es leer ist
  (`Cms::textToUri()`: Tags entfernt, Sonderzeichen zu Bindestrichen, Kleinschreibung,
  Umlaute und `/` bleiben, höchstens `system.link_length` Zeichen).
* **Ein Slug ist nie eine reine Ziffernfolge** – aus „2024" wird `-2024`. `Page`
  unterscheidet nach derselben Regel: reine Ziffern = `record_id`, alles andere = Slug.
* Ein gesetzter Slug wird **nicht** nachgeführt, wenn sich der Titel ändert – geteilte
  Links bleiben gültig.
* **Slug-Prefix:** Ein Bereich kann je Sprache ein Präfix führen
  (`area.details.slug_prefix`), das vor den Slug gesetzt wird: `/de_DE/farben/blau`.
  Eindeutig sein muss der ganze Pfad (`Record::slugPathOk()`); zwei Treffer für einen
  Pfad beantwortet `Page` mit „Ambiguous path".
* **Übersicht (Präfix allein):** Ist der Pfad genau der Slug-Präfix eines Bereichs in
  der Sprache des Aufrufs und hat der Bereich ein Übersichts-Template
  (`area.overview_template_id`, `Cms::overviewAreaIdByPath()`), bindet `Page` es ohne
  Datensatz ein – wie ein vom CMS gewähltes Template, ohne Standalone-Flag und ohne
  kanonische Umleitung. `Cms::$overviewAreaId` hält den Bereich; `CMS:URL` ohne `id`
  verweist dann auf dieselbe Übersicht (Sprachumschalter), `(CMS:URL area=…)` baut die
  Adresse von außen (`Cms::areaUrl()`). Ohne Übersichts-Template: 404.
* **Nur das Sprachkürzel** (`/en`, ohne Schrägstrich): Die Rewrite-Regel für das
  Sprachsegment verlangt den Schrägstrich, der Pfad kommt deshalb als Slug `en` an.
  Ist er ein Sprachkürzel des Auftritts (`InstanceCache::findByCode()`, dieselbe Regel
  wie für `/en/`), leitet `Page` mit 301 auf die Startseite dieser Sprache um – mit dem
  Kürzel aus der Tabelle `language` (`/en_EN/`), bei nur einer Sprache auf `/`. Nur
  GET/HEAD, nicht in der Vorschau. Bei einer Sprache geht ein Datensatz mit genau diesem
  Slug vor; bei mehreren trägt jede Datensatz-Adresse ohnehin ein Sprachsegment.
* **404:** Unbekannter Datensatz (auch einer aus einer fremden Instanz), unbekanntes
  Sprachkürzel, unbekanntes oder nicht freigegebenes URL-Template und eine fehlende
  Template-Datei enden mit Status 404 (`Page::notFound()`) und einer Klartextmeldung.
  Ein nicht freigegebenes Template bekommt bewusst 404 statt 403. Eine gestaltete
  Fehlerseite gibt es noch nicht.

## Template-Zuweisung

Welches Template ein Datensatz bekommt, entscheidet `Page`: Template aus der URL, sonst
`record.template_id`, sonst `area.default_template_id`. Ein Template aus der URL
braucht eine Berechtigung:

| Wo | Bedeutung |
|---|---|
| `rel_area_template.selectable` | in diesem Bereich auswählbar |
| `rel_area_template.override` | darf per URL das zugewiesene Template ersetzen |
| `template.standalone` | darf **ohne** Datensatz aufgerufen werden (`/0,{name}`) |
| `template.type = 'scheduler'` | nie über eine Adresse, nur vom Zeitplaner ([07](07-media-processing.md#zeitplaner-modul-scheduler)) |

Ohne Berechtigung endet der Request mit „Invalid template request".

## Templates im Backend

**Listen-Template** (`area.list_template_id`): Das Backend führt es je Datensatz aus und
zeigt die Ausgabe als letzte Spalte der Datensatzliste und in der ausführlichen
Lookup-Vorschau (`Page::renderRecord()`). Im Template steht der Datensatz als `$cms`
bereit; alle Befehle funktionieren wie im Frontend. Ein Fehler erscheint rot in der
Zelle, die Liste läuft weiter.

**Vorschau** (`backend/preview.php`): `$this->isPreview()` ist true, Anti-Caching- und
`noindex`-Header sind gesetzt.

## Trigger-Templates

Ein Inhaltsbereich kann ein ***Trigger-Template*** haben (`area.trigger_template_id`):
ein Template, das statt Ausgabe **Ereignis-Methoden** enthält. `Record` ruft sie aus
seinen Hooks auf (`TriggerTemplates::run()`):

```php
// Template "person_trigger" im Template-Bereich – nur Methoden, keine Ausgabe
function onSave(array &$affectedFields, array &$data, array &$previousData): void
{
    $data['title'] = trim($data['first_name'] . ' ' . $data['last_name']);
    if (!in_array('title', $affectedFields, true)) $affectedFields[] = 'title';
}
```

| Ereignis | Zeitpunkt |
|---|---|
| `onCreate(array &$data)` | `init()` – Vorbelegung |
| `onLoaded(array &$data)` | nach jedem `Record::load()` (nicht bei `CMS:SELECT`) |
| `onSave(&$affectedFields, &$data, &$previousData)` | vor Prüfung und Schreiben – neu gesetzte Felder in `$affectedFields` eintragen |
| `onSaved(&$affectedFields, &$data, &$previousData)` | nach dem Schreiben, vor dem Commit |
| `onSoftDelete`, `onRestore`, `onDelete`, `onDeleted` (`array $data`) | Papierkorb, Wiederherstellen, endgültiges Löschen vor/nach dem `DELETE` |

* Der kompilierte Code wird in den Rumpf einer anonymen Klasse gesetzt, die von
  `TriggerTemplate` erbt (`var/cache/trigger/{template_id}.php`). Daher: nur Methoden,
  keine Anweisungen außerhalb, keine Ausgabe, Klassen voll qualifiziert (kein `use`).
* Im Template: `$this->cms`, `$this->db`, `$this->record`, `$this->areaId`,
  `$this->temporary` (true beim Zwischenspeichern einer Arbeitskopie) und die
  nachladbaren Dienste von `Page` (`$this->email` …).
* Außer `onCreate`/`onLoaded` läuft alles in der Transaktion des Vorgangs; eine
  Exception (`throw new \Exception('Meldung')`) bricht ihn ab, die Meldung erscheint in
  der Maske.
* Beim Verschieben gilt das Trigger-Template des **Zielbereichs**.
* Ein Ereignis läuft für denselben Datensatz nicht verschachtelt (Schutz gegen
  Endlosschleifen). Ein Template, das sich nicht laden lässt, löst eine Exception aus.
* Die API steht über `$this->cms->api('record.search', [...])` bereit
  ([16](16-api.md#aus-templates-und-custom-tools)) – im Backend als der Benutzer, der
  speichert; bei einem Frontend-Benutzer (`CMS:USER_INPUT`) gibt es keine.

## Editor-Unterstützung und Meldungen

`styled_code` enthält den Quelltext mit Markup (`c9-command`, `c9-php`); Parameter mit
`'ref' => 'template'` werden zu Links auf das verwiesene Template. Der Parser meldet
Fehler mit Zeilennummer (unbekannter Befehl, unbekannter Parameter, fehlende Klammer)
und **Hinweise** (`TemplateParserTrait::inlineHandlerWarnings()`, gelb dargestellt):
`<script>` ohne `(CMS:NONCE)` sowie `on…`-Attribute und `javascript:`-Adressen im
HTML-Teil – beides blockiert eine aktive Script-Policy. Das Widget
„Template-Meldungen" listet alle Templates mit Meldungen.

## Frontend-Editing

`(CMS:FED)` gibt für angemeldete Benutzer einen Knopf zur Eingabemaske des
laufenden (oder mit `id` genannten) Datensatzes aus – als
`<a href="#" data-c9-fed="42" class="c9-fed">`, ohne Handler-Attribut.
`(CMS:FED_SCRIPT)` liefert einmal je Seite die Funktion `cFED()` und den Klick-Listener
(mit Nonce), üblicherweise im Footer. Ausgegeben wird nur, wenn das signierte Cookie
`fed` gültig ist (`Cms::fedActive()`, [14](14-security-web.md#frontend-editing)). Das
Ziel im Backend steht damit an genau einer Stelle (`Cms::cmsFedScript()`), nicht in
jeder ausgelieferten Seite.
