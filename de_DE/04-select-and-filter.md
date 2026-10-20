# 04 – Suchen und Filtern: `Cms::select()`

`Cms::select()` ist der zentrale Lesezugriff – derselbe Code für Backend-Listen,
Backend-Suche und den Template-Befehl `(CMS:SELECT …)`.

```php
$result = $cms->select($param, $tableObj = null, $completedSchema = []);
while ($row = $cms->fetch($tableName, $schema)) { … }
```

| Parameter | Bedeutung |
|---|---|
| `$param` | Filter- und Sortierkriterien (s. u.) |
| `$tableObj` | Dataset, dessen Tabelle durchsucht wird. **Ohne Angabe: `record`** – nur dann gibt es virtuelle Felder |
| `&$completedSchema` | Schema inklusive virtueller Felder – nur nötig, wenn lesbare Beschriftungen im Filtertext erscheinen sollen |

Das Ergebnis liegt danach im aktuellen `Database`-Result und wird mit `$cms->fetch()`
bzw. `$dataset->fetch()` zeilenweise abgeholt. Rückgabe von `select()`:

```php
[
    'filter_text'  => 'Name = Hen*',   // nur mit filtertext=true
    'sort_text'    => 'Name aufsteigend',
    'default_sort' => 'name ASC',
    'used_sort'    => 'name ASC',
    'sort'         => ['name'],
    'order'        => ['ASC'],
]
```

Trefferzahlen stehen in `$db->num_rows` und – mit `count => 'true'` –
`$db->num_rows_unlimited`.

## Parameter

| Schlüssel | Bedeutung |
|---|---|
| `filter` | `feldname => wert` oder `feldname => [wert1, wert2]` |
| `filterop` | Vergleichsoperatoren je Feld, indexgleich zu den Werten |
| `filtermode` | Verknüpfung gleichnamiger Werte: `OR` (Vorgabe) oder `AND` |
| `sort` | Feldname(n), optional mit Richtung: `'name ASC'` oder `['name', 'position']` |
| `order` | Richtungen, indexgleich zu `sort` |
| `skip`, `limit` | Versatz; Höchstzahl (`< 1` = unbegrenzt) |
| `count` | `'true'`: zusätzliche Zählabfrage für die Paginierung |
| `result` | `'false'`: nur zählen |
| `filtertext` | `'true'`: lesbaren Filter- und Sortiertext erzeugen |
| `debug` | `'true'`: SQL und Bind-Werte ausgeben |
| `request` | welche der Schlüssel `filter`, `filterop`, `filtermode`, `skip`, `limit`, `sort`, `order` zusätzlich aus `$_REQUEST` übernommen werden (`'all'` oder Liste; `filter[feld]` nur dieses Feld, `filter_op`/`filter_mode` als Schreibweise des Template-Befehls) |

**Nur aus `$param`, nie aus dem Request:**

| Schlüssel | Bedeutung |
|---|---|
| `sql` | zusätzliche WHERE-Bedingung(en), Rohtext – String oder Liste, jede geklammert angehängt |
| `src_id`, `relation` | Datensätze, auf die das Verweisfeld `relation` des Datensatzes `src_id` zeigt; `relation` ohne `src_id`/`dest_id` findet nichts |
| `dest_id`, `relation` | Rückrichtung: Datensätze, die über `relation` auf `dest_id` zeigen |
| | `relation`: ein Feldname, mehrere kommagetrennt (ODER) oder `any` (jede Zeile in `data_rel`) |
| `nextof` | mit `src_id`/`relation` (genau ein Feld): nur die Nachbarn des Datensatzes `nextof` in der Verweisliste |

Der Request kann eine Abfrage nur **einschränken** (`Cms::requestParams()`):

* `skip`, `limit`, `sort`, `order` aus `$param` ersetzen den Request-Wert vollständig;
  `limit` aus dem Request gilt nur als Zahl ≥ 1.
* Für jedes Hauptfeld, das `$param['filter']` filtert, übernimmt der Request weder
  `filter` noch `filterop`/`filtermode` – sonst ließe sich `filter[area_id]=5` durch einen
  zweiten Wert (ODER) erweitern oder per `filterop[area_id]=!=` umkehren. Hauptfeld heißt:
  `autor.name` gehört zu `autor`, `area` und `area_id` sind dasselbe Feld (der Bereichsname
  wird an `area_id` angehängt).
