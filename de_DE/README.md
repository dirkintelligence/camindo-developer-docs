# camindo CMS – Entwicklerdokumentation

Diese Dokumentation richtet sich an alle, die camindo CMS 9 **erweitern oder
weiterentwickeln**: Module, Medienprozessoren und Extensions schreiben, eigene
`(CMS:…)`-Befehle bereitstellen, den Core verstehen oder ändern.

Wer Inhalte pflegt, eine Installation betreut oder Templates schreibt, findet alles
Nötige in den drei Handbüchern (Anwender, Administratoren, Template-Programmierer),
die im Backend unter „Handbücher" erreichbar sind. Diese Dokumentation setzt deren
Begriffe voraus und beschreibt, was dahinter liegt.

## Voraussetzungen

* PHP ≥ 8.1 mit den Erweiterungen `mysqli`, `intl`, `gd`, `json`, `mbstring`
* MySQL ≥ 8.0.17 (JSON-Spalten, Multi-Value-Indizes)
* Composer 2
* optional: ImageMagick (`magick`/`convert`), `ffmpeg`/`ffprobe`, `mysqldump`
* Grundkenntnisse in PHP 8 (Typen, Constructor Property Promotion), SQL und jQuery

camindo CMS ist bewusst **frameworkfrei**: Bootstrap, Dependency-Injection-Container,
Datenbank-Abstraktion, Template-Compiler, Kommandozeile und Backend-Oberfläche sind
Eigenentwicklungen. Externe Abhängigkeiten beschränken sich auf wenige Bibliotheken
(PHPMailer, Parsedown, im Backend jQuery und Bootstrap).

## Die Grundidee in fünf Sätzen

1. ***Inhaltsbereiche*** (`area`) legen fest, welche ***Eingabefelder*** (`field`) es
   gibt – das Datenmodell wird im Backend zusammengestellt, nicht programmiert.
2. ***Datensätze*** (`record`) halten nur Metadaten; die Feldinhalte liegen typgetrennt
   in `data_txt`, `data_int`, `data_fil`, `data_rel` … (EAV-Muster mit
   Typ-Partitionierung).
3. `Cms::select()` setzt diese „virtuellen Felder" zur Laufzeit per JOIN zusammen –
   Filtern und Sortieren fühlt sich an wie auf einer echten Tabelle.
4. ***Templates*** werden in einer eigenen Befehlssprache `(CMS:…)` geschrieben, als
   Datei unter `var/templates/` gepflegt und beim Speichern **zu PHP kompiliert**; zur
   Laufzeit läuft reines PHP. Aus der Datenbank gelangt nie Code ins Dateisystem.
5. ***Prozessoren*** erzeugen beim Speichern abgeleitete Medienversionen (Thumbnails,
   Videoformate); lange Umwandlungen laufen als Hintergrundprozess.

## Inhalt

| Kapitel | Inhalt |
|---|---|
| [01 – Architektur](01-architecture.md) | Verzeichnisse, Bootstrap, Konfiguration, DI-Container, die vier Request-Wege, Backend-Layer, Kernklassen |
| [02 – Datenmodell](02-data-model.md) | Tabellen, EAV-Prinzip, Mehrsprachigkeit, Bäume, Relationen, Arbeitskopien, Medienablage, Löschen und Papierkorb |
| [03 – Dataset-API](03-dataset-api.md) | Die Tabellen-Abstraktion: Schema, Hooks, `init/load/save/delete`, Validierung |
| [04 – Suchen und Filtern](04-select-and-filter.md) | `Cms::select()`: Parameter, Filter-Syntax, Sortierung, Paginierung |
| [05 – Backend-Entwicklung](05-backend-development.md) | Aufbau eines Moduls, die Klasse `Backend`, Snippets, HTML-Vorlagen, JavaScript-API, Widgets |
| [06 – Template-Compiler](06-template-compiler.md) | Vom Quelltext zum PHP, Befehlsdefinition, eigene Befehle, Laufzeit der wichtigsten Befehle |
| [07 – Medien und Hintergrundprozesse](07-media-processing.md) | Prozessoren, Konverter, Thumbnails, Cron, Trigger, Zeitplaner |
| [08 – Komponenten und Extensions](08-components-and-extensions.md) | Komponenten-Vertrag, `settings.php`, Lebenszyklus, Extensions für Eingabefelder |
| [09 – Installation, Update, Kommandozeile](09-installation-and-update.md) | Pakete, Installer, Migrationen, Core-Update, Wartungsmodus, CLI |
| [10 – Komponenten veröffentlichen](10-publishing-components.md) | Composer-Pakete, Versionsregeln, Releases |
| [11 – Rechte](11-permissions.md) | Benutzergruppen, Modul-, Instanz-, Bereichs- und Sprachrechte, die Klasse `Access` |
| [12 – Frontend-Community](12-community.md) | Website-Benutzer, Sessions, die `CMS:USER_*`-Befehle, geschützte Medien |
| [13 – Sicherheit I: Anmeldung, Schlüssel, Protokoll](13-security-authentication.md) | Passwörter, Signaturen, Sessions, zweiter Faktor, Schlüsselrotation, Audit-Log |
| [14 – Sicherheit II: Web](14-security-web.md) | CSRF, HTML-Filter, Sicherheits-Header, Script-Policy, Uploads, ImageMagick, Backend-Domain |
| [15 – Konventionen](15-conventions.md) | Namen, Code-Stil, SQL, Sprachen, Schreibrichtung, Dateirechte |
| [16 – API](16-api.md) | Die API als Oberfläche: Sessions, Kanäle, Aufrufe, Schweregrade, eigene API-Methoden, Kommandozeile, Assistent (Modul Ai) |

