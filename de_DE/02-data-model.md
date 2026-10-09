# 02 – Datenmodell

Referenz ist `Core/install/database/schema.sql` (Struktur) mit `seed.sql`
(Stammdaten). Alle Tabellen: InnoDB, `utf8mb4`, Kollation `utf8mb4_0900_ai_ci` (neutral,
unabhängig von Akzenten und Groß-/Kleinschreibung; Ausnahmen wie Hashes und Tokens sind
`ascii`/`latin1`; `data_ftx.data` ist akzentgenau `utf8mb4_0900_as_ci`). Sprachabhängige Vergleiche setzt `Cms::select()` per `COLLATE`, siehe
[Kollation je Sprache](04-select-and-filter.md#kollation-je-sprache). Datums- und
Zeitwerte werden **in UTC** gespeichert und erst für die Anzeige umgerechnet.

## Das Grundprinzip: benutzerdefinierte Felder in typgetrennten Tabellen

Ein Redakteur legt ***Inhaltsbereiche*** (`area`) an und gibt ihnen Eingabefelder
(`field`). Ein Datensatz (`record`) gehört zu genau einem Bereich. Die Feldinhalte
stehen **nicht** in `record`, sondern in einer `data_*`-Tabelle je Datentyp:

```
record (Metadaten)        field (Felddefinition)      fieldname (globale Namensregistrierung)
   │                         │                            │
   └──────────┬──────────────┴────────────┬───────────────┘
              ▼                           ▼
        data_txt / data_int / data_fil / data_rel / …
        PK (language_id, record_id, fieldname_id)
```

### `fieldname` – ein Name, ein Typ

`fieldname` registriert jeden Feldnamen systemweit: `name → (fieldname_id, type)`,
`UNIQUE(name)`. Daraus folgt:

* Ein Feldname ist an **genau einen Typ** gebunden. Wer `title` als `text` anlegt, kann
  `title` in keinem anderen Bereich als `integer` anlegen.
* `Cms::select()` kann über Bereiche hinweg nach `filter[title]` filtern, ohne die
  `field_id` je Bereich zu kennen.

Der Primärschlüssel der `data_*`-Tabellen enthält deshalb `fieldname_id`, nicht
`field_id`; `field_id` ist eine Zusatzspalte für den Fremdschlüssel beim Löschen einer
Felddefinition. Unbenutzte `fieldname`-Einträge werden automatisch entfernt.

Feldnamen genügen `^[a-z][a-z0-9_]+$` (Flag `name` in `Dataset`: mit einem Buchstaben
beginnen, mindestens zwei Zeichen – `_…` bleibt Pseudo-Feldnamen wie `_default`
vorbehalten), `fulltext` ist reserviert, und sie dürfen nicht mit dem Kürzel eines **anderen**
Typs beginnen:

```
bit_ chr_ cur_ dat_ dec_ enc_ fil_ flt_ ftx_ rid_ ids_ int_ jsn_ pwd_ txt_ tim_ tsl_
ptr_ (pointer)  pn1_ (pointer1)  rel_ (relation)  rn1_ (relation1)  lnk_ (link)
```

### Typ → Tabelle

`Cms::$dataSuffix` ist die maßgebliche Zuordnung:

| Typ | Tabelle | Spalte `data` | Gruppe |
|---|---|---|---|
| `bit` | `data_bit` | `tinyint unsigned` | numerisch |
| `chars` | `data_chr` | `varchar(255)` | Text |
| `currency` | `data_cur` | `bigint` (Cent) | numerisch |
| `datetime` | `data_dat` | `datetime` | Datum |
| `decimal` | `data_dec` | `decimal(20,10)` | numerisch |
| `encrypted` | `data_enc` | `mediumblob` + `hash` | Text |
| `file` | `data_fil` | eigene Spalten, s. u. | Array |
| `float` | `data_flt` | `double` | numerisch |
| `fulltext` | `data_ftx` | `mediumtext` + FULLTEXT-Index | – |
| `id` | `data_rid` | `bigint unsigned` (FK `record`) | numerisch |
| `integer` | `data_int` | `bigint` | numerisch |
| `json` | `data_jsn` | `mediumtext` + Multi-Value-Index | Array |
| `password` | `data_pwd` | `varchar(255)` – Hash, nie im Cache | Text |
| `pointer`, `pointer1`, `relation`, `relation1`, `link` | `data_rel` | Kantentabelle | Relation |
| `text` | `data_txt` | `mediumtext` | Text |
| `time` | `data_tim` | `time` | – |
| `timestampl` | `data_tsl` | `datetime(3)` | Datum |

Die Gruppen entsprechen Arrays in `Cms` (`textTypes`, `numericTypes`, `dateTypes`,
`arrayTypes`, `relationTypes`, `singleRelationTypes`, `multiAreaTypes`,
`dependentTypes`, `multilingualTypes`).

**Sonderfälle**

* `data_dft` – genau eine Zeile je Datensatz und Sprache mit dem Textinhalt des
  ***Default-Felds*** des Bereichs (`area.default_field_id`). So lässt sich der „Titel"
  jedes Datensatzes mit einem JOIN holen, ohne zu wissen, welches Feld es ist
  (Pseudo-Feldname `_default` in `select()`). Ist das Default-Feld verschlüsselt, ein
  Passwort oder JSON, steht dort `ID {record_id}` – kein Klartext.
* `data_enc` – Klartext AES-verschlüsselt (Schlüssel `.camindo.key`), daneben ein
  deterministischer HMAC in `hash` für die Gleichheitssuche.
* `data_fil` – statt `data` die Metadaten der Datei: `extension`, `mime_type`,
  `filename`, `size`, `width`, `height`, `token`, `version`, `file_language_id` und ein
  JSON-Feld `data`. Die Datei selbst liegt im Dateisystem.
* `data_rel` – Kantentabelle `(src_id, fieldname_id, dest_id)` mit `position` für
  sortierbare Verknüpfungen; ein zweiter Index bedient die Gegenrichtung. Hat keine
  `language_id`.

## Mehrsprachigkeit

Alle `data_*`-Tabellen außer `data_rel` führen `language_id` im Primärschlüssel. Ein
Datensatz hat je Feld **genau eine Zeile je Sprache seiner Instanz** (Hauptsprache plus
`rel_instance_language`):

| Feld | Zeile je Sprache enthält |
|---|---|
| einsprachig (`field.multilingual = 0`) | denselben Wert in allen Sprachen |
| mehrsprachig | den Wert der Sprache, ohne Eingabe den Wert der Hauptsprache (***Fallback***) |
| `file`, mehrsprachig | je Sprache die eigene Datei (`file_language_id`), sonst die der Hauptsprache |
| `fulltext` | den Volltext aus den Quellfeldern dieser Sprache |

Der Grund: `select()` filtert und sortiert eine Sprache mit einer einzigen
JOIN-Bedingung (`language_id = …`), und der Fallback gilt **je Feld**. Ob eine Zeile ein
Fallback ist, steht in ihrer Spalte `empty` (1 = keine eigene Eingabe); damit findet der
Filterwert `(empty)` die noch nicht übersetzten Datensätze.

Mehrsprachig sein können `bit`, `chars`, `currency`, `datetime`, `decimal`,
`encrypted`, `float`, `integer`, `json`, `text`, `time`, `timestampl` und `file`. Nicht:
`fulltext` (wird erzeugt), Relationen, `id`, `password` und Metafelder.

Geschrieben werden die Zeilen von `VirtualFieldsTrait::saveVirtualFields()`. Ändert
sich die Sprachliste einer Instanz, zieht `LanguageData` den Bestand nach (neue Sprache:
Zeilen der Hauptsprache kopieren; entfernte Sprache: Zeilen, `record_cache` und
`privacy_template` löschen, eigene Dateien löschen – auch im Papierkorb –, Datei-Fallbacks
auf die Hauptsprache umstellen). Eine Sprache, die systemweit gelöscht wird, verschwindet
so aus allen Instanzen (`LanguageData::deleteLanguage()`); Hauptsprache einer Instanz
darf sie dabei nicht sein.

### `record_cache` – Inhalts-Cache je Sprache

```
record_id, language_id, cache MEDIUMTEXT      PK (record_id, language_id)
```

Das JSON-Abbild aller nicht-relationalen virtuellen Feldinhalte eines Datensatzes, eine
Zeile je Sprache, mit Fallback-Werten:

```json
{"title": ["chars", "Artikel"], "text": ["text", "English text", 1]}
```

Das dritte Element `1` kennzeichnet einen Fallback. `select()` joint die Zeile der
angeforderten Sprache als Spalte `cache`, `fetch()` liest die virtuellen Felder daraus –
eine Liste lässt sich so rendern, ohne je Zeile alle `data_*`-Tabellen zu joinen. Fehlt
die Zeile, baut `Cms::createRecordCache()` alle Sprachen neu auf. Passwörter stehen dort
nie, verschlüsselte Inhalte als Base64-Geheimtext.

## Kerntabellen

### `area` – Inhaltsbereiche

```
area_id, instance_id, type ENUM('record','template','folder','profile','ugc'), name, label,
icon, details JSON, dependent,
position, depth, parent_id, parent_ids JSON, lft, rgt,       ← Baum
default_field_id, sort_field_id, sort_order, default_template_id, list_template_id,
overview_template_id, login_name_field_id, login_password_field_id, deleted …
```

* `name` (`^[a-z][a-z0-9_]+$`, eindeutig im Instanz-Verbund) ist der Bezeichner für
  Templates: `filter[area]=artikel`.
* `type`: `folder` = Ordner im Baum, `record` = Inhalte, `profile`/`ugc` = Profile und
  Beiträge der Frontend-Community ([12](12-community.md)), `template` = Templates.
* `instance_id`: Website des Bereichs; `record.instance_id` und `template.instance_id`
  sind Kopien davon (erspart `select()` einen JOIN) und werden bei einem Wechsel
  nachgezogen.
* `dependent = 1`: ***abhängiger Bereich*** – seine Datensätze gehören je einem
  Quelldatensatz, der sie über ein Feld vom Typ `link` hält (s. u.).
* `list_template_id`: Template nur für die Backend-Darstellung (letzte Spalte der
  Datensatzliste, ausführliche Lookup-Vorschau).
* Trigger-Templates (Ereignis-Methoden, die bei jedem Speichern und Löschen laufen,
  [06](06-template-compiler.md#trigger-templates)) stehen nicht in `area`, sondern in
  `rel_area_trigger` (s. u.).
* `overview_template_id`: Template für die Adresse, die nur aus dem Slug-Präfix
  besteht (`/de_DE/termine`) – eine Übersicht ohne Datensatz ([06](06-template-compiler.md#url-und-slug)).
* `details` (JSON): u. a. `tabs` (mehrsprachige Maske mit Reitern), `slug_prefix` je
  Sprache, `profile_area_ids` (`ugc`), `second_factor` (`profile`).

### `field` – Eingabefelder

| Spalte | Bedeutung |
|---|---|
| `area_id`, `name`, `type`, `label` | Zuordnung und Identität |
| `display_type` | Darstellung im Backend (`text`, `textarea`, `select`, `lookup`, `switch`, `upload`, `thumbnail`, `embed` …) |
| `min`, `min_op`, `max`, `max_op`, `pattern`, `flags`, `default` | Eingabeprüfung, siehe [03](03-dataset-api.md#validierung) |
| `rel_field_id`, `rel_area_id`, `rel_field_area` | Gegenfeld und Zielbereich(e) einer Relation |
| `parent_field_id` | Thumbnail-Feld: das Feld, aus dem es erzeugt wird |
| `custom_order` | Verknüpfungen manuell sortierbar |
| `visibility_input`, `visibility_search`, `visibility_list` | wo das Feld erscheint |
| `default_field`, `meta`, `multilingual` | Default-Feld; Metafeld; Inhalt je Sprache |
| `details` (JSON) | u. a. `extension`/`extension_options`, `user_auth`, `profile_area_ids`, `lookup_detailed` |

Die Konverter eines Thumbnail-Felds stehen sortiert in `rel_field_converter`.
Typ und Mehrsprachigkeit eines bestehenden Felds ändert nur `Field::convert()` (Button
„Typ ändern"): Text ↔ Zahl wird zeilenweise umkopiert, `relation` ↔ `relation1` behält
bei `…1` nur die erste Verknüpfung, ein- ↔ mehrsprachig stellt das `empty`-Kennzeichen
bzw. die Dateien um. `conversionTargets()` nennt die möglichen Ziele,
`conversionLossy()` warnt vor Datenverlust. Bearbeitungssperren hält die Umwandlung
nicht auf: Der Arbeitsstand offener Arbeitskopien (ihr `record_cache`) bleibt stehen und
bekommt den neuen Typ (`Record::retypeWorkingCopies()`, bei Datei-Feldern samt neu
hochgeladener Dateien `Record::migrateWorkingCopyFiles()`), nur der Cache der produktiven
Datensätze wird verworfen; `data_dft` zieht nach. Fällt ein Feld aus den
Volltext-Quellen (etwa nach `encrypted`), werden die Volltexte aller nicht temporären
Datensätze des Bereichs neu aufgebaut – auch der gelöschten.

### `record` – Datensätze

```
record_id, area_id, instance_id, template_id,
position, depth, parent_id, parent_ids JSON, lft, rgt,     ← Baum
online_at, offline_at, status_id, needs_review,            ← Veröffentlichung
created_by, changed_by, created_at, changed_at (datetime(3)),
owner_id,                                                  ← Besitzer im Backend (Benutzer)
user_id,                                                   ← Besitzer im Frontend (Profil)
tmp, origin_id, locked, deleted                            ← Arbeitskopien, Papierkorb
```

### Weitere Tabellen

| Tabelle | Zweck |
|---|---|
| `admin` | Benutzer; Passwort-Hash, Signatur `sig`, verschlüsselte Kontaktdaten, `settings` JSON (Sprache, Zeitzone, Widgets …), zweiter Faktor |
| `group`, `rel_group_instance`, `rel_group_area`, `rel_group_language`, `rel_group_status` | Benutzergruppen und ihre Rechte ([11](11-permissions.md)) |
| `admin_session`, `admin_session_record` | Backend-Sessions; welche Arbeitskopie zu welcher Session gehört |
| `admin_login_attempt`, `admin_recovery_code`, `login_device`, `admin_passkey` | Anmeldeschutz, Wiederherstellungscodes, bekannte Geräte, Passkeys ([13](13-security-authentication.md)) |
| `instance`, `rel_instance_language`, `rel_instance_domain`, `rel_instance_instance` | Websites, ihre Sprachen, Domains und lesbaren Instanzen |
| `language` | Sprachen: `code` (URL-Segment, z. B. `de_DE`), `backend_code` (Snippet-Sprache), `timezone` |
| `template` | Templates: `code` (Kopie des Quelltexts, nur zur Suche), `styled_code`, `error_text`, `standalone`, `type` (`scheduler` = zeitgesteuert) |
| `rel_area_template` | welche Templates ein Bereich verwenden bzw. per URL erzwingen darf |
| `rel_area_trigger` | Trigger-Templates eines Bereichs, `position` = Ausführungsreihenfolge |
| `status` | Datensatz-Status (Symbol, Beschriftung, Farbe) |
| `converter`, `rel_field_converter` | konfigurierte Prozessoren und ihre Zuordnung zu Thumbnail-Feldern |
| `rel_fulltext_text` | Quellfelder eines Volltextfelds (optional über ein `link`-/`relation`-Feld: `rel_field_id`) |
| `component`, `system` | installierte Komponenten; Versionsstand des Cores |
| `bgproc` | Hintergrundprozesse |
| `scheduler` | zeitgesteuerte Aufgaben |
| `message` | interne Nachrichten (verschlüsselt) |
| `history` | zuletzt bearbeitete Datensätze und Templates je Benutzer |
| `audit_log` | Metadaten des Audit-Logs; die Inhalte liegen in `var/audit/` |
| `user_session`, `user_basket`, `user_token`, `user_login_attempt`, `user_second_factor`, `user_recovery_code` | Frontend-Community |
| `rate_limit` | Drosselung von Anfragen |
| `doc_section`, `doc_keyword` | Index der Handbücher |

## Bäume: Nested Set und Adjacency List

`record` und `area` führen beides parallel:

| Spalte | Rolle |
|---|---|
| `parent_id` | direkter Elternknoten |
| `position` | Reihenfolge unter den Geschwistern |
| `depth` | Ebene, **0-basiert** |
| `lft`, `rgt` | Nested Set – Teilbäume ohne Rekursion |
| `parent_ids` | JSON-Liste aller Vorfahren – Breadcrumbs ohne Abfrage |

Bei `record` gehört jeder Baum **einem Inhaltsbereich**: `lft`/`rgt` beginnen je Bereich
bei 1, jede Abfrage über `lft`/`rgt` filtert deshalb auch `area_id`. Ob ein Bereich
überhaupt einen Baum führt, entscheidet ein Metafeld namens `parent_id` im Bereich
(`isTreeArea()`); ohne es bleiben die Baumspalten leer und es findet keine Baumpflege
statt. `area` führt einen Baum über die ganze Tabelle.

Die Reihenfolge wird nicht als Zahl eingegeben, sondern als **vorangehender Datensatz**
(`previous_id`); `TreeManagerTrait::calculatePosition()` rechnet daraus `position`.
Umsortiert wird nur, wenn der Request `previous_id` oder einen anderen `parent_id`
mitbringt – ein `load()` + `save()` lässt den Datensatz an seinem Platz.

| Methode (`TreeManagerTrait`) | Zweck |
|---|---|
| `isTreeArea($areaId)` | führt der Bereich einen Baum? |
| `loadTree($rootId, $levels, …)` | Teilbaum laden |
| `liftChildren()` | Kinder eine Ebene hochziehen |
| `copySubtree()` | Teilbaum kopieren |
| `subtreeIds($data, $mode)` | Ids eines Teilbaums in Bearbeitungsreihenfolge |
| `resetTreeFields($areaId)` / `initTreeArea($areaId)` | Bereich flach machen / Baum aufbauen |
| `externalRebuildTree($areaId)` | `lft`/`rgt`/`depth` neu berechnen |

Enthält ein Dataset-Schema `parent_id`, **müssen** auch `parent_ids`, `lft`, `rgt`,
`position` und `depth` darin stehen – sonst wirft der Konstruktor. Reparatur eines
Bestands: `php Core/cron/rebuild_trees.php`.

## Relationen

| Typ | Bedeutung |
|---|---|
| `relation` | n:m, beidseitig sichtbar (Gegenfeld in `field.rel_field_id`) |
| `relation1` | wie `relation`, aber dieses Feld hält höchstens eine Verknüpfung |
| `pointer` | gerichtete Verknüpfung, mehrere Zielbereiche möglich (`rel_field_area`) |
| `pointer1` | gerichtet, höchstens ein Ziel |
| `link` | gerichtet wie `pointer`, aber die Ziele **gehören** der Quelle (s. u.) |

„Höchstens ein Ziel" setzt der Code an zwei Stellen durch: `createRelations()` weist
mehrere Ziele für ein `…1`-Feld ab, und beim Check-in verliert ein Zieldatensatz, dessen
Gegenfeld nur ein Ziel erlaubt, seine bisherige Verknüpfung
(`TemporaryTrait::_enforceSingleRelations()`).

```php
// VirtualFieldsTrait
createRelations($colDef, $srcId, $destIds, $replace = true, $checkTmp = true);
removeRelations($colDef, $srcId, $destId = null);
swapRelationPosition($colDef, $srcId, $destId1, $destId2);
checkRelation($srcFieldId, $srcId, $destId);
```

### Fest verknüpfte Inhalte (`link`)

Ein `link`-Feld hält Datensätze aus **abhängigen Bereichen** als Bestandteile der
Quelle – etwa die Text-Bild-Blöcke einer Seite. `LinkTrait` sorgt für zwei Regeln:

1. **Je Zieldatensatz höchstens eine Quelle.** Ein Datensatz, der schon einem anderen
   gehört, wird abgewiesen; eine Kopie bekommt keine eingehenden `link`-Kanten.
2. **Die Ziele teilen das Schicksal der Quelle.** Löschen, Wiederherstellen, endgültiges
   Löschen, Kopieren und Verschieben laufen über die Record-Hooks auf die Inhalte weiter
   (`cascadeDependents()`, rekursiv).

Mit `display_type = embed` stehen die Masken der Inhalte in der Maske der Quelle
(`RecordBackendEdit::_embedBlock()`, Aktionen über `ajax_embed.php`). Die Inhalte werden
dann mit der Quelle ausgecheckt und beim Check-in in derselben Transaktion eingecheckt,
kopiert oder – wenn keine Kante mehr an ihnen hängt – gelöscht
(`checkInDependents()`); Abbrechen verwirft sie mit.

## Volltextsuche

Ein Feld vom Typ `fulltext` sammelt den Inhalt anderer Felder (`text`, `chars`, `json`)
in `data_ftx`; Zuordnung in `rel_fulltext_text`. Quellen sind Felder des eigenen
Bereichs oder – mit `rel_field_id` – Felder der Zielbereiche eines `link`-, `relation`-
oder `relation1`-Felds des Bereichs (Kanten in `data_rel` mit `src_id` = Datensatz);
`Field::fulltextSourceOptions()` nennt die wählbaren (die Maske gruppiert sie je
Zielbereich), `Field::hookOnSave()` weist andere ab. Im Pseudo-Feld `source_fields`
(Maske, API, XML) steht eine Quelle als `field_id` bzw. `"{rel_field_id}:{field_id}"`
(`Field::sourceKey()`); der XML-Export schreibt `rel_field_id` als Attribut. Nicht
wählbar: Felder hinter `pointer`/`pointer1` – ein Verweis ist eine lose, einseitige
Referenz (oft auf mehrere Bereiche), deren Texte den Datensatz nicht beschreiben.
Änderungen am **Zieldatensatz** erreichen den Volltext nur bei `link` (der Inhalt gehört
dem Datensatz, siehe unten); bei `relation` erst beim nächsten Speichern des Datensatzes
oder per Neuaufbau (Zeitplaner-Aufgabe `fulltext` ohne `only_stale`) – bewusst, ein
Schlagwort kann an Tausenden Datensätzen hängen.
Beim Speichern verarbeitet `refreshAffectedFulltext()` **nach** dem Check-in die
Volltextfelder des eigenen Bereichs, in die ein geändertes Feld eingeht, und alle mit
Quellen über ein `link`-/`relation`-Feld (die Kanten können sich geändert haben); ist
der Datensatz Inhalt eines `link`-Felds, auch die seines Besitzers (Kante in
`data_rel`). `refreshFulltext($fieldId)` baut komplett neu auf.
Der Text wird vorher mit `Cms::indexText()` normalisiert (HTML-Tags entfernt,
Entities aufgelöst, nur Buchstaben und Ziffern; `&shy;` verbindet, der Punkt trennt;
mit Binde- oder Unterstrich verbundene Wörter kommen getrennt UND zusammengeschrieben
hinein: `sofa-weiss` → `sofa weiss sofaweiss`; Suchbegriffe mit
`indexText($t, forSearch: true)` ohne die zusammengeschriebene Variante; dazu gefaltet
nach den Regeln der Sprache, `Müller` → `mueller` im Deutschen, siehe
[Kollation je Sprache](04-select-and-filter.md#kollation-je-sprache)). Eine Zeile je
Sprache der Instanz. Quelle kann
auch ein Upload-Feld sein (`file`, ohne Thumbnails): sein Beitrag ist
`data_fil.filename`; `Record::renameFile()` baut den Volltext nach.

**Standard-Volltext `fulltext`.** Jeder Datensatzbereich (`Area::CONTENT_TYPES`) hat
ihn – ohne Zeile in `field`. Der Name ist reserviert (`Field::FULLTEXT`) und steht fest
in `fieldname` (Typ `fulltext`, Stammdaten in `seed.sql`; `Field::cleanupFieldname()`
lässt ihn stehen, `VirtualFieldsTrait::standardFulltextFieldnameId()` legt ihn notfalls
an). Seine `data_ftx`-Zeilen haben `field_id` NULL. Quellen sind immer alle Felder vom
Typ `text`, `chars`, `json` und `file` (Upload, keine Thumbnails) des Bereichs und über
seine `link`-Felder die der Zielbereiche, eine Ebene (`standardFulltextSources()`);
geschrieben wird er von `refreshStandardFulltext()` nach jedem Speichern
(`refreshAffectedFulltext()`). `select()`
filtert über den Feldnamen und braucht keine Felddefinition: `filter[fulltext]` wirkt
in jedem Bereich und bereichsübergreifend; die Suchmasken bieten ihn von selbst an
(`RecordBackendSearch`). Ein `fulltext` aus älteren Export-Dateien übergeht der Import.
Anzeige: Datensatz-Info, Reiter „Metadaten" → „Volltext anzeigen"
(`RecordBackendInfo::fulltext()`, alle Volltexte je Sprache); im Reiter „Inhalte"
erscheinen Volltextfelder nicht.

**Veraltet statt sofort neu (`FulltextState`).** Strukturänderungen, nach denen der
Volltext eines ganzen Bereichs nicht mehr zu den Quellen passt – Quellen eines
Volltextfelds geändert oder neues Volltextfeld (`Field::hookOnSaved()`), Quellfeld
gelöscht (`hookOnDeleted()`) oder umgewandelt (`convert()`, auch ein ↔ mehrsprachig),
Zielbereiche eines `link`-Felds geändert, neue Faltungsregeln einer Sprache
(`LanguageData::markFulltextStale()`, alle Bereiche der Instanz) –, bauen ihn nicht
sofort neu auf, sondern setzen `area.fulltext_stale_at` (UTC, der früheste Zeitpunkt
bleibt). Betroffen sind auch die Bereiche, die über ein `link`-Feld auf den Bereich des
Felds zeigen (`FulltextState::areasUsingField()`). Ausnahme: `convert()` nach
`encrypted` baut sofort neu auf, sonst stünde Klartext im Index.
Die Detailansicht und die Feldliste des Bereichs zeigen dann den Hinweis mit
„Jetzt aktualisieren“ (`AreaBackend::fulltextNotice()` / `fulltext()`): Hintergrundprozess
`camindo.php fulltext --area=ID --token=…` (`Cli\FulltextCommand`).
Neuaufbau: `RecordService::rebuildFulltext()` – aufgerufen vom CLI-Befehl `fulltext`
(`--area=…`, `--stale`) und der Zeitplaner-Aufgabe `fulltext` (Option
`details.only_stale`) – schreibt nur `data_ftx`, ohne Arbeitskopie, Sperre und
Audit-Eintrag, für jeden Datensatz mit `tmp = 0` (auch im Papierkorb). Je Bereich läuft
er unter einer Sperre (`var/tmp/fulltext-{area_id}.lock`, exklusiv angelegt, alle 50
Datensätze aufgefrischt, nach 15 Minuten ohne Auffrischung verwaist); einen gesperrten
Bereich überspringt ein zweiter Lauf. Nach einem vollständigen Lauf fällt die
Markierung – nur, wenn sie nicht jünger ist als dessen Beginn.
Volltextfelder stehen weder in `record_cache` noch im Audit-Schnappschuss.

## Arbeitskopien

`record` wird nie direkt bearbeitet, sondern über eine ***Arbeitskopie***
(`TemporaryTrait`):

```
checkOut()        Kopie anlegen: tmp = 1, origin_id = Original
                  record_cache-Zeilen des Originals → Arbeitsstand der Kopie
                  Verknüpfungen mitkopieren, Bearbeitungssperre setzen

save(…, stayTemporary: true)
                  Zwischenspeichern (Reiterwechsel): nur der Arbeitsstand in record_cache
                  der Kopie; Uploads liegen schon unter der Kopie, Thumbnails noch nicht

save(…)  = checkIn()
                  Arbeitsstand + Eingaben zusammenführen und als Ganzes prüfen
                  data_*-Zeilen und record_cache des Originals schreiben
                  Dateien umbenennen (tmp-Id → record_id), data_fil schreiben
                  Thumbnails EINMAL erzeugen, Verknüpfungen zurückkopieren
                  Arbeitskopie löschen

cancel($tmpId)    Arbeitskopie samt Dateien verwerfen
```

Der Arbeitsstand liegt im selben Format wie der produktive Stand in `record_cache`,
unter der Id der Kopie; `VirtualFieldsTrait::_loadTmpData()`/`_saveTmpData()`
übersetzen ihn in die Arbeitsform (`feld => [typ, wert]`, mehrsprachig
`{language_id: eingabe}`).

* **Bearbeitungssperre:** Eine Arbeitskopie sperrt das Original für andere Benutzer,
  solange ihr `changed_at` jünger als `admin.edit_lock_sec` (Vorgabe 5400 s) ist. Beim Abmelden
  werden die Arbeitskopien der Session verworfen.
* **Uploads sofort, Thumbnails später:** Eine Datei übersteht so eine gescheiterte
  Prüfung, die teure Umwandlung läuft nur einmal.
* Eine vorhandene Arbeitskopie desselben Benutzers wird wiederverwendet
  (`findTemporaryByOrigin()`).

## Datensätze verschieben

Ein Datensatz kann in einen anderen Bereich wandern (`MoveTrait`). Weil Felder dem
Bereich gehören, ist das eine **Abbildung** `[quellfeld => zielfeld | '']` – zulässig
nur bei gleichem Typ und gleicher Mehrsprachigkeit (`moveCompatible()`). Der Weg führt
über eine Arbeitskopie: `movePrepare()` baut sie zum Zielbereich um, `moveSave()`
speichert sie mit den Regeln des Zielbereichs. Scheitert die Prüfung, zeigt das Backend
die Maske des Zielbereichs mit den Fehlern; Abbrechen lässt den Datensatz unverändert.

## Medienablage

```
public/media/
├── {instance_id}/.htaccess                     Domain-Sperre der Instanz
├── {instance_id}/{field_id}/{record_id}.{ext}  Uploads und Thumbnails
├── {instance_id}/{field_id}/.htaccess          nur bei geschützten Feldern
└── default/                                    Platzhalterbilder
```

Mehrsprachige Upload-Felder heißen `{record_id}_{language_id}.{ext}`. Pfade baut
ausschließlich `Cms::mediaFolder()` / `Cms::mediaPath()`. Immer gesperrt sind
ausführbare Endungen (`php`, `phtml`, `phar`, `cgi`, `htaccess` …,
`VirtualFieldsTrait::$forbiddenExtensions`); Endung und erkannter Mime-Typ müssen
zusammenpassen. Zugriffsschutz und Auslieferung: [07](07-media-processing.md#ablage-und-auslieferung).

## Löschen

**Benutzer löschen nur soft.** Endgültig löscht erst der Papierkorb:

```
delete()   → deleted = 1, Dateien nach var/deleted/     (restore() macht das rückgängig)
purge()    → Zeile entfernen, Dateien endgültig löschen  (Zeitplaner-Aufgabe trash
                                                          nach admin.trash_retention_days)
```

Soft-Delete kennen `record`, `area` und `admin`; Tabellen ohne `deleted`-Spalte löschen
sofort endgültig. Jede Änderung läuft über **Einzelfunktionen**, die genau einen
Datensatz samt Hooks behandeln – `softDelete()`, `restoreOne()`, `finalDelete()`.
Orchestrierungen (`delete($id, $withChilds)`, `restore()`, `purge()`, die Kaskade von
Bereich zu Datensätzen, der Papierkorb) sammeln nur Ids. Kein Datensatz verschwindet an
seinen Hooks vorbei. Einzelheiten: [03](03-dataset-api.md#löschen-und-wiederherstellen).

* **Reihenfolge ist Pflicht:** Löschen Blätter zuerst, Wiederherstellen Eltern zuerst
  (`record.parent_id` hat `ON DELETE CASCADE`).
* **Vorgangs-Zeitstempel:** Alle Datensätze eines Vorgangs bekommen denselben
  `changed_at`; `restore()` holt nur diese zurück.
* **Dateien wandern mit:** `var/deleted/{field_id}_{record_id}_{version}.{ext}`; die
  `data_fil`-Zeilen bleiben bis zum endgültigen Löschen.
* **Große Vorgänge** (mehr als `admin.mass_delete_threshold` Datensätze) laufen über
  `MassDeletion` als Hintergrundprozess – derselbe Aufruf, kein zweiter Löschpfad.

## Zeitzonen

`Cms` stellt `system.timezone` ein, danach die Zeitzone des Benutzers
(`admin.settings.timezone`) bzw. der Inhaltssprache (`language.timezone`) – für PHP und
für die Datenbankverbindung. Umrechnung: `Cms::utcToUserTZ()`, Formatierung
`Cms::_formatDate()` / `_formatTime()` (ICU, proleptisch gregorianisch) in der Locale des
Benutzer.
