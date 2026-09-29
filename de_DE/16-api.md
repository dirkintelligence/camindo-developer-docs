# 16 – API

Die API ist eine **weitere Oberfläche** zum Backend – für Programme, die Kommandozeile,
den Zeitplaner, Custom Tools und einen LLM-Chat. Es gibt dafür keine eigenen Benutzer
und kein eigenes Rechtemodell: Angemeldet ist immer ein Administrator, und es gelten
die Rechte seiner Benutzergruppe ([11](11-permissions.md)). Was die API tut, erscheint
im Audit-Log als Handlung dieses Administrators.

Klassen: `Camindo\Core\Api\` (Verzeichnis `Core/src/Api/`).

| Klasse | Aufgabe |
|---|---|
| `ApiSession` | anmelden, mitarbeiten, wieder aufnehmen, abmelden, Auskunft |
| `Api` | Methoden finden, aufrufen, beschreiben |
| `ApiContext` | Oberfläche und Kanal des Prozesses, Prüfung je Aufruf, Fehler- und Sprachumrechnung |
| `ApiMethod` | Attribut, das eine Methode aufrufbar macht |
| `ApiException` | Fehler mit maschinenlesbarem Code |
| `ApiValues` | Feldwerte im neutralen Format |
| `SystemApi`, `RecordApi`, `AreaApi` | die Methoden (siehe unten) |

## Oberflächen und Kanäle

Eine ***Oberfläche*** ist der Weg, über den sich ein Admin anmeldet: `backend`, `cli`
(Kommandozeile), `php` (PHP-API). Welche davon eine Gruppe benutzen darf, legt der
Superuser in der Gruppe fest (`group.details.interfaces`, ohne Angabe nur `backend`).
Jede Session gehört zu genau einer Oberfläche ([13](13-security-authentication.md#sessions-und-cookies)).

Der ***Kanal*** sagt, wer über die Session arbeitet. Er steht im Audit-Log als Herkunft
(`AuditLog::setContext()`); Handelnder bleibt der Admin.

| Kanal | Wer |
|---|---|
| `llm` | ein LLM im Chat des Backends, in der Session des Admins |
| `tool` | ein Custom Tool, das aus einem Backend-Request heraus läuft |
| `scheduler` | eine Aufgabe des Zeitplaners |
| `cli`, `php` | die Oberfläche selbst |

## Sessions

| Methode | Oberfläche | Session |
|---|---|---|
| `login($name, $password, $maxLifetime, 'php')` | `php` | eigene; bei zweitem Faktor zuerst `ApiException` `second_factor` mit `uid`, dann `loginCode($uid, $code, …)` |
| `runAs($admin, $channel, $maxLifetime)` | `cli` | eigene, **ohne Passwort** – nur mit `PHP_SAPI === 'cli'`: wer die Kommandozeile des Servers hat, ist ohnehin vertrauenswürdig |
| `attach($channel)` | `backend` | keine neue: arbeitet in der geladenen Backend-Session mit (LLM-Chat, Custom Tool) |
| `resume($sessionId, $channel)` | `cli`, `php` | nimmt eine bestehende Session in einem weiteren Prozess oder Request wieder auf |
| `logout()` | – | eigene Session: Arbeitskopien verwerfen, Zeile löschen; nach `attach()` nur die Mitarbeit beenden |
| `whoami()` | – | Admin, Gruppe, Oberfläche, Kanal, Modulrechte, Instanzen mit Sprachen (Codes) und Lese-/Schreibrecht |

Eigene Sessions (`cli`, `php`) verfallen nach `api.max_inactive_sec` Untätigkeit und
spätestens zur Höchstlaufzeit (`$maxLifetime`, begrenzt durch `api.max_lifetime`). Sie
haben kein Cookie und kein Anmeldegerät. Passwort erneuern und TOTP einrichten gehen nur
im Backend – bis dahin weist `login()` ab. Passkeys lassen sich über die API nicht
bestätigen.

Mit `attach()` sieht die API genau, was der Admin sieht, und hinterlässt seine Spuren.
Nur dann gibt es Vorschau-Links – sie hängen an einer Backend-Session
([14](14-security-web.md#feste-backend-domain)).

## Aufrufen

```php
use Camindo\Core\Api\{Api, ApiSession};