Einsteigern empfiehlt sich die Reihenfolge 01 → 02 → 03 → 05; wer nur einen
Medienprozessor oder eine Extension schreiben will, liest 01, 07 bzw. 08.

## Lizenz

camindo CMS steht unter der **GNU AGPL v3 oder später** (`Core/LICENSE`, Vermerk und
Zusatzerlaubnis in `Core/LICENSING.md`). Die Zusatzerlaubnis nach § 7 nimmt Templates,
die von ihnen ausgelieferten Dateien und alle Inhalte aus: Sie dürfen unter beliebiger
Lizenz stehen. Änderungen am Core und Komponenten (Module, Prozessoren, Extensions)
bleiben AGPL-pflichtig. Wer einen **geänderten** Core im Netz betreibt, muss den Nutzern
den Quellcode seiner Fassung anbieten (§ 13). Die Lizenzen der Fremdbibliotheken führt
`Core/THIRD-PARTY-NOTICES.md` auf.

## Schreibweisen

* `Klasse::methode()` bezeichnet eine Methode, gleich ob statisch oder nicht.
* Klassen ohne Namespace-Angabe liegen in `Camindo\Core\` (Verzeichnis `Core/src/`),
  Modulklassen in `Camindo\Core\Modules\{Modul}\` (`Core/Modules/{Modul}/`).
* Pfade sind relativ zum Projektverzeichnis (dem Verzeichnis mit `composer.json`).
* Konfigurationsschlüssel stehen in Punkt-Schreibweise: `admin.max_inactive_sec`
  meint `$_cfg['admin']['max_inactive_sec']` in `.camindo.php`.
* ***Fett und kursiv*** gesetzte Begriffe werden an dieser Stelle erklärt.

## Glossar

| Begriff | Bedeutung |
|---|---|
| **Administrator** | Backend-Benutzer (Tabelle `admin`) |
| **User** | Website-Besucher mit Konto; sein Profil ist ein Datensatz in einem Bereich vom Typ `profile` |
| **Instanz** | eine Website bzw. ein Mandant (Tabelle `instance`) |
| **Inhaltsbereich** (Area) | Container mit eigener Felddefinition; Typen `record`, `profile`, `ugc`, `template`, `folder` |
| **Feld** | benutzerdefiniertes Eingabefeld eines Bereichs |
| **Metafeld** | fest im PHP-Schema definiertes Feld, eine echte Tabellenspalte |
| **Virtuelles Feld** | benutzerdefiniertes Feld; der Inhalt liegt in einer `data_*`-Tabelle |
| **Arbeitskopie** | temporärer Datensatz (`tmp = 1`) während der Bearbeitung |
| **Hauptsprache** | `instance.primary_language_id` – Rückfall aller mehrsprachigen Felder |
| **Komponente** | Oberbegriff für Modul, Prozessor und Extension, jeweils Core oder Zusatz |
| **Prozessor** | Implementierung einer Medienumwandlung (z. B. `ImageGd`) |
| **Konverter** | konfigurierte Instanz eines Prozessors (Tabelle `converter`) |
| **Extension** | JavaScript-Erweiterung für ein Eingabefeld im Backend |
| **Snippet** | mehrsprachiger Textbaustein, `$cms->snippet('admin.field_admin_name')` |
