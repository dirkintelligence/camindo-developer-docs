# 09 – Installation, Update und Kommandozeile

Das Handbuch für Administratoren beschreibt die Installation Schritt für Schritt. Dieses
Kapitel erklärt die Mechanik dahinter: Pakete, Versionen, Migrationen, die Abläufe von
Installer und Update und die Kommandozeile.

## Grundsatz: drei Ebenen, eine Richtung

Jede Installation, jedes Update und jede Deinstallation berührt drei Ebenen:

```
DATEIEN                   DATENBANK                     LAUFZEIT
Composer                  Migrationen                   generierte Dateien unter var/
Core/ Modules/ …          schema, seed, migrations/     var/config/components/*.php
vendor/                   Tabellen component, system    var/config/core.php, Caches
```

1. **Hinzufügen und Aktualisieren: Dateien → Datenbank → Laufzeit.** Composer legt
   Dateien hin; die Datenbank folgt in einem bewussten, eigenen Schritt; die
   Laufzeitdateien werden daraus abgeleitet. Nie automatisch aus einem Composer-Hook –
   der läuft auch auf Build-Servern ohne Datenbank.
2. **Entfernen: Laufzeit → Datenbank → Dateien.** Die Deinstallation braucht die Dateien
   der Komponente; also erst `component uninstall`, dann `composer remove`.
3. **Die Laufzeit fragt nie die Datenbank nach dem Zustand.** Was aktiv ist, welche
   Version der Core hat, ob Wartung läuft, steht in Dateien unter `var/`.
4. **Zwischen Dateien und Datenbank darf ein Fenster liegen, aber kein Absturz.** Eine
   Komponente mit wartendem Update ist inaktiv (Funktion fehlt, bricht nicht); für den
   Core gilt Wartungsmodus.

## Pakete

| Paket | Composer-Typ | Landet in | Inhalt |
|---|---|---|---|
| `camindo/project` | `project` | Projektwurzel | das Skelett: `composer.json`, `load_camindo.php`, `public/` mit Stubs, leere Verzeichnisse (Quelle: `Core/install/draft/`) |
| `camindo/cms` | `camindo-core` | `Core/` | das CMS |
| `camindo/composer-plugin` | `composer-plugin` | `vendor/` | bringt die camindo-Pakettypen an ihren Ort |
| `{vendor}/camindo-module-{name}` | `camindo-module` | `Modules/{Name}/` | Zusatzmodul |
| `{vendor}/camindo-processor-{name}` | `camindo-processor` | `Processors/{Name}/` | Zusatzprozessor |
| `{vendor}/camindo-extension-{name}` | `camindo-extension` | `Extensions/{Name}/` | Extension |

**Das Composer-Plugin** ist eine Unterklasse von Composers `LibraryInstaller`, die nur
den Zielpfad bestimmt (`Core/`, `Modules/{name}/` …). `{name}` kommt aus
`extra.installer-name` der Komponente (StudlyCase, Pflicht) – Composer erzwingt
kleingeschriebene Paketnamen, Verzeichnisse sind aber Klassennamen. Ohne gültigen
Eintrag verweigert das Plugin die Installation. Ein eigenes Plugin statt
`composer/installers`, weil jenes nur eine feste Liste von Framework-Typen kennt.

`composer.json` des Skeletts (gekürzt):

```json
{
    "name": "camindo/project",
    "type": "project",
    "require": { "php": ">=8.3", "camindo/composer-plugin": "^1.0", "camindo/cms": "^9.0" },
    "autoload": { "psr-4": { "Camindo\\": "" } },
    "config": { "allow-plugins": { "camindo/composer-plugin": true } },
    "scripts": {
        "post-create-project-cmd": "@php Core/cron/camindo.php install",
        "post-install-cmd":        "@php Core/cron/camindo.php status --after-composer",
        "post-update-cmd":         "@php Core/cron/camindo.php status --after-composer"
    }
}
```

