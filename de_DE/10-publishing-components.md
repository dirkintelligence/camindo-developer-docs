# 10 – Komponenten veröffentlichen

Wie aus einem Modul, einem Prozessor oder einer Extension ein Paket wird, das andere mit
`composer require` installieren – und welche Versionsregeln dabei gelten.

## Wie Composer ein Paket findet

Ein Composer-Paket ist ein Git-Repository mit einer `composer.json` im Wurzelverzeichnis
und **Tags** als Versionen. Es gibt kein „Hochladen" – Composer lädt den Stand des Tags
direkt aus dem Repository.

| Weg | Eintrag im Projekt | Repository privat? | Bemerkung |
|---|---|---|---|
| Packagist | keiner | nein | öffentliches Repo und `license` sind Pflicht |
| `vcs` | `composer config repositories.blog vcs https://…/camindo-module-blog.git` | ja, mit Zugangstoken beim Nutzer | Composer liest Tags und Branches direkt |
| `path` | `{"type": "path", "url": "../camindo-module-blog"}` | ja (lokal) | Entwicklung: Symlink, Änderungen wirken sofort |

Zwei Regeln bestimmen alles Weitere:

1. **`repositories` gilt nur in der `composer.json` des Projekts.** Einträge in der
   `composer.json` eines Pakets ignoriert Composer.
2. **Versionen sind Tags.** `^1.0` findet nur ein Paket mit einem Tag `v1.0.0`; ein Branch
   `main` heißt für Composer `dev-main`. Schreiben Sie **kein** `version`-Feld in die
   `composer.json` – es müsste bei jedem Release mitgezogen werden.

## Ein Repository je Komponente

```
camindo-module-blog/                Repo-Name frei, üblich: camindo-{art}-{name}
├── composer.json
├── README.md                       kurz: Zweck, Installation, Lizenz, Links auf docs/
├── LICENSE
├── Blog.php
├── install/
│   ├── settings.php                version, requires, tables …
│   ├── schema.sql                  CREATE TABLE IF NOT EXISTS …
│   ├── seed.sql                    optional
│   ├── Installer.php               optional
│   └── migrations/                 1.1.0.sql, 1.1.0.php, 1.2.0.sql …
├── lang/  backend/  public/
├── docs/de_DE/01-module-blog.md    Handbuch (siehe 08, „Handbuch einer Komponente“)
├── docs/en_US/01-module-blog.md
└── .gitignore                      /vendor/, composer.lock
```

Das Verzeichnis im Projekt (`Modules/Blog/`) kommt aus `extra.installer-name`, nicht aus
dem Repo-Namen.

```json
{
    "name": "acme/camindo-module-blog",
    "description": "Blog module for camindo CMS",
    "type": "camindo-module",
    "license": "MIT",
    "require": {
        "php": ">=8.3",
        "camindo/cms": "^9.0"
    },
    "autoload": { "psr-4": { "Camindo\\Modules\\Blog\\": "" } },
    "extra": { "installer-name": "Blog" }
}
```

