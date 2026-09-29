# 03 – Dataset-API

`Camindo\Core\Dataset` ist die abstrakte Tabellen-Abstraktion. Jede Tabelle eines
Moduls bekommt eine Kindklasse, die im Wesentlichen ein **Schema** und optional
**Hooks** festlegt. Validierung, Transaktionen, Rechteprüfung, Audit-Log, Papierkorb und
Bäume kommen aus der Basisklasse.

```php
namespace Camindo\Core\Modules\Status;

use Camindo\Core\Dataset;

class Status extends Dataset
{
    protected function hookDefinition(): void
    {
        $this->tableName        = 'status';
        $this->moduleName       = 'Status';
        $this->defaultSortOrder = 'position ASC';
        $this->defaultFieldName = 'label';
        $this->schema = [
            'status_id'   => ['type' => 'integer', 'label' => 'status_id', 'min' => 0],
            'label'       => ['type' => 'chars', 'label' => '#status.field_status_label',
                              'max' => 64, 'flags' => ['trim', 'not_empty']],
            'icon'        => ['type' => 'chars', 'label' => '#status.field_status_icon', 'max' => 64],
            'color_class' => ['type' => 'chars', 'label' => '#status.field_status_color', 'max' => 64],
            'position'    => ['type' => 'integer', 'label' => '#status.field_status_position', 'min' => 0],
        ];
    }
}
```

```php
$status = $app->get(Status::class);        // immer über den Container
$data = $status->init();                   // Vorgaben
$data['label'] = 'Freigegeben';
$id = $status->save(['label', 'icon', 'position'], $data);
if ($id === null) echo $cms->errorMsg();
```

Abhängigkeiten (`Config`, `Database`, `Crypt`, `Cms`, `System`, `Session` …) kommen per
Autowiring. Der Primärschlüssel heißt `{tableName}_id`.

## Schema

Ein Schema ist ein Array `feldname => spaltendefinition`:

```php
'name' => [
    'type'         => 'chars',
    'label'        => '#admin.field_admin_name',   // führendes # = Snippet
    'tooltip'      => '#admin.tooltip_field_name',
    'display_type' => 'text',
    'min'          => 3,       'min_op' => '>=',    // '>=' (Vorgabe) oder '>'
    'max'          => 64,      'max_op' => '<=',
    'pattern'      => '',
    'prefix'       => '',      'suffix' => '',
    'default'      => '',
    'flags'        => ['trim', 'not_empty', 'name', 'unique'],
    'table'        => 'admin', // bei Typ 'id': Zieltabelle
    'options'      => [],      // bei display_type 'select'/'radio'
    'extension'    => '',      // Extension für das Eingabefeld, z. B. 'Password'
],
```

Alle Schema-Felder sind ***Metafelder*** (`'meta' => 1` wird im Konstruktor gesetzt) –
im Gegensatz zu den virtuellen Feldern der Inhaltsbereiche, die `Record` zur Laufzeit
aus `field` ergänzt.

**Subfelder.** Ein Punkt im Namen bedeutet: der Wert liegt in einem JSON-Feld.

```php
'details.width'  => ['type' => 'integer', 'label' => '#proc.imagegd.field_width'],
```

`init()` erzeugt daraus `$data['details']['width']`. Ein Subfeld wird nur geprüft, wenn
das übergeordnete Feld (`details`) geschrieben wird.

**Felder ohne eigene Spalte: `reltable`.** Angaben, die in einer eigenen
Relationstabelle stehen (etwa die Sprachen einer Instanz), bekommen den Typ `reltable`.
Er verhält sich wie `json` (Wert ist ein Array, `min`/`max` zählen Einträge,
`not_empty` = mindestens ein Eintrag), wird von `save()` aber nicht als Spalte
geschrieben: Die Kindklasse schreibt ihn in `hookOnSaved()` und liest ihn in
`hookOnLoaded()`. `select()` kann danach weder filtern noch sortieren.