`Camindo\Core\` bringt `Core/composer.json` mit; `Camindo\` → `./` deckt die
Zusatzkomponenten ab. Die Hooks sind **passiv**: `status --after-composer` schreibt
nichts, verträgt eine fehlende Datenbank und meldet nur, was wartet:

```
camindo 9.1.0 – core update pending (database: 9.0.3)   -> php Core/cron/camindo.php upgrade
Modules/Blog 1.2.0 – update pending (installed: 1.1.0)  -> php Core/cron/camindo.php component update Blog
```

**Composer schreibt nur in `Core/`, `Modules/`, `Processors/`, `Extensions/`, `vendor/`
und `composer.lock`.** Alles, was eine Installation individuell macht, liegt außerhalb –
deshalb darf der Docroot nie in `Core/` liegen.

### Entwicklung am Core

Wer den Core selbst entwickelt, hält `Core/` als Git-Checkout und lässt Composer ihn
nicht anfassen. Die `composer.json` des Entwicklungsprojekts erklärt per `replace`, dass
sie `camindo/cms` selbst bereitstellt (Vorlage `Core/install/draft/composer-dev.json`):

```json
{
    "name": "camindo/project-dev",
    "replace": { "camindo/cms": "9.x-dev" },
    "require": { "camindo/composer-plugin": "^1.0", "phpmailer/phpmailer": "^7.0", "…": "…" },
    "autoload": { "psr-4": { "Camindo\\": "", "Camindo\\Core\\": "Core/src/" } }
}
```

Zusatzkomponenten mit `camindo/cms: ^9.0` sind damit zufrieden. Eine Komponente in
Entwicklung liegt als eigener Checkout direkt in `Modules/{Name}/`; `component install`
arbeitet mit den Dateien, gleich woher sie kommen.

## Versionen

| Ort | Wert | Wer schreibt |
|---|---|---|
| `CAMINDO_VERSION` in `Core/Core.php` | `9.1.0` | Entwickler, beim Release |
| Git-Tag des Core-Repos | `v9.1.0` | Entwickler – Composer liest daraus die Paketversion |
| `system.core_version` + `var/config/core.php` | `9.1.0` | Installer und `upgrade`, nach erfolgreichen Migrationen |

`var/config/core.php` (`['version' => …, 'upgraded_at' => …]`) ist der
**Versionsstempel** für die Laufzeit: Weicht er von `CAMINDO_VERSION` ab, antwortet das
Frontend mit 503 und das Backend zeigt Superusern den Hinweis auf `upgrade` – neuer Code
erreicht das Frontend nie auf altem Schema. Fehlt der Stempel (und `var/install.lock`),
gilt die Installation als nicht vorhanden.

Für Zusatzkomponenten ist `install/settings.php['version']` maßgeblich; installiert steht
die Version in `component.version`, eine neuere Dateiversion in `update_version`.

## Schema und Migrationen

`Core/install/database/schema.sql` (Struktur) und `seed.sql` (Stammdaten) beschreiben
immer den **aktuellen** Stand; eine Neuinstallation braucht keine Migration.
Migrationen bringen bestehende Installationen dorthin. Regel für jede Schemaänderung im
Core:

1. Migration schreiben: `Core/install/migrations/{version}.sql` und/oder `.php`
2. `schema.sql`/`seed.sql` nachziehen – per `php Core/cron/camindo.php schema export`,
   nicht von Hand (nur mit `system.development_version`)
3. beides im selben Commit

Der Dateiname ist die Version, die die Migration **einführt**. Beim Update von `from`
auf `to` laufen alle Dateien mit `from < v <= to`, sortiert nach `version_compare()`,
`.sql` vor `.php` derselben Version. Komponenten haben dasselbe in
`install/migrations/`; Core-Komponenten haben keine eigenen – ihre Tabellen sind
Core-Tabellen.

**`.sql`** – Anweisungen, getrennt durch `;` am Zeilenende, einzeln ausgeführt (ein
Fehler nennt die Anweisung). `DELIMITER` wird nicht unterstützt.

**`.php`** – gibt eine Closure zurück:

```php
<?php
use Camindo\Core\Install\InstallContext;

