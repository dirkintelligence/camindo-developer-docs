# 11 – Rechte

Rechte im Backend hängen an ***Benutzergruppen*** (`group`). Jeder Benutzer gehört
genau einer Benutzergruppe. Die Frontend-Community hat ein eigenes Rechtemodell
([12](12-community.md)).

## Das System in Kürze

| Ebene | Gepflegt in | Recht |
|---|---|---|
| Oberfläche | Benutzergruppe → Formular | Backend, Kommandozeile (`cli`), PHP-API (`php`) – worüber sich Benutzer der Benutzergruppe anmelden dürfen |
| Modul | Benutzergruppe → Formular | Zugriff ja/nein je gelistetem Modul, darunter die **Zusatzrechte** des Moduls |
| Instanz | dito | zugewiesene Instanzen; nur ihre Bereiche erscheinen |
| Sprache | dito (instanzübergreifend) | kein Zugriff / lesen / lesen und schreiben – für Inhalte |
| Status | dito und Status → Formular („Setzen dürfen“) | welche Status die Benutzergruppe einem Datensatz geben darf (flacher Workflow); lesen dürfen alle alle Status. Neue Benutzergruppe: alle |
| Bereich | Benutzergruppe → Reiter „Bereichsrechte“ und Bereich → Reiter „Zugriff“ | Lesen · eigene Benutzergruppe: Schreiben/Löschen · andere Benutzergruppe: Schreiben/Löschen · „Erweitert“ (Owner ändern); Template-Bereiche nur Lesen/Schreiben; Ordner haben keine Rechte |

Benutzergruppe **1 (Superuser)** darf alles, hat keine Rechte-Zeilen und ist nicht bearbeitbar.
Verwaltung der Benutzergruppen, Bereichsrechte, XML-Import/-Export und einige Werkzeuge sind
Superusern vorbehalten.

Ableitungen, die überall gelten:

* Leserecht auf einen Bereich = Leserecht auf alle seine Datensätze; Schreiben setzt
  Lesen voraus.
* Die **Benutzergruppe eines Datensatzes** ist die Benutzergruppe seines Owners (`record.owner_id` →
  `admin.group_id`); danach gelten die Rechte „eigene Benutzergruppe“ oder „andere Benutzergruppe“.
* **Verschieben** braucht im Quellbereich Schreiben und Löschen, im Zielbereich Schreiben.
* **Ändern** braucht neben dem Schreibrecht das Recht, den **aktuellen Status** des
  Datensatzes zu setzen (`Access::recordWrite()`): Was die Benutzergruppe nicht in einen
  Status bringen darf, darf sie in diesem Status auch nicht mehr bearbeiten.
* **Verknüpfen** braucht nur Leserecht auf den Zielbereich. Ohne Leserecht auf
  mindestens einen Zielbereich wird ein Verknüpfungsfeld nicht gezeigt, beim Speichern
  nicht angefasst, und der Datensatz lässt sich weder verschieben noch kopieren.
* Eine neu zugewiesene Instanz gibt der Benutzergruppe alle ihre Bereiche (volle Rechte ohne
  „Erweitert“) und Sprachen (lesen) frei; ein neu angelegter Bereich bekommt die Rechte
  der beim Anlegen gewählten Benutzergruppen, eine Bereichskopie die des Originals.

## Datenmodell

```
group                 group_id, label, details JSON {"modules": {"Record": ["batch"], "Tools": []},
                                                    "interfaces": ["backend", "php"]}, sig
rel_group_instance    group_id, instance_id                                     (Zeile = Zugriff)
rel_group_area        group_id, area_id, group_write, group_delete,
                      other_write, other_delete, set_owner, sig                 (Zeile = Leserecht)
rel_group_language    group_id, language_id, can_write, sig                     (Zeile = Leserecht)
rel_group_status      group_id, status_id, sig                                  (Zeile = darf setzen)
record.owner_id       Besitzer im Backend (FK admin, ON DELETE SET NULL)
```