* Felder in `Cms::$unsearchableTypes` (`password`) sind nie filter- oder sortierbar, auch
  nicht aus `$param`: Ein Präfixfilter oder die Sortierung gäbe den Hash Zeichen für Zeichen
  preis. Der Filter liefert `0 = 1` und registriert `error_field_not_searchable`.

Falsch typisierte Request-Werte werden verworfen und als `error_param_invalid` registriert.
`CMS:SELECT` lässt in seinem `request` nur einzelne Schlüssel zu (`Cms::templateRequestKeys()`):
kein `all`, `filter`/`filterop`/`filtermode` nur mit Feldname – sonst dürften Besucher nach
jedem Feld der gefundenen Datensätze suchen. Die Backend-Listen nutzen weiter `'all'`; `CMS:ZIP`,
`RecordUser` und die API bauen ihre Parameter selbst und setzen kein `request`.

```php
$cms->select(['request' => 'skip,limit,sort,order', 'filter' => ['area_id' => 8]], $record);

// der auf 2411 folgende Eintrag in der Galerie von 2422 – bzw. mit order DESC der vorige
$cms->select(['src_id' => 2422, 'relation' => 'gallery', 'nextof' => 2411, 'limit' => 1]);

// alle Datensätze, die über irgendein Verweisfeld auf 4902 zeigen
$cms->select(['relation' => 'any', 'dest_id' => 4902]);
```