return static function (InstallContext $ctx): void {
    if (!$ctx->columnExists('record', 'owner_id')) {
        $ctx->db->query("ALTER TABLE `record` ADD `owner_id` bigint UNSIGNED NULL");
    }
    $ctx->db->query("UPDATE `record` SET `owner_id` = `created_by` WHERE `owner_id` IS NULL");
    $ctx->log('owner_id filled');
};
```

Regeln:

* **Ein Zeiger, kein Protokoll.** Der erreichte Stand (`system.core_version` bzw.
  `component.version`) wird nach **jeder** Version fortgeschrieben; nach einem Abbruch
  beginnt der nächste Lauf dort. Das Log steht in `var/log/{Datum}_upgrade.log`.
* **Idempotent, wo es billig ist** (`IF NOT EXISTS`, `columnExists()`). DDL ist in MySQL
  nicht transaktional – das Netz ist die Sicherung vor dem Lauf.
* **Keine Backend-Klassen** (`Backend`, `Session`). Dataset-Klassen laufen mit neuem Code
  gegen ein halb migriertes Schema – im Zweifel reines SQL.
* **Migrationen bauen keine Caches** – das macht `upgrade` am Ende.
* Keine Seed-Zeile mit der Id 0 in Tabellen mit Eingabemaske (0 heißt dort „neu").

**SQL-Modus.** Das Schema setzt den nicht-strikten Modus voraus: Viele `NOT NULL`-Spalten
haben kein `DEFAULT`, INSERTs lassen sie weg, und einige Spalten haben die Vorgabe
`0000-00-00 00:00:00` (geschrieben wird `Cms::DATE_MIN`, siehe [03](03-dataset-api.md#schema)). `Database` setzt deshalb bei jeder Verbindung
`SET SESSION sql_mode` auf `db.sql_mode` (Vorgabe `NO_ENGINE_SUBSTITUTION`), vor den
`db.init_querys` – unabhängig vom Standard des Servers (MySQL 8: `STRICT_TRANS_TABLES`
u. a.). `null` lässt den Modus des Servers stehen.

## Der Installer

```
composer create-project camindo/project mysite     Skelett + Core + vendor
    └─ post-create-project-cmd: camindo install