$app->get(ApiSession::class)->resume($sessionId);
$result = $app->get(Api::class)->call('system.whoami');

// ['ok' => true,  'data' => [...], 'warnings' => []]
// ['ok' => false, 'error' => 'invalid_param', 'message' => '…', 'errors' => [...], 'warnings' => []]
```

`call()` prüft vor jedem Aufruf Session, geladene Rechte und Oberfläche
(`ApiContext::require()`) – ohne sie wäre `Access` ungeladen und verweigerte nichts.
Parameter sind die der PHP-Methode in snake_case (`$areaId` → `area_id`); unbekannte,
fehlende oder unpassend typisierte Parameter sind ein Fehler. Werte werden nur
verlustfrei umgewandelt (`"12"` → `12` bei `int`, `"true"` → `true` bei `bool`).
Unerwartete Fehler landen in `var/log/{Datum}_api-errors.log`; die Antwort nennt dann
nur `internal`. Eine offen gebliebene Transaktion wird zurückgerollt.

| Fehlercode | Bedeutung |
|---|---|
| `not_logged_in` | keine gültige Session, oder die Gruppe darf die Oberfläche nicht (mehr) benutzen |
| `second_factor` | Anmeldung braucht noch den Code |
| `access_denied` | Rechte der Gruppe reichen nicht |
| `not_found` | Objekt nicht gefunden |
| `locked` | ein anderer Admin bearbeitet den Datensatz |
| `open_in_backend` | der Admin hat den Datensatz selbst noch in einer Eingabemaske offen |
| `validation` | Eingabeprüfung gescheitert – Einzelheiten in `errors` |
| `invalid_param` | unbekannte Methode oder Parameter |
| `internal` | unerwarteter Fehler |

`errors` ist eine Liste aus `field`, `language` (Code), `label` und `message`
(`ApiContext::errorsFromCms()` macht sie aus `Cms::$errors`; Schlüssel `feld.{language_id}`
werden zu Feld und Sprachcode).

`system.describe` liefert je Methode ein **JSON-Schema** der Parameter mit den
Beschreibungen aus den `@param`-Zeilen – brauchbar als Werkzeug-Definition für ein LLM.

## Methoden

`system.describe` liefert zu jeder Methode alle Parameter mit Beschreibung; hier der
Überblick.

| Methode | Parameter (Auswahl) | Ergebnis |
|---|---|---|
| `system.whoami` | – | Admin, Gruppe, Oberfläche, Kanal, Modulrechte, Instanzen mit Sprachen |
| `system.methods`, `system.describe` | `method` | Verzeichnis, JSON-Schema |
| `area.list` | `instance_id`, `types` | lesbare Inhaltsbereiche mit Anzahl Datensätze und Rechten der Gruppe |
| `area.info` | `area_id` | Instanz, Sprachen, Default- und Sortierfeld, Baum, Slug-Präfix, Rechte |
| `area.fields` | `area_id`, `with_meta` | Eingabefelder: Typ, Mehrsprachigkeit, Pflicht, Grenzen, Auswahlwerte, Zielbereiche |
| `record.search` | `area_id` (eine Id oder Liste, 0 = global), `filter`, `filterop`, `filtermode`, `sort`, `order`, `skip`, `limit`, `url`, `language`, `relation` + `src_id`/`dest_id` | Trefferliste wie die Backend-Liste: `columns`, `rows` (Werte als Text), `filter_text`, `paging` (`total`, `next_skip`) |
| `record.info` | `record_id`, `tab` (`content`, `meta`, `history`, `fulltext`), `language` (Code, `all`), `fields`, `truncate` | Kopf (Bereich, Status, Zeiten, Sperre, `open_in_backend`, Rechte, Sprachen) und die Felder im neutralen Format; gekürzte Texte mit `truncated`, Verknüpfungen mit `total`/`more` |
| `record.field_value` | `record_id`, `field`, `language`, `skip`, `limit` | voller Inhalt eines Feldes, Verknüpfungen seitenweise, eingebettete Inhalte (`link`) mit ihren Feldern |
| `record.update` | `record_id`, `data`, `fields`, `relations`, `files`, `parent_id`, `previous_id`, `owner_id`, `unlock`, `simulate` | Auschecken, Ändern, Einchecken in einem Schritt; `record_id`, `simulated`, `record` (Kopfdaten) |
| `record.create` | `area_id`, `data`, `relations`, `files`, `parent_id`, `previous_id`, `owner_id`, `simulate` | neuer Datensatz (nicht in abhängigen Bereichen) |
| `record.copy` | `record_id`, `data`, `relations`, `files`, `simulate` | Kopie wie in der Maske (Dateien, eingebettete Inhalte, hinter dem Vorbild), mit Änderungen |
| `record.move_options` | `record_id`, `target_area_id` | je Quellfeld: Inhalt vorhanden, passende Zielfelder, Vorauswahl |
| `record.move` | `record_id`, `target_area_id`, `mapping`, `auto_map`, `unlock`, `simulate` | verwendete Zuordnung, `dropped` (Felder, deren Inhalt verworfen wird) |
| `record.delete` | `record_id`, `with_childs`, `unlock` | Papierkorb (`deleted`) bzw. Hintergrundprozess (`background`) |
| `record.restore` | `record_id` | aus dem Papierkorb zurück, wie im Modul Papierkorb |
| `record.message` | `record_id`, `to` (Id oder Name), `subject`, `body` | interne Nachricht, der Datensatz ist verlinkt |
| `record.preview_url` | `record_id`, `language` | Vorschau-Adresse – nur nach `attach()` |
| `record.batch` | `area_id`, `mode` (`edit`, `move`, `delete`, `fulltext`), Umfang: `ids` \| `filter`/`filterop`/`filtermode` \| `all`; `data`, `relations`, `files` (edit), `target_area_id`, `mapping`, `auto_map` (move), `simulate`, `background` (`auto`, `yes`, `no`) | `total`, `done`, `failed`, `errors` je Datensatz – bzw. `background.token`; Zusatzrecht „Stapelverarbeitung" |
| `record.export_fields` | `area_id`, `type` (`csv`, `docx`) | wählbare Feldschlüssel mit Vorauswahl und Unterfeldern |
| `record.export` | `area_id`, `type` (`csv`, `docx`, `xml`), Umfang `ids` bzw. `filter` (sonst alle), `fields`, `languages` (CSV: Codes; DOCX: [von, nach]), `options` (`header`, `bom`, `separator`, `enclosure`, `eol`; XML: `files`), `background` | `zip`, `path`, `size`, `records`, `valid_until` – bzw. `background.token`; Rechte wie in der Maske |
| `record.exports` | – | vorhandene Exportdateien |
| `system.processes`, `system.process`, `system.process_abort` | `token` | eigene Hintergrundprozesse: `status` (`running`, `finished`, `timeout`, `failed`), `progress`, `message` |

Die Parameter von `record.search` sind die von `Cms::select()` ([04](04-select-and-filter.md));
`sql`, `request` und `debug` gibt es nicht. Die Liste liefert die Werte in der
Kurzform der Backend-Liste (Text in Locale des Admins, gekürzt) – für die vollständigen
Werte im neutralen Format ist `record.info` da. Ohne `language` gilt die Sprache des
Backends (bevorzugte Sprachversion des Admins, sonst die Hauptsprache).

Große Mengen laufen wie im Backend als ***Hintergrundprozess*** – ab
`admin.mass_delete_threshold` (Stapel) bzw. `admin.export_background_threshold`
(Export) oder mit `background: yes`. Die Antwort nennt dann `background.token`; den Stand
liefert `system.process`, das Ergebnis kommt zusätzlich als interne Nachricht. Im
Audit-Log trägt ein Stapellauf den Kanal mit (`llm-batch`, `scheduler-batch` …).

### Schweregrad

Jede Methode trägt einen ***Schweregrad*** (`ApiMethod::LEVEL_*`, in `system.methods` und
`system.describe` als `level`): wie leicht sich ihre Wirkung rückgängig machen lässt.
Danach entscheidet der [Assistent](#assistent-zusatzmodul-camindomodule-ai), ob er vor dem Ausführen nachfragt.

| Stufe | Konstante | Methoden |
|---|---|---|
| 0 | `LEVEL_READ` | Lesen, Anlegen, Kopieren, Exportieren – alle nicht genannten |
| 1 | `LEVEL_CHANGE` | `record.update`, `record.restore`, `record.message`, `system.process_abort` |
| 2 | `LEVEL_DELETE` | `record.delete` (Papierkorb) |
| 3 | `LEVEL_MOVE` | `record.move` – nur über einen Umweg rückgängig zu machen |
| 4 | `LEVEL_MASS` | `record.batch` |
| 5 | `LEVEL_ADMIN` | administrative Änderungen (Bereiche, Felder, Status, Rechte) – noch keine Methode |

### Schreiben

Schreibende Methoden gehen denselben Weg wie die Eingabemaske (`RecordService`): Rechte
der Gruppe, Bearbeitungssperre eines anderen Admins (`locked`, mit `unlock` übergehbar
wie in der Maske), die eigene noch offene Maske (`open_in_backend` – die API verwirft
deren Arbeitskopie nicht), Validierung nach den Felddefinitionen, Check-in in einer
Transaktion. Mit `simulate` wird nur geprüft.

```json
{
  "record_id": 5508,
  "data":      {"title": {"en_EN": "Summer sale", "de_DE": "Sommerschlussverkauf"},
                "online_at": "2026-10-01T10:00:00+02:00", "price": "19.90"},
  "relations": {"tags": {"add": [3014], "remove": [3056]}, "categories": [2681]},
  "files":     {"image": {"serverfile": "fotos/sale.jpg"},
                "pdf":   {"content": "<Base64>", "filename": "flyer.pdf"},
                "old":   null}
}
```

* `data`: Werte im neutralen Format (`ApiValues::input()`), mehrsprachig als
  `{Code: Wert}` – nur Sprachen mit Schreibrecht; ein einzelner Wert gilt der
  Hauptsprache. Zeitpunkte mit oder ohne Offset (ohne: Zeitzone des Admins), Wandzeit
  ohne Offset, Dezimalpunkt, Beträge als `"19.90"`.
* `relations`: eine Liste ersetzt alle Verknüpfungen; `set`, `add`, `remove`, `order`
  (nur Felder mit manueller Sortierung) lassen sich kombinieren.
* `files`: `serverfile` (unter `var/upload`), `content` (Base64, höchstens
  `api.base64_max_bytes`), `tmpfile` (unter `var/tmp`), `null` entfernt; mehrsprachig
  `{Code: …}`. Eine Datei per URL holt die API bewusst nicht.
* Baum und Reihenfolge über `parent_id` und `previous_id`, der Owner über `owner_id`
  (Recht „Owner anpassen"). Eingebettete Inhalte (`link`) ändert man über ihre eigene
  `record_id`.

## Aus Templates und Custom Tools

```php
$result = $cms->api('record.search', ['area_id' => 8, 'filter' => ['status_id' => 1]]);
if ($result['ok']) foreach ($result['data']['rows'] as $row) { … }
```

`Cms::api()` nimmt die laufende API-Session, wenn es eine gibt (Zeitplaner, CLI,
PHP-API). Sonst arbeitet der Aufruf in der geladenen Backend-Session mit – Kanal
`tool`, danach wieder abgemeldet: ein Trigger-Template, das beim Speichern im Backend
läuft, handelt so als der Admin, der speichert. Ohne Admin-Session (Frontend) antwortet
die API mit `not_logged_in`. Ein Script des Zeitplaners (Typ `php`) nimmt die Session
aus `CAMINDO_API_SESSION` ([07](07-media-processing.md#zeitplaner-modul-scheduler)).

## Kommandozeile

```bash
php Core/cron/camindo.php api login --admin=NAME [--lifetime=SEK]    # gibt session_id aus
export CAMINDO_API_SESSION=…                                         # oder --session=…
php Core/cron/camindo.php api whoami
php Core/cron/camindo.php api methods
php Core/cron/camindo.php api describe system.describe
php Core/cron/camindo.php api call system.describe '{"method": "system.whoami"}'
php Core/cron/camindo.php api call METHODE --params-file=params.json
php Core/cron/camindo.php api logout
```

Ausgabe ist JSON; der Exit-Code ist 1, wenn die API einen Fehler meldet. Die Gruppe des
Admins braucht die Oberfläche „Kommandozeile".

## Eigene API-Methoden

Ein Modul nennt seine API-Klassen in `install/settings.php` (relativ zum Modul-Namespace)
und kennzeichnet die aufrufbaren Methoden mit dem Attribut:

```php
// Modules/Blog/install/settings.php
'api' => ['BlogApi'],
```

```php
namespace Camindo\Modules\Blog;