**Datumswerte.** Datumsspalten sind `NOT NULL`. „Kein Datum" ist
`Cms::DATE_MIN = '1000-01-01 00:00:00'`, „kein Ende" `Cms::DATE_MAX =
'9000-01-01 00:00:00'`; beide gibt `_formatDate()` als Leerstring aus. Kleinere Werte
werden abgewiesen, nicht stillschweigend angehoben. SQL-Funktionen (`NOW()`,
`UTC_TIMESTAMP()` – nur aus `Cms::$funcWhitelist`) sind von `min`/`max` ausgenommen.
`created_at`/`created_by` werden bei Änderungen weder geschrieben noch geprüft.

**Vorgaben (`default`).** Skalare Typen: der Wert. Array-Typen (`file`, `json`,
Relationen, `reltable`): **jede Zeile** ist eine wählbare Option. Datumstypen: `now`,
`today`, optional mit Versatz (`now+3d`, `today-2h`; Einheiten `s|m|h|d`).

## Schema-Kennzeichen

Der Konstruktor leitet aus dem Schema Kennzeichen ab und prüft die Konsistenz:

| Kennzeichen | Bedingung | Wirkung |
|---|---|---|
| `hasParentId` | `parent_id` im Schema | Baum möglich |
| `hasDeleted` | `deleted` im Schema | Soft-Delete |
| `hasTmp` | in `hookDefinition()` gesetzt | Arbeitskopien |
| `hasAreaId` | `area_id` im Schema | `unique_in_area` möglich |
| `hasChangedAt` / `hasChangedBy` | Schema-Feld oder `hasTmp` | Änderungs-Metadaten |

Harte Regeln (Exception bei Verstoß): `parent_id` ⇒ auch `parent_ids`, `lft`, `rgt`,
`position`, `depth`; `hasTmp` ⇒ `tmp` und `deleted` im Schema sowie die Spalten
`origin_id`, `changed_by`, `changed_at` in der Tabelle.

Weitere Eigenschaften, die eine Kindklasse setzen kann: `$sanitizeHtml = false`
(HTML-Filter beim Schreiben aus – nur `Template`), `$audit = false` (kein Audit-Log),
`$auditIgnore` (Felder, die nicht protokolliert werden), `auditSecret()` für umkehrbar
verschlüsselte Werte.

## Traits

```php
class Record extends Dataset {
    use TreeManagerTrait, VirtualFieldsTrait, TemporaryTrait, LinkTrait, MoveTrait;
}
```

| Trait | bringt |
|---|---|
| `TreeManagerTrait` | Nested Set + Adjacency List ([02](02-data-model.md#bäume-nested-set-und-adjacency-list)) |
| `VirtualFieldsTrait` | benutzerdefinierte Felder, Uploads, Thumbnails, Relationen, Volltext |
| `TemporaryTrait` | Arbeitskopien, Bearbeitungssperren |
| `LinkTrait` / `MoveTrait` | fest verknüpfte Inhalte / Verschieben zwischen Bereichen |

## Hooks

Alle Hooks sind `protected` und haben leere Vorgaben; nur `hookDefinition()` ist
abstrakt.

| Hook | Zeitpunkt |
|---|---|
| `hookDefinition()` | Konstruktor – Tabelle, Modul, Schema, Sortierung |
| `hookOnCreate(&$data)` | in `init()`, nach den Vorgaben |
| `hookOnLoaded(&$data)` | in `load()`, nach dem Lesen |
| `hookOnSave(&$fields, &$data, &$previousData)` | in `save()`, **vor** Prüfung und Schreiben |
| `hookOnSaved(&$fields, &$data, &$previousData)` | in `save()`, **nach** dem Schreiben, vor dem Commit |
| `hookOnSoftDelete($data)` | beim Markieren `deleted = 1` |
| `hookOnRestore($data)` | beim Wiederherstellen |
| `hookOnDelete($data)` | beim **endgültigen** Löschen, vor dem `DELETE` |

**Fehler in Hooks: Exception werfen.** Nur so greift der Rollback der umgebenden
Transaktion. Beginnt die Meldung mit `#`, gilt sie als bereits in `Cms::$errors`
registriert.

Weil `hookOnSave()` **vor** der Prüfung läuft, wird ein dort gesetzter Wert normal
validiert. Setzt der Hook ein Feld, das nicht in `$fields` steht, muss er es dort
eintragen, sonst wird es nicht geschrieben.

```php
// Template::hookOnSave() – Quelltext kompilieren
$result = $this->parse($data['code'], $data['area_id']);
$data['styled_code'] = $result['styledCode'];
$data['error_text']  = implode('<br>', (array)$result['errors']);

// Record::hookOnSave() – leeren Slug aus dem Default-Feld füllen
$data['slug'] = $this->cms->textToUri($titel);
```