```

Auch nachträglich: `php Core/cron/camindo.php install`. Es gibt bewusst **keinen
Webinstaller** – wer Composer ausführt, hat eine Shell, und ein Webinstaller wäre ein
zweiter, ungeschützter Weg in die Datenbank.

| Schritt (`--step=`) | Tut |
|---|---|
| `requirements` | PHP-Version und -Erweiterungen, Schreibrechte, Programme (`magick`, `ffmpeg`, `mysqldump`) suchen |
| `key` | `.camindo.key` erzeugen (32 Zufallsbytes, `0600`) – **Verlust = verschlüsselte Inhalte unlesbar**. Eine vorhandene Datei bleibt |
| `config` | `.camindo.php` aus `Core/install/templates/camindo.php.dist` |
| `sqlpwd` | `.sqlpwd` für `mysqldump` |
| `dirs` | `var/…`, `public/media/default`, `public/backend/assets` als Kopie von `Core/public_master/backend/assets` (`--symlink-assets`: relativer Symlink für die Arbeit am Core, scheitert er, wird kopiert). `upgrade` frischt eine Kopie auf, ein Symlink bleibt |
| `database` | `schema.sql` + `seed.sql`; danach die Standard-Instanz (Id 1, Auffang für alle Domains) mit der Hauptsprache `en_US` – ohne Domain und Protokoll, beides kommt aus dem Aufruf. Eine vorhandene Instanz 1 (`--fixtures`) bleibt unberührt |
| `admin` | erster Benutzer (Superuser); das Passwort muss der Richtlinie genügen |
| `core` | `system.core_version` |
| `components` | Zusatzkomponenten **entdecken**, nicht installieren |
| `runtime` | Laufzeitlisten, Caches, `.htaccess`-Block, Versionsstempel |
| `tasks` | Wartungsaufgaben des Zeitplaners (`Scheduler::createDefaults()`, nur fehlende Typen) |
| `finish` | `var/install.lock`, Zusammenfassung mit der Cron-Zeile |

Existiert `var/install.lock`, endet `install` sofort mit Exit-Code 0; `--force --yes`
setzt bewusst neu auf (Datenbank wird geleert). Einzelne Schritte lassen sich mit
`--step=NAME` wiederholen, um eine beschädigte Installation zu reparieren.

**Nicht-interaktiv** (Docker, CI): jede Frage hat einen Parameter und eine
Umgebungsvariable – `--db-host` / `CAMINDO_DB_HOST`, `--db-name`, `--db-user`,
`--db-pass-file`, `--admin-name`, `--admin-email`, `--admin-pass-file`, `--admin-lang`,
`--timezone`, `--locale`, `--https=y|n`, `--backend-domain`, `--web-user` (jeweils `CAMINDO_…`).
Passwörter nur aus Dateien oder Umgebungsvariablen, nie als Argument (Prozessliste).
`CAMINDO_DB_PASS` gilt auch im Betrieb ([13](13-security-authentication.md#geheimnisse-aus-der-umgebung)):
Kommt das Passwort von dort, schreibt der Installer es nicht in `.camindo.php` und legt
keine `.sqlpwd` an; mit `CAMINDO_KEY` entsteht keine `.camindo.key`.

**Eigentümer der Geheimnis-Dateien** (`SecretFile`): `.camindo.key`, `.camindo.key.retired`,
`.camindo.php` und `.sqlpwd` entstehen mit `0600`. Das passt, wenn CLI und Webserver unter
demselben Benutzer laufen (PHP-FPM-Pool je Kunde, suEXEC, mpm-itk). Läuft der Installer
als root (Docker-Entrypoint, `sudo`), übergibt er die Dateien per `chown` an den
Webserver-Benutzer: `--web-user` / `CAMINDO_WEB_USER`, sonst den Eigentümer des
Projektverzeichnisses bzw. von `public/`, falls nicht root. Geraten wird nicht – mit
mod_php auf einem Server mit mehreren Kunden wäre ein für den gemeinsamen Apache-Benutzer
lesbarer Schlüssel für jeden Kunden lesbar. Ohne root ändert der Installer nichts; ohne
bekannten Benutzer warnen `install` und `status`. Mit `--web-user` übernimmt `install`
auch vorhandene Dateien, `install --step=key --web-user=NAME` repariert also den
Eigentümer. `chown` scheitert nie mit Abbruch, nur mit einer Warnung. `key rotate`
übernimmt als root den Eigentümer der bisherigen `.camindo.key`.

Mit `--no-interaction` bricht ein fehlender Pflichtwert mit Exit-Code 2 ab, bevor etwas
geschrieben wird. `--create-db` legt die Datenbank an, `--fixtures` lädt statt der
Stammdaten einen Entwicklungsbestand (`dev.sql`, falls vorhanden).

## Core-Update

```
php Core/cron/camindo.php maintenance on        empfohlen
composer update camindo/cms                     Dateien; der Hook meldet „core update pending"
php Core/cron/camindo.php upgrade               Datenbank und Laufzeit
```

`upgrade` im Einzelnen (jeder Schritt protokolliert, ein Fehler bricht ab und lässt den
Wartungsmodus stehen):

1. Wartungsmodus setzen, falls nicht gesetzt
2. Sicherung: `mysqldump` → `var/dbbackup/upgrade-{von}-{nach}-{zeit}.sql`
3. Core-Migrationen von `system.core_version` bis `CAMINDO_VERSION`
4. Core-Komponenten in `component` abgleichen
5. `ComponentManager::refresh()`; ohne `--no-components` alle Zusatzkomponenten mit
   wartendem Update in Abhängigkeitsreihenfolge aktualisieren – eine, deren
   `requires.core` die neue Version nicht erlaubt, bleibt inaktiv
6. Laufzeit: Listen, Snippet- und HTML-Cache, Extension-Assets, kopierte Backend-Assets, `InstanceCache`,
   `.htaccess`-Block, Templates neu kompilieren, Handbuch-Index
7. `system.core_version`, `var/config/core.php`
8. Wartungsmodus aufheben (nur den eigenen, oder mit `--unlock`)

`upgrade` ist **wiederholbar**: ohne Versionswechsel läuft nur der Laufzeitteil – das
ist zugleich der Befehl „alles neu aufbauen" nach einem Deployment.

**Rollback** ist ein Rezept, kein Befehl – DDL ist nicht zurückrollbar, und die Dateien
gehören Composer:

```
php Core/cron/camindo.php maintenance on
mysql … < var/dbbackup/upgrade-9.0.3-9.1.0-….sql      Datenbank zurück
git checkout composer.lock && composer install         Dateien zurück
php Core/cron/camindo.php upgrade                      Laufzeit neu
php Core/cron/camindo.php maintenance off
```

Daraus folgt: `composer.lock` gehört ins Git des **Projekts**.

## Zusatzkomponenten

```
composer require acme/camindo-module-blog                 Dateien nach Modules/Blog/
php Core/cron/camindo.php component install Blog --activate