use Camindo\Core\Api\{ApiContext, ApiException, ApiMethod};

class BlogApi
{
    public function __construct(private Blog $blog, private ApiContext $context) {}

    /**
     * @param int $postId  Id des Beitrags
     */
    #[ApiMethod('blog.post', 'Einen Beitrag lesen.')]
    public function post(int $postId): array
    {
        $data = $this->blog->load($postId);
        if ($data === null) throw $this->context->exceptionFromCms(ApiException::NOT_FOUND);
        return ['post_id' => $postId, 'title' => $data['title'],
                'changed_at' => $this->context->isoTime($data['changed_at'])];
    }
}
```

Nach einer Änderung an `settings.php`: `camindo.php refresh`.

Regeln:

* **Keine eigene Logik neben der Oberfläche.** Was Backend und API gemeinsam brauchen,
  gehört in eine Service-Klasse ohne Request und Ausgabe – für Datensätze
  `RecordService` ([05](05-backend-development.md#aufbau-eines-moduls)). Die API-Klasse
  prüft Parameter und übersetzt Ergebnisse und Fehler.
* **Rechte wie im Backend** – über `Access` und die Dataset-Klassen; `Cms::select()`
  beschränkt die Abfragen ohnehin auf die Instanzen und Bereiche der Gruppe. Parameter
  für `select()` nur über eine Whitelist weiterreichen: `sql`, `request` und `debug`
  nie.
* **Fehler werfen** (`ApiException`), nicht zurückgeben; Hinweise über
  `ApiContext::warn()`.
* **Neutrale Werte:** Ganzzahlen ohne Trennzeichen, Dezimalzahlen mit Punkt, JSON als
  JSON, Text als Text. In UTC gespeicherte Zeitpunkte als ISO 8601 mit Offset in der
  Zeitzone des Admins (`ApiContext::isoTime()`), Wandzeit (`datetime`) ohne Offset.
  Sprachen als Codes (`de_DE`, `ApiContext::languageId()` / `languageCode()`).
* **Schweregrad angeben:** `#[ApiMethod('blog.publish', '…', write: true, level:
  ApiMethod::LEVEL_CHANGE)]`. Ohne Angabe gilt `LEVEL_READ` – der Assistent führt die
  Methode dann ohne Rückfrage aus.
* **Offene Masken beachten:** Wer einen Datensatz ohne Maske ändert, prüft mit
  `RecordService::loadForChange()` – bei einer noch offenen Maske desselben Admins
  (etwa im Chat mit einem LLM) antwortet die API mit `open_in_backend`, statt dessen
  Arbeitskopie zu verwerfen.

## Assistent (Zusatzmodul `camindo/module-ai`)

Das Zusatzmodul Ai ist ein LLM-Chat im Seitenpanel des Backends. Das Modell arbeitet
ausschließlich über die API: Jede Methode ist ein ***Werkzeug*** – Name mit `__` statt
Punkt (`record__search`), Schema aus `system.describe` –, aufgerufen nach
`ApiSession::attach('llm')` in der Backend-Session des Admins. Es sieht also nur, was
der Admin sieht, und jede Änderung steht im Audit-Log unter dessen Namen mit der
Herkunft `llm`. Ab Stufe 1 ([Schweregrad](#schweregrad)) führt der Assistent eine
Methode erst nach Bestätigung durch den Admin aus.

Für eigene API-Methoden heißt das: Sie stehen dem Assistenten automatisch zur
Verfügung – mit der Beschreibung aus `#[ApiMethod]` und den `@param`-Zeilen als
Anleitung. Ein fehlender oder zu niedriger Schweregrad lässt ihn ohne Rückfrage
handeln. Anbieter, Einstellungen und Aufbau beschreibt die Dokumentation des Moduls.

## Grenzen

* Frontend-Requests laden die Backend-Snippets nicht; Meldungen aus Backend-Snippets
  erscheinen dort als Schlüssel.
* Ohne Session gibt es keine API – auch nicht in Trigger-Templates, die ein
  Frontend-Benutzer auslöst (`CMS:USER_INPUT`).