Für Datensätze gibt es zusätzlich **Trigger-Templates** – Ereignis-Methoden, die ein
Administrator je Inhaltsbereich hinterlegt ([06](06-template-compiler.md#trigger-templates)).

## Lesen und Schreiben

### `init(array $defaults = []): ?array`

Vollständiges Daten-Array mit allen Vorgaben; `$defaults` überschreibt einzelne Werte
(Subfelder als Unter-Array). Bei `record` ist `$defaults['area_id']` Pflicht – erst der
Bereich bestimmt die virtuellen Felder.

### `load(string|int $value, string $fieldName = '', bool $forUpdate = false): ?array`

Lädt über den Primärschlüssel oder eine andere Spalte; `$forUpdate` setzt `FOR UPDATE`
(nur in einer Transaktion sinnvoll). Nicht gefunden → Meldung in `Cms::$errors`
(`{modul}.error_{table}_not_found`, sonst `error_record_not_found`), Rückgabe `null`.
Passwort-Hashes werden nie zurückgegeben.

### `save(array $fields, array $data, bool $stayTemporary = false): ?int`

* `$fields` = Namen der zu schreibenden Felder; **leeres Array = alle Felder**.
* `$data["{tableName}_id"]` entscheidet: 0 = INSERT, > 0 = UPDATE.
* Rückgabe: Id oder `null` bei Fehlern (Meldungen in `Cms::$errors`).
* Alles läuft in einer Transaktion; Audit-Eintrag und Rechteprüfung inklusive.
* Nicht übermittelte Felder bleiben bei einem UPDATE unverändert.

Bei Tabellen mit Arbeitskopien (`record`) gilt zusätzlich: gespeichert wird nur eine
**Arbeitskopie** des Session-Admins (vorher `checkOut()` bzw. `tmpInit()`);
`$stayTemporary = true` schreibt nur den Arbeitsstand (Reiterwechsel), `false` ist der
Check-in. Beim Check-in wird der **ganze** Datensatz geprüft – Request und Arbeitsstand
zusammen. Metafelder, die die Maske nicht zeigt, behalten ihren bisherigen Wert.

`Record::importSave()` ist die Variante für Importe und Skripte: Werte liegen bereits im
Datenbankformat vor (`$storedFormatFields`), Passwort-Hashes werden übernommen
(`$prehashedFields`), die Anzahlprüfung von Verknüpfungen lässt sich aufschieben.

### `transactional(callable $task, mixed $onError = null)`

Kapselt `beginTransaction`/`commit`/`rollback` – `protected`, für eigene
Mehrschritt-Vorgänge in der Kindklasse.

### `fetch()` / `select(array $param = [])`

Dünne Hüllen um `Cms::fetch()` bzw. `Cms::select($param, $this)`, siehe
[04](04-select-and-filter.md).

## Löschen und Wiederherstellen

Einzelfunktionen behandeln genau einen Datensatz und feuern dessen Hooks;
Orchestrierungen sammeln nur Ids:

| Einzelfunktion | Wirkung |
|---|---|
| `softDelete(int $id): bool` | `deleted = 1`, `hookOnSoftDelete()` |
| `restoreOne(int $id): bool` | `deleted = 0`, `hookOnRestore()` |
| `finalDelete(int $id): bool` | Zeile entfernen, `hookOnDelete()` |

| Orchestrierung | ruft |
|---|---|
| `delete(int $id, bool $withChilds = false)` | `softDelete()` – ohne `$withChilds` rücken Kinder eine Ebene hoch |
| `restore(int $id, bool $withChilds = true)` | `restoreOne()` – nur Datensätze mit dem Zeitstempel des Löschvorgangs |
| `purge(int $id, bool $withChilds = true)` | `finalDelete()` |

Tabellen ohne `deleted` leiten `delete()` direkt auf `purge()` um. Eine Orchestrierung
öffnet **eine** Transaktion für den ganzen Vorgang; eine direkt aufgerufene
Einzelfunktion ihre eigene (so kann der Papierkorb einen scheiternden Datensatz
überspringen). Wird eine Id einer Arbeitskopie übergeben, ist das Original gemeint.

**Kaskaden über Tabellengrenzen** reichen den Vorgangs-Zeitstempel durch:

```php
// Area::hookOnSoftDelete()
$record = $this->recordDataset();
$record->setMarkTimestamp($this->getMarkTimestamp());
foreach ($record->cascadeIds('area_id', $areaId, 'delete') as $recordId) {
    if (!$record->softDelete($recordId)) {
        throw new \Exception('#' . $this->cms->errorMsgLast());
    }
}
```

Damit eine Tabelle daran teilnehmen kann, müssen `changed_at` und `changed_by` in ihrem
Schema stehen. Große Vorgänge verlagert `MassDeletion` in einen Hintergrundprozess
(Schwelle `admin.mass_delete_threshold`); dort läuft derselbe Aufruf.

## Validierung

`validateValue()` läuft für jedes Feld, das tatsächlich geschrieben wird:

1. **Typisierung** – `bit` über `FILTER_VALIDATE_BOOLEAN` (`1/true/on/yes` bzw.
   `0/false/off/no/""`, sonst Fehler); Array-Typen als Array; sonst String.
2. **Aufbereitung** – `strip_tags`, `trim` (implizit bei `name` und Zahlentypen),
   Kleinschreibung bei `display_type = email`. Ohne `strip_tags` läuft jeder Textwert
   durch den HTML-Filter (`HtmlSanitizer`, [14](14-security-web.md#html-filter)).
   `password` ist ausgenommen.
3. **Flags** (Tabelle unten).
4. **Muster** – aus `name`, `display_type = email` oder `pattern`.
5. **Grenzen** `min`/`max` – bei Texten die Zeichenzahl, bei Zahlen der Wert, bei
   Array-Typen die Anzahl.
6. **Passwort-Richtlinie** – für Felder mit `display_type password` und der Extension
   `Password` ([13](13-security-authentication.md#passwort-richtlinie)).
7. **Eindeutigkeit** – `unique` / `unique_in_area`.

| Flag | Wirkung |
|---|---|
| `trim` | Leerraum am Rand entfernen |
| `strip_tags` | HTML entfernen |
| `not_empty` | Pflichtfeld; `"0"` gilt als gefüllt. Ein leeres Passwort beim Update behält das bisherige. Bei Upload-Feldern prüft der Check-in, ob eine Datei vorliegt |
| `not_zero` | nicht `0` |
| `name` | `trim` und Muster `^[a-z][a-z0-9_]+$` (mindestens zwei Zeichen) |
| `unique` | eindeutig innerhalb der Instanz samt der mit ihr verbundenen Instanzen (`InstanceCache::uniqueScope()`), bei Tabellen ohne `instance_id` tabellenweit |
| `unique_in_area` | eindeutig innerhalb des Bereichs |
| `ft` | fließt in die Volltextspalte ein |

Bei virtuellen Feldern prüft `uniqueVirtualOk()` gegen die `data_*`-Tabelle (produktive,
nicht gelöschte Datensätze); bei `encrypted` wird der HMAC verglichen. Relationen und
Dateien kennen keine Eindeutigkeit. Das Feld `slug` prüft statt dessen den ganzen
URL-Pfad samt Slug-Prefix (`Record::slugPathOk()`).

### Mehrsprachige Felder

Der Wert liegt als `language_id => eingabe` vor (`data[feld][{language_id}]` aus der
Maske). Die Hauptsprache wird vollständig geprüft, eine weitere Sprache nur, wenn dort
etwas eingegeben wurde – leer heißt „nicht übersetzt". Fehler werden unter
`{feld}.{language_id}` registriert; `unique` gilt je Sprache. Ein einzelner Skalar
(Import, CLI) gilt als Wert der Hauptsprache.

### Fehlermeldungen

```php
$cms->error('Meldung', 'feldname', 'Feld-Label');
$cms->errorsReset();
$cms->errorMsg('<br>');    // alle Meldungen
$cms->errorMsgLast();      // die letzte
$cms->getErrorFields();    // betroffene Felder (Hervorhebung im Formular)
$cms->hasErrors;
```

Standard-Snippets: `error_value_mandatory`, `error_invalid_format`,
`error_min_ge_length`, `error_max_le_value`, `error_unique_value`,
`error_record_not_found`, `error_access_denied` …; `{value}` wird mit dem Grenzwert
gefüllt.

### Datums- und Zeiteingaben

`dateInputToIso()` und `timeInputToIso()` verstehen unvollständige und lokale
Schreibweisen und füllen auf:

```
2023              → 2023-01-01 00:00:00   (bei > und =)   bzw. 2023-12-31 23:59:59 (bei <)
2023-06-15 14:30  → exakt
15.06.2023        → Schreibweise der Locale (CLDR-Muster über IntlDateFormatter)
```

Ein Muster gilt nur, wenn es die **ganze** Eingabe verarbeitet hat. Selbsttest:
`php Core/cron/check_dateparse.php`.

## Weitere Methoden

```php
$dataset->getSchema();            $dataset->getTableName();       $dataset->getModuleName();
$dataset->getDefaultSortOrder();  $dataset->getDefaultFieldName(); $dataset->hasTree();
$dataset->getBindType($type);     // mysqli-Bind-Typ 'i', 'd', 's', 'b'
```

In Kindklassen außerdem `areaInfo($areaId)` (Name, Typ, Instanz eines Bereichs aus dem
Dateicache statt aus der Tabelle `area`).