`filter[relation_any]=4902,4903` ist dasselbe als Filterwert (mehrere Ziele, ODER). Die
Backend-Liste (`RecordBackendList`) reicht `relation` + `dest_id` aus dem Request geprüft
weiter („Alle Verweise auf diesen Datensatz" im Metafeld-Reiter) und beschränkt ohne
festen Bereich auf die lesbaren Bereiche der Benutzergruppe (`Backend::areaAccessSql()`). Das
Suchfeld `_url` (Id, Slug mit/ohne Slug-Präfix oder URL) löst
`RecordService::urlFilter()` in `record_id` bzw. `slug` (+ `area_id`, `instance_id`) auf.

## Automatische Bedingungen

`select()` ergänzt ohne Zutun:

* `tmp = 0` und `deleted = 0`, wenn die Tabelle diese Spalten hat – Arbeitskopien und
  gelöschte Datensätze erscheinen nie versehentlich.
* **Felder im Papierkorb** werden ignoriert: Jeder Zugriff auf eine `data_*`-Tabelle
  (Filter, Sortierung, Relationen, `relation`/`src_id`, Slugs) bekommt
  `field_id NOT IN (…)`, sobald ein Feld dieses Namens im Papierkorb liegt
  (`Cms::liveFieldSql()`, Ids aus `InstanceCache::deletedFieldIds()`). Ein gelöschtes
  Feld verhält sich wie eines, das der Bereich nicht hat – auch eine Negation findet
  nichts. Liegt kein Feld im Papierkorb, bleibt die Abfrage unverändert.
* **Instanzbeschränkung** für `area`, `record`, `template`, sobald
  `Cms::setInstanceAccess()` gesetzt ist: im Frontend die Instanz der Domain plus ihre
  lesbaren Instanzen, im Backend die Instanzen der Benutzergruppe.
* **Bereichsbeschränkung** für `record`, `template` (`Cms::setAreaAccess()`): im Backend
  nur Bereiche, die die Benutzergruppe lesen darf ([11](11-permissions.md)).

`select()` kennt **keinen** Veröffentlichungsfilter – das Backend sieht alles. Der
Frontend-Befehl `CMS:SELECT` hängt ihn über `sql` an (`Cms::publishedSql()`: Status 0,
`online_at` erreicht, `offline_at` nicht erreicht). Wer selbst über
`Database::query()` abfragt, muss Instanz- und Veröffentlichungsbedingung selbst
anhängen.

## Pseudo-Feldnamen

| Name | Wirkung |
|---|---|
| `_default` | das Default-Feld jedes Bereichs (JOIN auf `data_dft`); bei anderen Tabellen `getDefaultFieldName()` |
| `area` | Bereich über seinen **Namen**: `filter[area]=artikel`, `=artikel,topic`, `=!artikel`; eine Zahl gilt als `area_id`. `sort=area` sortiert nach Bereichsname |
| `instance` | nur Sortierung, für `area`/`template`: Reihenfolge der Instanzen im Backend |
| `verweis.feld` | Sortierung nach einem Feld des verknüpften Datensatzes |

`filter[area]` wird vor dem Aufbau der Abfrage über den Instanz-Dateicache in
`filter[area_id]` übersetzt – keine Unterabfrage. Zugelassen sind `=`, `==`, `!`, `!=`,
`!!`. Ein unbekannter Name liefert positiv gesucht **nichts** und wird als
`error_filter_value_invalid` registriert.

## Filter-Syntax

Gilt gleichermaßen für die Suchmaske im Backend, für `filter[…]` im Code und in
Templates.

* Alle gefüllten Felder werden mit **AND** verknüpft.
* Mehrere Werte **eines** Felds, per Komma getrennt, mit **OR** (`filtermode` schaltet
  auf `AND`).
* Enthält ein Wert ` OR ` oder ` AND `, wird er nicht am Komma geteilt, sondern als
  Ausdruck ausgewertet.

### Operatoren

| Eingabe | Bedeutung |
|---|---|
| `henrik`, `=henrik` | gleich (Vorgabe), `*` ist Platzhalter |
| `==henrik` | exakt gleich – `*` ist ein normales Zeichen |
| `!henrik`, `!=henrik` | ungleich |
| `!*mann` | endet nicht auf „mann" |
| `!!*mann` | ungleich dem Literal `*mann` |
| `>100`, `>=100`, `<100`, `<=100` | Vergleich |
| `=` bzw. `==` allein | leerer Wert; bei Zahlen `0` |
| `(empty)` / `!(empty)` | kein eigener Inhalt in dieser Sprache (leer oder Fallback) / hat einen; bei Datei-Feldern (bare oder `.filename`): keine Datei (`NOT EXISTS` einer `data_fil`-Zeile mit Dateinamen) / eine Datei; bei Verweisfeldern (relation/pointer, auch n:1): kein Verweis auf einen produktiven Datensatz / mindestens einer – Kanten auf Arbeitskopien und gelöschte Ziele zählen nicht |

`*` steht für beliebig viele Zeichen (intern `LIKE` je Schreibvariante, siehe unten;
`%` und `_` im Suchwert werden maskiert): `Hen*`, `*mann`, `*berg*`. Ausdrücke in einem Feld: `Hen* OR Her*`,
`>2 AND <4`, `!2 AND !3`. Ein Wert, der mit einem HTML-Tag beginnt (`<section>`,
`</p>`, `<!--`), ist kein Operator (`Cms::extractOperator()`) – enthalten:
`*<section>*`.

Im Template darf der Operator auch zwischen Schlüssel und Wert stehen
(`filter[preis] >= 100`); der Compiler stellt ihn dem Wert in Zweizeichen-Form voran,
der Wert selbst bleibt dann frei von Operatoren (siehe
[06](06-template-compiler.md#syntax)).

### Kollation je Sprache

Text- und Zeichenfelder (`text`, `chars`, `_default`, Dateiname und OCR-Text) werden
nach den Regeln der Sprache verglichen und sortiert, in der gesucht wird. Die Spalten
stehen auf der neutralen Kollation; weicht die Sprache ab, hängt `Cms::collate()` ein
`COLLATE` an den Spaltenausdruck – in Bedingungen und in `ORDER BY`:

```sql
`_vf_0`.`data` COLLATE utf8mb4_de_pb_0900_ai_ci = ?
```

* **Ableitung:** `Camindo\Core\Collation::forLocale()` aus `language.code` (`de_AT` →
  `de`): Deutsch `de_pb` (ä = ae, ö = oe, ü = ue), Schwedisch `sv` (ö ≠ o), Dänisch/
  Norwegisch (aa = å), Türkisch usw. – nur `_ai_ci`-Varianten, sonst neutral (auch
  Chinesisch/Japanisch, die MySQL nur akzentgenau kennt). Kandidaten nacheinander:
  `utf8mb4_{x}_0900_ai_ci` (MySQL 8), `utf8mb4_uca1400_{name}_ai_ci` (MariaDB ≥ 10.10),
  `utf8mb4_{name}_ci`; es gilt die erste, die der Server kennt.
* **Cache:** `InstanceCache` legt das Ergebnis je Sprache in `instances.php` ab
  (`collations`, `InstanceCache::collation()`); `''` heißt „wie die Spalte“ – dann
  entfällt `COLLATE`. Eine Kollation im Ausdruck verhindert einen Bereichszugriff auf
  `data` im Index; `language_id` und `fieldname_id` bleiben nutzbar.
* **Welche Sprache:** Mit Sprachkontext (Frontend) die angeforderte. Ohne (Backend) die
  der Instanzen aus `filter[area_id]` (sonst aller) laut `$languageMap`; ergeben sie
  verschiedene Kollationen, bleibt es bei der neutralen – ein `COLLATE` gilt für die
  ganze Abfrage (`Cms::textCollation()`). Andere Tabellen als `record` vergleichen mit
  der Spaltenkollation.
* **Platzhalter:** `LIKE` vergleicht Zeichen für Zeichen; ein Zeichen entspricht dort
  nie zweien (`weiß` LIKE `'weiss%'` ist in jeder Kollation falsch). Deshalb erzeugt
  `Collation::likeVariants()` die Schreibvarianten, die die Kollation gleichsetzt
  (überall ß/ss, æ/ae, œ/oe; Deutsch ä/ae, ö/oe, ü/ue; Dänisch/Norwegisch å/aa,
  höchstens 16), verknüpft mit `OR`. `LOCATE()` ist kein Ersatz: MySQL vergleicht dort
  nur Abschnitte gleicher Byte-Länge (`LOCATE('e', 'Café')` = 0).
* **Volltext:** `MATCH()` kennt kein `COLLATE` je Abfrage. Deshalb steht
  `data_ftx.data` auf `utf8mb4_0900_as_ci` (akzentgenau), und `Cms::indexText()` faltet
  Text und Suchbegriff mit `Collation::fold()` nach den Regeln der Sprache – per ICU
  (`\Collator`, Stufe primär), das dieselben CLDR-Regeln kennt wie MySQL: neutral
  „Müller“ → `muller`, Deutsch `mueller`, Schwedisch bleibt „förvaring“. Gefaltet wird
  jede `data_ftx`-Zeile in ihrer Sprache (`_refreshFulltext()`), der Suchbegriff in der
  Suchsprache; ergeben mehrere Suchsprachen (Backend) verschiedene Begriffe, wird mit
  jedem gesucht (`OR`). Metaspalten anderer Tabellen (`admin.ft`) bleiben ungefaltet.
  Ändern sich die Regeln einer Sprache für vorhandenen Volltext – Sprache kommt zu einer
  Instanz oder bekommt ein anderes Kürzel –, markieren die Hooks von Instance und
  Language ihn mit `LanguageData::markFulltextStale()` als veraltet, wenn
  `Collation::sameRules()` verneint (Neuaufbau siehe
  [Volltextsuche](02-data-model.md#volltextsuche)).
* **MariaDB:** Kennt der Server `utf8mb4_0900_ai_ci` bzw. `_as_ci` nicht, ersetzt
  `SqlFile` sie beim Einspielen von `schema.sql` und Migrationen durch die nächstbeste
  (`Collation::portable()`: `uca1400_ai_ci`/`unicode_ci` bzw. `uca1400_as_ci`/`bin`);
  der Installer legt die Datenbank ebenso an.

### Datumsfelder

Unvollständige Angaben werden aufgefüllt – bei `>` und `=` auf den Anfang, bei `<` auf
das Ende des Zeitraums: `>=2023` ab 01.01.2023 00:00:00, `<2023-06` bis 30.06.2023
23:59:59. Lokale Schreibweisen der Locale werden verstanden.

### Verschlüsselte Felder

Verglichen wird der deterministische HMAC (`data_enc.hash`), nicht der Geheimtext.
Daraus folgt: nur exakte Gleichheit und Ungleichheit, byte-genau, keine Platzhalter,
keine Bereiche. Ein Platzhalter oder Bereichsoperator ist ein **Fehler** und liefert
nichts (`1=0`) – bewusst nicht „Bedingung weglassen", das gäbe alle Datensätze zurück.

### Volltextfelder

Boolean-Syntax von MySQL: `müller`, `+müller -gesperrt`, `müller meier`. Keine
Platzhalter. Ein Wort, das `indexText()` in mehrere zerlegt (`sofa-weiss`), wird zur
Wortfolge `"sofa weiss"` (mit `*` am Ende: jedes Wort mit dem Operator). Bleibt nach
`Cms::indexText()` kein Wort übrig (nur Tags, Satzzeichen), trifft die Bedingung nichts
(`1=0`) statt wegzufallen. `filter[fulltext]` trifft den
[Standard-Volltext](02-data-model.md#standard-volltext-fulltext) jedes Bereichs – auch ohne `area`. Index und Suchbegriff sind nach den Regeln der
Sprache gefaltet, siehe [Kollation je Sprache](#kollation-je-sprache).

### Baum-Filter

`depth` ist ein Modifikator für `filter[parent_id]`, relativ zum Elternknoten:

| `depth` | Wirkung |
|---|---|
| fehlt oder `< 0` | alle Nachkommen |
| `0` | leeres Ergebnis |
| `1` | direkte Kinder |
| `n` | bis zur relativen Tiefe n |

`['parent_id' => '', 'depth' => 1]` liefert die Wurzelknoten. Der Bereich des
Elternknotens wird automatisch mitgefiltert.

### Grenzen

Nicht möglich sind reguläre Ausdrücke, Komma **und** `OR`/`AND` im selben Feld, Werte,
die selbst ein Komma enthalten, sowie Platzhalter in verschlüsselten und
Volltextfeldern.

## Wie die Abfrage entsteht

1. `filter()` baut die WHERE-Klausel und sammelt einen **JOIN-Kontext**; jeder
   virtuelle Feldzugriff registriert genau einen JOIN, den Filter und Sortierung
   gemeinsam nutzen.
2. Jeder virtuelle Skalarwert wird auf seine `data_*`-Tabelle gejoint, eingeschränkt
   über `fieldname_id` **und** `language_id` (Index `idx_filter(language_id,
   fieldname_id, data)`). Die Sprache liefert `Cms::languageExpr()`: im Frontend die
   angeforderte, sonst die Hauptsprache jedes Datensatzes. Weil jede Sprache genau eine
   Zeile hat, bleibt der JOIN 1:1.
3. Bei `record` wird zusätzlich `record_cache` der Sprache als Spalte `cache` gejoint.
4. Relationen werden als `EXISTS`-Unterabfrage oder als JOIN-Gruppe gebaut, Dateifilter
   über `EXISTS`; nach Datei-Metadaten (`filename`, `mime_type`, `extension`, `size`,
   `width`, `height`) lässt sich sortieren.
5. Die Zählabfrage bekommt nur die Filter-JOINs.
6. `DISTINCT` nur, wenn ein JOIN Mehrfachzeilen erzeugen kann.
7. Alles läuft über Prepared Statements.

`debug => 'true'` zeigt das fertige SQL.

## Unterabfragen

`select()` überschreibt das aktuelle Ergebnis. Wer während einer Iteration eine zweite
Abfrage stellt, sichert es:

```php
$this->db->pushResult();
$cms->select(['filter' => ['parent_id' => $id]], $record);
while ($child = $cms->fetch()) { … }
$this->db->popResult();
```

## Paginierung

`Pagination` baut aus `Cms::$list_rows`, `$db->num_rows` und `$db->num_rows_unlimited`
Blätterleisten. Die Klasse wird erst beim ersten Zugriff geladen
(`LazyServicesTrait`, `$this->pagination` in `Backend` und `Page`).

| Methode | Template-Befehl | Ausgabe |
|---|---|---|
| `jumpList($param)` | `(CMS:JUMP_LIST)` | `1 … 5 6 7 8 9 … 15` |
| `jumpPrev($param)` | `(CMS:JUMP_PREV)` | Link auf die vorige Seite |
| `jumpNext($param)` | `(CMS:JUMP_NEXT)` | Link auf die nächste Seite |

Wichtige Parameter von `jumpList()`: `skip`, `maximum` (7), `boundaries` (erste/letzte
Seite immer), `separator`, `tpl_link`, `tpl_active`, `ellipsis`, `prefix`, `suffix`,
`ifempty`. `Cms::$list_rows` kommt aus `admin.skiplist_default` (20). Im Backend gibt es
fertige Hüllen: `Backend::paginationTop()`, `paginationTopShort()`,
`paginationBottom()`.

`{href}` in den Vorlagen ist HTML-maskiert (`&amp;` statt `&`), `'` und `\` sind
prozentkodiert – siehe [14](14-security-web.md).

## `fetch()` – Werte aufbereiten

`$cms->fetch($tableName, $schema)` holt die nächste Zeile und bereitet sie anhand des
Schemas auf: virtuelle Felder aus `cache`, Entschlüsselung, JSON-Dekodierung,
Umrechnung von Datumswerten in die Zeitzone des Benutzers, HTML-Filter auf
String-Werte ([14](14-security-web.md#html-filter)).

Formatierer für die Anzeige:

```php
$cms->_formatDate($iso, $displayTimezone = false);
$cms->_formatTime($iso, $displayTimezone = false);
$cms->_formatFloat($value, $fractionDigits = 0);
$cms->_nl2br($text);   $cms->_strlimit($text, $limit);   $cms->_strip_tags($text);
$cms->utcToUserTZ($date);   $cms->indexText($text);   $cms->mb_ucfirst($string);
```