| Feld | Regel |
|---|---|
| `name` | `{vendor}/camindo-{module\|processor\|extension}-{name}`, kleingeschrieben |
| `type` | `camindo-module`, `camindo-processor` oder `camindo-extension` – sonst landet das Paket in `vendor/` |
| `require.camindo/cms` | derselbe Wert wie `settings.php['requires']['core']` |
| `autoload` | optional – das Projekt bildet `Camindo\` auf `./` ab; mit Eintrag ist das Paket vollständig beschrieben |
| `extra.installer-name` | Pflicht, StudlyCase |
| `version` | nicht setzen |
| `require.camindo/composer-plugin` | nicht setzen – das Plugin bringt das Projekt mit |

Namespaces: Module `Camindo\Modules\{Name}\`, Prozessoren `Camindo\Processors\{Name}\`,
Extensions `Camindo\Extensions\{Name}\`. Bibliotheken, die eine Extension braucht,
kommen über ihr `require` nach `vendor/` (Asset-Pfade mit `vendor:`).

`composer.lock` gehört **nicht** ins Repository einer Komponente – nur ins Git eines
Projekts.

## Lizenz einer Komponente

Eine Komponente läuft im selben Prozess wie der Core und nutzt dessen Klassen – sie ist
damit Teil des Gesamtwerks und braucht eine **AGPL-verträgliche** Lizenz, etwa AGPL-3.0,
GPL-3.0, LGPL-2.1/3.0, MIT, BSD oder Apache-2.0. GPL-2.0-*only* und proprietäre Lizenzen
passen nicht. Bibliotheken, die eine Komponente über `require` holt, müssen ebenfalls
verträglich sein. Die Lizenzdatei gehört als `LICENSE` ins Repository, die Kennung
(SPDX) in `license`.

Templates fallen nicht darunter: Die Zusatzerlaubnis in `Core/LICENSING.md` stellt sie
und alle Inhalte frei.

## Versionsregeln

`MAJOR.MINOR.PATCH`, Tags mit `v`-Präfix (`v1.2.0`).

| Stelle | Erhöhen, wenn … |
|---|---|
| **MAJOR** | eine Installation, die mit der Vorversion lief, angepasst werden muss (entfernte Befehle, geänderte Tabellen, die andere nutzen, geänderte Pfade) |
| **MINOR** | neue Funktionen oder eine neue Migration – Bestehendes läuft weiter |
| **PATCH** | nur Fehlerbehebung, **keine Migration** |

Faustregel: **Migration ⇒ mindestens MINOR.** `component update` findet Migrationen über
den Versionsvergleich.

Eine Komponente hat zwei Versionsangaben, die im selben Commit erhöht werden:

| Ort | Wahrheit für |
|---|---|
| `install/settings.php['version']` | den `ComponentManager` – steuert, welche Migrationen laufen |
| Git-Tag `v1.2.0` | Composer – steuert, welche Dateien ankommen |

Den benötigten Core nennt die Komponente zweimal, mit gleichem Wert:

```json
"require": { "camindo/cms": "^9.0" }             composer.json – Composer prüft beim require/update
```
```php
'requires' => ['core' => '^9.0']                 settings.php – ComponentManager prüft beim install/activate/upgrade
```

`^9.0` heißt „9.0.0 bis unter 10.0.0". Läuft die Komponente mit Core 10 unverändert, wird
daraus `^9.0 || ^10.0` in einem PATCH-Release; braucht sie Anpassungen, kommt ein
MAJOR-Release mit `^10.0`. Unterstützte Syntax in `settings.php`
(`Install\VersionConstraint`): `^`, `~`, `*`, `>=`, `<`, `||`, Komma.

Vorabversionen (`v1.3.0-beta.1`) liegen per `version_compare()` **vor** der Freigabe und
erfüllen kein `^1.3`; Composer installiert sie nur mit angepasster `minimum-stability`.

## Release einer Komponente

```bash
# 1. settings.php: 'version' => '1.2.0'; ggf. neue Migration install/migrations/1.2.0.sql
# 2. auf einer Entwicklungsinstallation prüfen (Komponente als Checkout in Modules/Blog/)
php Core/cron/camindo.php component update Blog
# 3. Commit, Tag, Push
git -C Modules/Blog commit -am "Release 1.2.0"
git -C Modules/Blog tag -a v1.2.0 -m "Blog 1.2.0"
git -C Modules/Blog push origin main --tags
```

Ein veröffentlichtes Tag wird nie mehr verschoben – Composer und Packagist speichern es
zwischen.

## Auf Packagist

Öffentliches Repository, `license` in der `composer.json`, auf packagist.org einreichen
und den GitHub-Hook bzw. die Packagist-App aktivieren, damit neue Tags ohne Zutun
erscheinen. Der Vendor-Name muss dort noch frei sein.

## Was der Nutzer tun muss

| Komponente liegt … | Eintrag im Projekt | dann |
|---|---|---|
| auf Packagist | nichts | `composer require acme/camindo-module-blog` |
| in einem öffentlichen Repository | `composer config repositories.blog vcs https://…` | dito |
| in einem privaten Repository | dito, dazu ein Zugangstoken (`composer config -g github-oauth.github.com …`) | dito |

Und immer danach: `php Core/cron/camindo.php component install Blog --activate`. Die
README der Komponente nennt beide Befehle. Alles Weitere steht im Handbuch der
Komponente (`docs/`), das nach der Installation auch im Backend erscheint; die README
verlinkt dessen Startseite je Sprache:

```markdown
## Manual

- English: [docs/en_US/01-module-blog.md](docs/en_US/01-module-blog.md)
- Deutsch: [docs/de_DE/01-module-blog.md](docs/de_DE/01-module-blog.md)
```