composer update acme/camindo-module-blog                  neue Dateien
php Core/cron/camindo.php component update Blog           Migrationen – unmittelbar danach

php Core/cron/camindo.php component uninstall Blog [--purge]
composer remove acme/camindo-module-blog
```

* `install` legt Tabellen und Zeile an, die Komponente ist danach **inaktiv** (Schema
  anlegen und scharfschalten sind verschiedene Entscheidungen); `--activate` kürzt ab.
* `update` behält den Status; zwischen `composer update` und `component update` läuft
  neuer Code auf altem Schema – deshalb beides direkt nacheinander, im Deploy-Script
  unter Wartungsmodus.
* `uninstall` behält die Tabellen standardmäßig; `--purge` löscht die in `tables`
  genannten. Weil `tables` in der Zeile gespeichert ist, lässt sich auch eine verwaiste
  Komponente (Dateien schon weg) vollständig entfernen.
* Vor `install`, `update` und `uninstall` läuft eine Sicherung (`--no-backup` schaltet
  sie ab).

## Wartungsmodus

`var/maintenance.lock` (Zeitpunkt und Grund) – gesetzt von `maintenance on`, `upgrade`
und im Modul Tools.

| Wo | Verhalten |
|---|---|
| Frontend (`SystemState::guardFrontend()`, vor jedem Datenbankzugriff) | 503 mit `Retry-After`, statische Seite (überschreibbar durch `var/custom/maintenance.html`) |
| Backend | Superuser: Hinweisbalken, alles erreichbar; andere: Wartungsseite |
| Cron-Scripte | beenden sich sofort |
| Kommandozeile | läuft |

Dasselbe Verhalten gilt, wenn der Versionsstempel nicht zur Code-Version passt.

## Die Kommandozeile

Ein Einstieg, Unterbefehle, keine Bibliothek:

```
php Core/cron/camindo.php <befehl> [argumente] [--optionen]
```

| Befehl | Tut |
|---|---|
| `status [--after-composer] [--json]` | Dateien gegen Datenbank: Versionen, wartende Migrationen und Updates, Wartungsmodus |
| `install [--no-interaction --fixtures --create-db --symlink-assets --force --yes --step=…]` | Erstinstallation |
| `upgrade [--no-components --no-backup --unlock]` | Core-Migrationen, Komponenten-Updates, Laufzeit, Caches |
| `maintenance on [--reason=…] \| off \| status` | Wartungsmodus |
| `backup [--reason=…] \| list \| prune --keep=N` | Sicherungen nach `var/dbbackup/` |
| `schema export` | `schema.sql`, `seed.sql`, `dev.sql` aus der Entwicklungsdatenbank |
| `cache snippets\|html\|docs\|templates\|include\|all [--prefix=… \| --remove=… \| --template=…] \| list` | Caches |
| `cache records [--instance=ID] [--area=ID,…]` | `record_cache` leeren (`RecordCache::clear()`, nicht Teil von `all`; [02](02-data-model.md#record_cache--inhalts-cache-je-sprache)) |
| `component list \| install \| activate \| deactivate \| update \| uninstall NAME` | Lebenszyklus einer Komponente; `install --all`, `update --all` |
| `refresh` | Verzeichnisse → `component`, Laufzeitlisten; `refresh snippets\|html\|records` wie `cache …` |
| `htaccess [status] \| renew` | verwalteten Block von `public/.htaccess` prüfen (Exit-Code 1: veraltet) bzw. schreiben (`HtaccessWriter`) |
| `admin password NAME [--pass-file=…] \| sign` | Backend-Passwort setzen; alle Signaturen neu schreiben |
| `key status \| rotate \| retire KID` | Schlüsselbund von `.camindo.key` ([13](13-security-authentication.md#schlüsselrotation)) |
| `audit verify [TYP ID] \| restore [--dry-run] \| reseal TYP ID \| delete TYP ID \| strip-names \| migrate` | Audit-Log: Kette prüfen, nach dem Zurückspielen eines Dumps kürzen, gebrochene Kette neu verketten, Verlauf löschen, Namen entfernen ([13](13-security-authentication.md#audit-log)) |
| `api login \| whoami \| methods \| describe \| call \| logout` | API auf der Kommandozeile ([16](16-api.md#kommandozeile)) |
| `fulltext [--area=ID,…] [--stale]` | Volltext neu aufbauen – alle Datensatzbereiche, die genannten oder die als veraltet markierten ([02](02-data-model.md#volltextsuche)) |
| `scheduler list \| defaults \| run ID\|TYP` | Aufgaben des Zeitplaners auflisten, fehlende Wartungsaufgaben anlegen, eine Aufgabe sofort ausführen (etwa `run trash`) ([07](07-media-processing.md#zeitplaner-modul-scheduler)) |

Exit-Codes: 0 ok, 1 Fehler, 2 Bedienfehler; `--quiet` unterdrückt alles außer Fehlern.
Befehle sind Klassen unter `Core/src/Cli/{Name}Command.php`. Installation und Core-Update
gibt es bewusst nur auf der Kommandozeile – ein Core-Update aus dem Backend tauschte den
Code aus, der es gerade ausführt.

Weitere Scripts unter `Core/cron/`: `cron.php` (der einzige Cronjob), `scheduler.php` (von ihm gestartet),
`template_sync.php`, `trigger_ff.php`, `trigger_magick.php` (von ihm gestartet),
`record_export.php`, `record_import.php`, `record_batch.php`, `mass_delete.php`,
`thumbnails.php` (Hintergrundprozesse aus dem Backend), `rebuild_trees.php` (Bäume
reparieren) und die Selbsttests `check_dateparse.php`, `check_nested_select.php`,
`check_tree_positions.php`, `check_template_params.php`, `check_include_cache.php`,
`check_webauthn.php`, `check_record_files.php` (Arbeitskopie und Datei-Felder). Die CI führt
sie nach einer Neuinstallation gegen MySQL 8.0 sowie MariaDB 10.6 und 11.4 aus.
