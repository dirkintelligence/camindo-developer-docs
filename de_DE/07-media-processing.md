# 07 – Medien und Hintergrundprozesse

## Begriffe

| Begriff | Bedeutung |
|---|---|
| ***Prozessor*** | die Implementierung – eine Klasse, die Dateien umwandelt (`ImageGd`, `Magick`, `FFmpegVideo` …) |
| ***Konverter*** | eine **konfigurierte Instanz** eines Prozessors: Zeile in `converter` |
| ***Thumbnail-Feld*** | ein `file`-Feld mit `display_type = thumbnail` und `parent_field_id` auf das Feld, aus dem es entsteht |

Ein Prozessor allein tut nichts. Erst der Konverter legt fest, auf welche Medientypen er
anspringt und mit welchen Einstellungen; ein Thumbnail-Feld verweist auf einen oder
mehrere Konverter. Prozessoren laufen nur, wenn ein Datensatz mit einer neuen Datei
gespeichert wird (oder über Werkzeuge → Daten „Thumbnails neu erzeugen").

## Core-Prozessoren

| Prozessor | Typ | Grundlage |
|---|---|---|
| `ImageGd` | image | PHP-GD – einfache Bildbearbeitung ohne externe Programme, kein Hintergrundmodus |
| `Magick` | image | ImageMagick (`paths.magick`) – auch TIFF, PSD, HEIC, AVIF; immer unter eigener Sicherheits-Policy ([14](14-security-web.md#imagemagick)) |
| `FFmpegVideo` | video | ffmpeg – Videokonvertierung |
| `FFmpegAudio` | audio | ffmpeg – Audiokonvertierung |
| `FFmpegImage` | image | ffmpeg – Standbild aus einem Video |

## Einen Prozessor schreiben

```
Processors/MeinProc/               (Core: Core/Processors/…)
├── MeinProc.php                   extends Camindo\Core\MediaProcessor
├── install/settings.php           version, type (image|audio|video|document|other), label, description, requires
├── lang/{code}.ini                Präfix proc.meinproc
└── backend/
    ├── form.php                   Einschub in die Konverter-Maske
    ├── info.php                   Einschub in die Konverter-Detailansicht
    ├── html/*.html                Präfix proc.meinproc
    └── lang/{code}.ini
```

Namespace: `Camindo\Processors\{Name}` (Zusatz) bzw. `Camindo\Core\Processors\{Name}`.
Ein Zusatzprozessor **überschreibt** einen gleichnamigen Core-Prozessor.

```php
namespace Camindo\Processors\MeinProc;

use Camindo\Core\MediaProcessor;

class MeinProc extends MediaProcessor
{
    protected function hookDefinition(): void
    {
        // Vorschläge für die Konverter-Maske
        $this->defaults['media_types_incl'] = "image/jpeg\nimage/png";
        $this->defaults['media_types_excl'] = '';

        // eigene Einstellungen – landen in converter.details (JSON)
        $this->schema = [
            'details.width' => ['type' => 'integer', 'label' => '#proc.meinproc.field_width',
                                'default' => '300', 'min' => 0, 'suffix' => 'px'],
        ];
    }

    public function convert(string $srcPath, array $srcData, string $destPath,
                            array $details, bool $forceBgproc): array
    {
        // …
        return ['size' => $size, 'width' => $w, 'height' => $h,
                'mime_type' => 'image/webp', 'extension' => 'webp'];
    }

    // Hooks des Konverter-Datensatzes werden durchgereicht:
    public function hookOnSave(array &$fields, array &$data, array &$previousData): void {}
    // hookOnCreate, hookOnLoaded, hookOnSaved, hookOnDelete ebenso
}
```

| Parameter von `convert()` | Bedeutung |
|---|---|
| `$srcPath`, `$destPath` | Quelle und Ziel, **relativ zu `paths.media_dir`** |
| `$srcData` | Metadaten der Quelle (`mime_type`, `extension`, `width`, `height` …) |
| `$details` | Einstellungen des Konverters |
| `$forceBgproc` | Ausführung als Hintergrundprozess erzwingen |

Rückgabe: die Daten der Zieldatei; im Fehlerfall `['size' => 0, 'errors' => '…']`;
`['unsupported' => true]`, wenn der Prozessor die Datei nicht verarbeiten kann – dann
kommt der nächste Konverter des Thumbnail-Felds an die Reihe. Zusätzliche Angaben zur
**Quelle** (`width`, `height`, `data`) werden übernommen.

Externe Programme nur über `Execute` mit einem **Argument-Array** aufrufen, nie über
eine Shell-Zeichenkette. Dateinamen im Kommando stammen aus dem CMS, nicht aus
Benutzereingaben.

Registrieren: `php Core/cron/camindo.php component install MeinProc --activate`. Die
Laufzeitliste `var/config/components/processors.php` enthält danach den Prozessor, die
Konverter-Maske bietet ihn an.

## Konverter

| Spalte | Bedeutung |
|---|---|
| `processor` | Name des Prozessors |
| `type` | `image`, `audio`, `video`, `document`, `other` |
| `media_types_incl` / `media_types_excl` | Muster, auf die der Konverter anspringt bzw. nicht (je Zeile eins) |
| `extension` | Endung der Zieldatei |
| `details` | JSON mit den Einstellungen des Prozessors |

Muster (`VirtualFieldsTrait::_matchMediaType()`): exakter Mime-Typ (`image/jpeg`),
Platzhalter (`image/*` – passt nicht auf SVG) oder Endung (`.pdf`).

Die Konverter eines Thumbnail-Felds sind sortiert (`rel_field_converter.position`); die
Thumbnail-Erzeugung nimmt den ersten, dessen Muster passt und der die Datei verarbeiten
kann. `Converter::settingsHash()` (Prozessor, Endung, Medientypen, `details`) erkennt
Konverter über Datenbankgrenzen hinweg wieder – der XML-Import ordnet damit zu.

`details.background_process = 1` schickt die Umwandlung in einen Hintergrundprozess.
`ImageGd` kennt keinen Hintergrundmodus: Entsteht ein Bild im Hintergrund (etwa per
`FFmpegImage`), lassen sich weitere Versionen davon nur mit einem hintergrundfähigen
Prozessor (`Magick`) erzeugen.

## Thumbnail-Erzeugung

```
Upload-Feld  (file, upload)
   └── Thumbnail-Feld A  (file, thumbnail, parent_field_id = Upload)
          └── Thumbnail-Feld B  (parent_field_id = A)      Kaskade möglich
```

Ablauf beim Speichern:

1. `secureUploads()` prüft die Datei (gesperrte Endungen, Mime-Typ passt zur Endung) und
   legt sie sofort unter der Id der Arbeitskopie ab – markiert als `temporary`,
   `thumbnails_pending`.
2. Beim Check-in benennt `promoteTmpFiles()` die Dateien auf die `record_id` um,
   schreibt `data_fil` und erhöht `version` (Cache-Brecher `?v=…`).
3. `writeDeferredThumbnails()` erzeugt die Thumbnails mit den endgültigen Pfaden –
   Bilder sofort, Videos wie konfiguriert im Hintergrund –, rekursiv über die Kaskade;
   jedes Feld höchstens einmal je Vorgang. Ist eine Stufe im Hintergrund entstanden,
   laufen alle folgenden ebenfalls im Hintergrund.

So übersteht ein Upload eine gescheiterte Prüfung, und die teure Umwandlung läuft nur
einmal.

## Ablage und Auslieferung

```
public/media/{instance_id}/.htaccess                  Domain-Sperre, nosniff, Sandbox für aktiven Inhalt
public/media/{instance_id}/{field_id}/{record_id}[_{language_id}].{ext}
public/media/{instance_id}/{field_id}/.htaccess       nur bei details.user_auth: Require all denied
public/media/default/                                 Platzhalterbilder
```

`MediaAccess` schreibt die `.htaccess`-Dateien (nach jeder Änderung im Modul Instance,
bei geschützten Feldern beim Speichern des Felds); wechselt ein Bereich die Instanz,
zieht `MediaAccess::moveAreaMedia()` die Verzeichnisse um. Die Domain-Sperre lässt nur
Requests über die Domains der Instanz und der Instanzen durch, die sie lesen dürfen. Sie
prüft den `Host`-Header und verhindert Fremdverlinkung – eine Zugriffskontrolle ist sie
nicht; die leistet `user_auth` ([12](12-community.md#geschützte-medien)).

Dieselbe `.htaccess` trägt die Regel für **sprechende Dateinamen**: `CMS:MEDIA` schreibt
den ursprünglichen Dateinamen mit Komma vor die Endung
(`/media/1/12/1977,strand-nord.jpg`, `Cms::mediaUriName()`); die Rewrite-Regel wirft den
Zusatz weg.

**Das Backend verlinkt nie direkt** ins Medienverzeichnis, sondern immer auf
`backend/file.php?file={field_id}/{record_id}[/{language_id}]&v={version}`
(`RecordBackendServerFile::filePreview()`): Umleitung auf die Datei, wenn sie produktiv,
ungeschützt und ein Web-Rasterbild ist, sonst Auslieferung durch das Script – nur mit
Leserecht auf den Bereich. Im Frontend liefert `public/file.php` (`MediaDelivery`)
geschützte Dateien und Downloads (`CMS:MEDIA auth=true`, `download="name"`).

Für nicht webfähige Dateien und Versionen, die gerade im Hintergrund entstehen, liefert
`Backend::defaultImg()` ein Platzhalterbild. Webfähig sind `image/jpeg`, `image/png`,
`image/gif`, `image/webp`, `image/bmp`.

Zu große Dateien für einen HTTP-Upload können nach `var/upload/` gelegt und in der
Maske ausgewählt werden.

## Hintergrundprozesse

### `Execute` – Prozesse starten und überwachen

Plattformunabhängig (POSIX und Windows):

```php
$execute->execute($command, $stdout, $stderr, $timeout);                // blockierend
$execute->executeBackground($command, $stdoutFile, $stderrFile, $error); // → PID
$execute->processState($pid, $token);
$execute->terminateProcess($pid, $force, $token);
$execute->tokenizeParameters($parameters);   // Zeichenkette → Argument-Array
```

`$command` ist immer ein Array; Argumente werden korrekt maskiert, eine Shell ist nicht
beteiligt.

### `BackgroundProcess` – Tabelle `bgproc`

```php
$bgproc->start($command, $type, $token, $stdoutFile, $stderrFile, $error);
$bgproc->setProgressByToken($token, $progress, $message);
$bgproc->setStatusByToken($token, $status);
$bgproc->list($adminId);          // gleicht gespeicherten mit tatsächlichem Zustand ab
$bgproc->abort($token);  $bgproc->remove($token);
$bgproc->lock($type);    $bgproc->unlock();
```

Status: `STATUS_FINISHED` 0, `STATUS_RUNNING` 1, `STATUS_TIMEOUT` 2, `STATUS_FAILED` 3.
Ein Hintergrundprozess meldet seinen Fortschritt über sein **Token** – ohne Session.
`abort()`/`remove()` wirken nur auf Prozesse aus `bgproc`, und `Execute::matchToken()`
prüft die Kommandozeile gegen das Token, damit eine wiederverwendete PID nicht den
falschen Prozess trifft. Das Kommando enthält den Platzhalter
`BackgroundProcess::TOKEN_PLACEHOLDER`, `start()` setzt das Token ein.

Im Backend fragt `cApp.polling` die Liste ab, solange etwas läuft, und zeigt ein
Abzeichen an der Sanduhr im Kopf.

Aus dem Backend gestartete Hintergrundaufgaben (Massenlöschung, Stapelverarbeitung,
Import, Export, Thumbnails) übergeben ihren Auftrag als JSON-Datei in `var/tmp/` und
laufen mit den Rechten des auslösenden Benutzers (`Session::setCliAdmin($id, withAccess:
true)`); das Ergebnis kommt als interne Nachricht.

## Cron

```
* * * * *  php /pfad/zum/projekt/Core/cron/cron.php
```

Das ist der **einzige** Cronjob.

`cron.php` startet `TriggerCron::run()`, eine **selbstsperrende Schleife**: eine
Sperrdatei verhindert parallele Durchläufe, nach knapp zehn Minuten endet sie und der
nächste Cron-Aufruf übernimmt. Im Wartungsmodus beendet sie sich sofort.

| Takt | Aufgabe |
|---|---|
| `tick2s()` | Trigger für Hintergrund-Umwandlungen (`trigger_ff.php`, `trigger_magick.php`) |
| `tick1m()` | Zeitplaner (`scheduler.php`), Template-Abgleich (`template_sync.php`), Herzschlag für die Betriebshinweise. Alles Periodische darüber hinaus – auch die Wartung – ist eine Aufgabe des Zeitplaners |

### Trigger

`Core/cron/TriggerMedia.php` ist die gemeinsame Basis aller Trigger. Je `bgproc.type`
gibt es **einen** Trigger (das Fortschrittsformat hängt am Werkzeug, nicht am
Prozessor): `TriggerFFmpeg` liest die `-progress`-Datei von ffmpeg, `TriggerMagick` die
`-monitor`-Ausgabe von ImageMagick. Je Eintrag: Fortschritt eintragen, überfällige
Prozesse beenden, bei Abschluss `MediaProcessor::rectifyParams()` (Metadaten der
fertigen Datei), `data_fil` nachtragen
(`VirtualFieldsTrait::promoteBackgroundThumbnails()`), darauf aufbauende Thumbnails
starten, Eintrag entfernen.

Welcher Prozessor die fertige Datei auslesen darf, steht als `proc` in
`bgproc.message` – dort abgelegt von `MediaProcessor::startBackgroundProcess()`, das
Prozessoren statt `BackgroundProcess::start()` benutzen.

Ein neuer Prozessor mit vorhandenem Fortschrittsformat braucht am Trigger nichts:
`BGPROC_TYPE` passend setzen, `rectifyParams()` implementieren, mit
`startBackgroundProcess()` starten. Ein neues Format braucht eine Kindklasse von
`TriggerMedia` (`BGPROC_TYPE`, `PROGRESS_PREFIX`, `DEFAULT_PROCESSOR`,
`readProgress()` → `['percent' => int, 'finished' => bool]`), ein CLI-Script nach dem
Muster von `trigger_ff.php` und einen Aufruf in `TriggerCron::tick2s()`.

## Zeitplaner (Modul Scheduler)

Zeitgesteuerte Aufgaben in der Tabelle `scheduler`:

| Typ | Aufgabe |
|---|---|
| `url` | Adresse abrufen (optional mit HTTP-Basic-Auth); Ausgabe nach `var/custom/html/` |
| `php` | ein Script aus `var/custom/php/` auf der Kommandozeile ausführen |
| `dbbackup` | `mysqldump` nach `var/dbbackup/` mit Rotation (`Backup`) |
| `logcleanup` | Logdateien und Anmeldeversuche nach Fristen aufräumen |
| `template` | ein Frontend-Template mit `template.type = 'scheduler'` ausführen (`Page::renderScheduled()`) |
| `fulltext` | Volltext der Bereiche `details.area_ids` (leer = alle außer Ordnern) neu aufbauen (`RecordService::rebuildFulltext()`); bricht bei `max_minutes` ab |
| `cachecleanup` | Ausgabe-Cache von `CMS:INCLUDE` aufräumen (`IncludeCache::cleanup()`) |
| `trash` | Papierkorb leeren (`ToolDatabase::purgeTrash()`, `admin.trash_retention_days`), Sperre `trash`, Audit-Herkunft `trash` |
| `tmpcleanup` | `var/tmp` (`admin.tmp_retention_hours`; `*.lock`, `*.stamp` bleiben) und `RecordExport::cleanup()` |
| `privacy` | `PrivacyJobs::runDue()` in Runden zu `RUN_LIMIT`, solange eine Runde voll und fehlerfrei war |
| `retention` | `IpAnonymizer::run()`, `AuditLog::purge()`, `Message::purgeExpired()`, `RecordUser::purgeUnconfirmed()` – jeder Schritt für sich |
| `auditverify` | `AuditLog::verifyAll()` + `storeVerifyResult()` (für `SystemHealth`); Befunde = Fehler |
| `updatecheck` | `UpdateCheck::run()`; ohne `paths.composer` nichts zu tun |
| `messagedigest` | `MessageDigest::run()`: Zusammenfassung neuer Nachrichten per E-Mail an Benutzer mit `settings.message_digest = 1`; neu = seit dem vorigen Lauf (`last_run` vor dem Start) |

Die sechs Typen von `trash` bis `updatecheck` (`Scheduler::MAINTENANCE_TYPES`) haben keine `details`. Der Installer legt
`Scheduler::DEFAULT_TASKS` an (`Scheduler::createDefaults()`, Schritt `tasks` nach
`runtime`, weil `adminError()` die Gruppendateien braucht): nachts zwischen 2 und 4 Uhr
verteilt, `tmpcleanup` stündlich, `messagedigest` um 7 Uhr, Beschriftung als Snippet `#scheduler.label_type_{typ}`.
`camindo.php scheduler defaults` ergänzt fehlende Typen, `scheduler list` und
`scheduler run ID|TYP` dienen Betrieb und Test. `SystemHealth` meldet
Wartungstypen ohne eingeschaltete Aufgabe (Prüfung `maintenance`).

Die Zeitsteuerung steht in Cron-Syntax (Minute, Stunde, Tag, Monat, Jahr, Wochentag).
`TriggerCron::tick1m()` startet `cron/scheduler.php`; `SchedulerRunner::dispatch()`
startet jede fällige Aufgabe als eigenen Prozess und hält Ergebnis und Meldung in
`last_*` fest.

**Ausgaben und Scripte getrennt.** Ausgaben von `url`, `php` und `template` schreibt
der Runner nur nach `var/custom/html/` (`Scheduler::outputPath()`, `outputDir()`),
Scripte des Typs `php` kommen nur aus `var/custom/php/` (`Scheduler::scriptFile()`,
`scriptDir()`). Eine abgeholte Antwort kann so nie als Script ausgeführt werden;
Ausgabedateien mit PHP-Endung (`php*`, `phtml`, `phar`, `pht`, `phps`) sind zusätzlich
gesperrt. `Scheduler::cleanPath()` verwirft `.`/`..`-Segmente und ungewöhnliche Zeichen;
beim Ausführen bzw. Schreiben prüft der Runner zusätzlich den per `realpath()`
aufgelösten Pfad, damit kein Symlink aus dem Verzeichnis führt, und schreibt nicht in
eine bestehende Verknüpfung.

**Unter welchem Benutzer.** Jede Aufgabe hat einen Benutzer (`scheduler.admin_id`, Vorgabe der
vom Installer angelegte Benutzer `scheduler`, `Installer::createSchedulerAdmin()`).
`SchedulerRunner::run()` öffnet für ihn eine eigene Session – `ApiSession::runAs()`,
Oberfläche `cli`, Kanal `scheduler` ([16](16-api.md)) – und schließt sie danach. Es
gelten seine Gruppenrechte, das Audit-Log zeigt ihn mit der Herkunft `scheduler`. Ein
`php`-Script bekommt die Session als Umgebungsvariable `CAMINDO_API_SESSION`:

```php
require '/pfad/zum/projekt/load_camindo.php';
$app->get(\Camindo\Core\Api\ApiSession::class)->resume(getenv('CAMINDO_API_SESSION'), 'scheduler');
$result = $app->get(\Camindo\Core\Api\Api::class)->call('record.search', ['area_id' => 8]);
```

Ein Template läuft im selben Prozess in dieser Session. Einem `url`-Aufruf wird die
Session nie mitgegeben. Ohne gültigen Benutzer (gelöscht, gesperrt, Benutzergruppe ohne `cli`,
`Scheduler::adminError()`) scheitert der Lauf.

**Typ `template`.** `Page::renderScheduled()` stellt die Umgebung eines Seitenaufrufs
ohne Datensatz her: Instanz des Templates samt lesbaren Instanzen
(`Cms::setInstanceAccess()`), Inhaltssprache aus `details.language_id` (0 =
Hauptsprache) mit Zeitzone, Snippet-Sprache und Locale. `$set['last_run']` /
`$set['this_run']` (Zeitzone der Sprache, zusätzlich `_utc`) bilden das Zeitfenster
`(last_run, this_run]`. Sein Ende steht nach jedem **erfolgreichen** Lauf in
`details.window_end` (UTC) – `last_run` der Tabelle taugt dafür nicht, `dispatch()`
überschreibt ihn schon beim Start. Ein Lauf mit Meldungen in `Cms::$errors` gilt als
fehlgeschlagen; das Fenster bleibt dann stehen. Schutz: `Page` beantwortet ein
Template mit `type = 'scheduler'` als Adresse (`/0,name`) immer mit 404, und
`Template::saveAccess()` setzt für solche Templates `standalone = 0`.

## Externe Programme konfigurieren

```php
'paths' => [
    'magick'    => '/usr/bin/magick',     // ImageMagick 6: convert
    'ffmpeg'    => '/usr/bin/ffmpeg',
    'php'       => '/usr/local/bin/php',  // startet die Trigger
    'mysqldump' => '/usr/bin/mysqldump',
],
```

`TriggerCron` prüft `is_executable(paths.php)` und bricht mit einer Meldung ab, wenn der
Pfad nicht stimmt. Der Installer sucht die Programme und schlägt die Pfade vor.