`group.details.modules`: Schlüssel vorhanden = Modulzugriff, Werte = gewährte
Zusatzrechte. `group.details.interfaces`: erlaubte ***Oberflächen*** aus
`Access::INTERFACES` (`backend`, `cli`, `php`); fehlt der Schlüssel, nur `backend`. Alle Rechte-Zeilen sind **signiert** – eine per SQL eingefügte oder
geänderte Zeile gilt als nicht vorhanden ([13](13-security-authentication.md#signaturen)).

Sprachrechte nimmt `Group::save()` je Sprache als Stufe (`Group::LANGUAGE_NONE|READ|WRITE`)
oder als Liste davon – so schickt sie die Gruppenmaske (Hiddenfeld `0` plus Checkboxes
„lesen“ `1` und „schreiben“ `2`, `data[languages][{id}][]`); es gilt die höchste. Auch
abgewählte Sprachen kommen so an, damit `Group::grantInstance()` sie bei einer neu
zugewiesenen Instanz nicht wieder mit Leserecht vorbelegt.

Eine Gruppe löscht `Group::deleteReplacing($groupId, $replacementId)`: die aktiven
Benutzer der Gruppe wechseln per `Admin::save(['group_id'])` (Signatur, Audit-Log) in die
Ersatzgruppe, dann wird die Gruppe gelöscht; alles in einer Transaktion. Die
Superuser-Gruppe ist als Ersatz ausgeschlossen. `Group::hookOnDelete()` weist eine Gruppe
mit aktiven Benutzern weiterhin ab; ihre **gelöschten** Benutzer verlieren dort die
Zuordnung (`admin.group_id` = NULL, nullable; der Fremdschlüssel verweist mit
`ON DELETE RESTRICT`), neu signiert, mit Eintrag im Audit-Log des Kontos – gelöschte
Benutzer lassen sich nicht wiederherstellen, ihre Gruppe ist nur Historie.

## Laufzeit: Gruppendatei und `Access`

`InstanceCache` schreibt je Benutzergruppe eine Datei `var/cache/group-{id}.php` (sichtbare
Instanzen, lesbare Bereiche, Modul-, Bereichs-, Sprach- und Statusrechte). `Session::load()`
lädt sie je Request in `Camindo\Core\Access` – danach sind alle Prüfungen
Array-Zugriffe. Außerdem setzt die Session für Nicht-Superuser
`Cms::setInstanceAccess()` und `Cms::setAreaAccess()`: **jede** Abfrage über
`Cms::select()` auf `area`, `record`, `template` ist damit automatisch auf die Instanzen
und lesbaren Bereiche der Benutzergruppe beschränkt – Listen, Suche, Lookups, Exporte.

```php
$access = $backend->access();            // bzw. $app->get(Access::class)

$access->isSuperuser();
$access->module('Record');               $access->moduleRight('Record', 'batch');
$access->instanceIds();                  $access->instance($instanceId);
$access->areaRead($areaId);              $access->readableAreaIds(['record']);
$access->areaWrite($areaId, $ownerId);   $access->areaDelete($areaId, $ownerId);
$access->areaCreate($areaId);            $access->areaSetOwner($areaId);
$access->templateRead($areaId);          $access->templateWrite($areaId);
$access->languageRead($languageId);      $access->writableLanguageIds();
$access->statusWrite($statusId);         $access->settableStatusIds();
$access->recordWrite($areaId, $ownerId, $statusId);   // areaWrite() + statusWrite()
$access->interface('php');               $access->groupInterface($groupId, 'cli');   // ohne Laden
$access->script('Tools', 'index_developer.php');
$access->denied();                       // Meldung „Zugriff verweigert“ und Ende
echo $access->deniedHtml();              // nur die Meldung
```

### Oberflächen

Die API ist eine weitere Oberfläche zum Backend: dieselben Benutzer, dieselben Rechte ihrer
Benutzergruppe ([16](16-api.md)). Welche Oberfläche eine Benutzergruppe benutzen darf, prüfen die
Anmeldung (`Admin::loginByName()`, `ApiSession::runAs()`) und jedes Laden einer Session
(`Session::load()`): Wird einer Benutzergruppe eine Oberfläche entzogen, enden ihre laufenden
Sessions dort beim nächsten Request. `Access::interface()` antwortet – anders als die
übrigen Methoden – ungeladen mit Nein. Superuser dürfen alle Oberflächen.

Ohne geladene Rechte (Frontend, Cron ohne Benutzer) ist `Access` **ungeladen und verweigert
nichts** – das Frontend prüft über `RecordUser`, Cron läuft als System. Hintergrundjobs,
die ein Benutzer im Backend auslöst (Import, Export, Stapelverarbeitung, Massenlöschung),
laden seine Rechte (`Session::setCliAdmin($id, withAccess: true)`); sonst ließe sich
jede Prüfung über „als Hintergrundprozess“ umgehen.

Rebuild der Gruppendateien: bei jeder Änderung an Benutzergruppen, Instanzen, Sprachen,
Bereichen und Rechten automatisch; von Hand `InstanceCache::rebuild()` bzw. Werkzeuge →
Entwickler.

## Durchsetzung in zwei Schichten

1. **Oberfläche** – Listen, Masken, Buttons, Bäume und Auswahllisten zeigen nur, was
   erlaubt ist (`Backend::access()`). Nur lesbare Sprachen erscheinen als Reiter; eine
   Sprache ohne Schreibrecht zeigt den Wert als grauen Text statt als Eingabefeld und
   steht nicht in `shown` – ihre Werte bleiben beim Speichern unangetastet.
2. **Datenebene** – `Dataset::accessCheck()` in `save()`, `softDelete()`, `restore()`,
   `purge()`, `checkOut()`, `tmpInit()`, `movePrepare()` verweigert unabhängig davon mit
   `error_access_denied`. Geprüft werden `record` und `template`; die übrigen Tabellen
   (Admin, Field, Area …) schützt der Modulzugriff. Sprachrechte prüft `save()` bewusst
   nicht – das erledigt der `shown`-Mechanismus der Maske. Den **Status** prüft
   `save()` bei `record`: Bleibt `status_id` unverändert, ist nichts zu beachten; ändert
   er sich, muss die Benutzergruppe den neuen Status setzen dürfen
   (`error_status_denied`). Den Bearbeiten-Zugang zu einem Datensatz in einem nicht
   setzbaren Status sperren `RecordService::loadForAction('edit')` (Maske,
   Stapelverarbeitung, API) und die Oberfläche (Schaltfläche ausgegraut, Kontextmenü ohne
   „Bearbeiten“).

Dazu prüft `Backend::backendPath()` für **jedes** Script `Access::script()`:
Modulzugriff, Zusatzrecht des Scripts (`script_access`), Ausnahmen (`shared`,
`personal`). Eine Verweigerung erscheint als Meldung im Layer.

## Zusatzrechte

Ein gelistetes Modul definiert Rechte, die eine Benutzergruppe neben dem Modulzugriff einzeln
bekommt – in `install/settings.php`:

```php
'access' => [                          // interner Name [a-z0-9_]+ => Snippet der Beschriftung
    'publish' => 'blog.access_publish',
],
'script_access' => [                   // Script => nötiges Zusatzrecht
    'blog_publish.php' => 'publish',
],
'shared'   => ['blog_list.php'],       // darf von anderen Modulen als Lookup (src_field gesetzt) gerufen werden
'personal' => ['blog_prefs.php'],      // darf jeder Benutzer rufen, auch ohne Modulzugriff
'entries'  => ['index.php', 'index_archive.php'],   // Einstiege der Kachel, der erste erlaubte gewinnt
'superuser' => true,                   // Modul nur für Superuser
```

Snippet-Konvention: `{modul}.access_{recht}`. Im Code:
`$backend->access()->moduleRight('Blog', 'publish')`. Nach einer Änderung an
`settings.php`: `camindo.php refresh`.

Zusatzrechte im Core (Auszug):

| Modul | Rechte |
|---|---|
| `Record` | `import_csv`, `import_media`, `export_csv`, `import_docx`, `export_docx`, `batch`, `personal_data` |
| `Admin` | `admin_read_group`, `admin_read_other`, `admin_create`, `admin_change` |
| `Tools` | `server`, `database`, `data`, `developer` (je Reiter) |

## Owner

Jeder Datensatz gehört einem Benutzer (`record.owner_id`): beim Anlegen im Backend
dem Session-Benutzer, im Frontend dem Parameter `owner` von `CMS:USER_REGISTER` /
`CMS:USER_INPUT` bzw. `user.default_owner_id`. Ändern darf ihn nur, wer im Bereich
„Erweitert“ hat (`areaSetOwner()`) – und nur auf Benutzer der eigenen Benutzergruppe (Superuser:
alle); `Record::hookOnSave()` setzt sonst den bisherigen Wert zurück.

## Rechte testen

Benutzergruppe anlegen, einen Test-Benutzer in diese Benutzergruppe setzen, `Session::create($adminId)` –
danach lässt sich `Access` direkt abfragen oder per HTTP mit dem Cookie `sessid` gegen
`/backendModules/…` prüfen. Test-Benutzer und Session hinterher entfernen.
