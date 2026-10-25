# simplysmart/tools

> **Pakiet porzucony.** Narzędzia (wydania, paczki prywatne, wdrożenie, skille AI) są teraz częścią
> [`simplysmart/admiro`](https://github.com/marcinulezalka/simply-admiro) (`docs/narzedzia.md`, `vendor/bin/simplysmart`).
> Zamień `simplysmart/tools` w `require-dev` na `simplysmart/admiro` i w `boost.json` na `simplysmart/admiro`.

Przybornik deweloperski dla projektów i pakietów simplysmart. Bez zależności poza PHP ^8.4,
bez frameworka.

## Instalacja

```bash
composer require --dev simplysmart/tools
```

## Polecenie simplysmart

`vendor/bin/simplysmart` to jedno wejście do narzędzi pakietu, jak `artisan`: nazwy z dwukropkiem
(kropka działa tak samo — `deploy.onboarding` = `deploy:onboarding`), bez nazwy albo z `list` — lista poleceń.

| Polecenie | Co robi |
| --- | --- |
| `release` | wydanie wersji (= `vendor/bin/version-release`) |
| `private:install` | paczki prywatne (= `vendor/bin/composer-private`) |
| `deploy:lock` | manifest wdrożeniowy (= `vendor/bin/composer-private --deploy`) |
| `deploy` | wdrożenie na serwer |
| `deploy:onboarding` | wdrożenie i konta pierwszej instalacji |

Opcje poleceń są takie jak w samodzielnych narzędziach (`--dry-run`, `--root=<katalog>`); `vendor/bin/version-release`
i `vendor/bin/composer-private` działają dalej.

## Wydanie wersji

Projekt i każdy pakiet mają własne `CHANGELOG.md` ([Keep a Changelog](https://keepachangelog.com/pl/1.1.0/))
i `VERSION` (`MAJOR.MINOR.PATCH`). Skrypt czyta wpisy z `## [Unreleased]`, ustala podbicie,
zapisuje nową wersję do `VERSION` i zamienia `[Unreleased]` na datowaną sekcję wydania.

```bash
vendor/bin/version-release                                        # bieżący katalog
vendor/bin/version-release --root=packages/simplysmart/mainboard  # inny projekt lub pakiet
vendor/bin/version-release --major-confirm                        # gdy wymagany MAJOR
vendor/bin/version-release --dry-run                              # podgląd: pokazuje wersję, nie zmienia plików
```

| Sekcja w `[Unreleased]` | Podbicie |
| --- | --- |
| `Removed` lub wpis zaczynający się od `**BREAKING**` / `BREAKING:` | MAJOR (wymaga `--major-confirm`) |
| `Added`, `Deprecated` | MINOR |
| `Changed`, `Fixed`, `Security` | PATCH |

Wygrywa najwyższe podbicie wśród niepustych sekcji. Inne nagłówki i wpisy bez nagłówka
`### <sekcja>` są odrzucane. Datę wydania liczy strefa `RELEASE_TIMEZONE`
(domyślnie `Europe/Warsaw`).

## Paczki prywatne

`vendor/bin/composer-private` instaluje paczki z prywatnych repozytoriów tak, żeby publiczny `composer.json` i `composer.lock` projektu ich nie wymieniały. Lista leży w `private-packages.json` w katalogu projektu (dodaj go do `.gitignore`):

```json
{
    "require": {
        "<vendor>/<paczka>": "@dev"
    }
}
```

Narzędzie zapisuje `composer.private.json` (= `composer.json` z dopisanymi paczkami), przy pierwszym uruchomieniu kopiuje `composer.lock` jako `composer.private.lock` (wersje publicznych paczek zostają) i uruchamia `composer update <paczki>` z `COMPOSER=composer.private.json`. Oba pliki dodaj do `.gitignore`.

```bash
vendor/bin/composer-private              # bieżący katalog
vendor/bin/composer-private --dry-run    # pokazuje paczki; nie zapisuje plików i nie uruchamia composera
vendor/bin/composer-private --root=<katalog>
```

Bez `private-packages.json` kończy się kodem 0 i nic nie robi (publiczny klon, CI). Zwykłe `composer install` / `update` usuwa paczki prywatne z `vendor/`, więc potem trzeba uruchomić narzędzie ponownie. W projekcie wygodnie podpiąć je jako skrypt: `"private:install": "@php vendor/bin/composer-private"`.

### Wdrożenie: paczki z repozytoriów git

Lokalnie paczki z repozytoriów typu `path` (np. `packages/*/*`) są dowiązaniami, a `composer.lock` zapisuje je jako „z katalogu”. Na serwerze tych katalogów nie ma, więc tryb `--deploy` buduje osobny manifest wdrożeniowy:

- `composer.deploy.json` — `composer.json` bez repozytoriów `path`, z repozytorium VCS (adres `git remote origin`) każdej paczki z tych katalogów, którą projekt wymaga (`require`, `require-dev`, paczki prywatne), i z dopisanymi paczkami prywatnymi;
- `composer.deploy.lock` — kopia `composer.private.lock` (albo `composer.lock`) zaktualizowana przez `composer update <paczki> --no-install --no-scripts`: paczki przypięte do commitów z repozytoriów zdalnych, `vendor/` bez zmian.

```bash
vendor/bin/composer-private --deploy             # po wypchnięciu repozytoriów paczek
vendor/bin/composer-private --deploy --dry-run   # pokazuje paczki; nie zapisuje plików
```

Paczka z niewypchniętymi commitami dostaje ostrzeżenie (lock wskaże kod z repozytorium zdalnego), paczka bez `origin` to błąd. Oba pliki dodaj do `.gitignore` i wyślij na serwer, a tam: `COMPOSER=composer.deploy.json composer install --no-dev`. Do repozytoriów prywatnych composer potrzebuje tokenu (`composer config -g github-oauth.github.com <token>`). W projekcie: `"deploy:lock": "@php vendor/bin/composer-private --deploy"`.

## Wdrożenie na serwer

`vendor/bin/simplysmart deploy` uruchamia się na komputerze deweloperskim, w katalogu projektu:

1. sprawdza, czy projekt i paczki z `packages/*/*` są wypchnięte (niewypchnięte commity zatrzymują wdrożenie,
   niezacommitowane zmiany i nieudany `git fetch` dają ostrzeżenie; katalogi bez `.git` są pomijane),
2. tworzy manifest wdrożeniowy (jak `deploy:lock`; bez repozytoriów `path` zostaje istniejący lock),
3. wysyła `composer.deploy.json` i `composer.deploy.lock` na serwer (`scp`),
4. uruchamia na serwerze skrypt `resources/deploy/remote.sh` przez `ssh … 'bash -s'`: Composer 2.8+, `.env`,
   `git pull --ff-only`, token GitHub, `composer install --no-dev`, `APP_KEY`, migracje, role, słowniki, cache
   i skrypt crona `storage/app/cron.sh` (`mainboard:cron:script --php=<DEPLOY_PHP>`, gdy Core ma tę komendę; w panelu
   hostingu: program BASH, ten plik, co minutę).
   Skrypt i token idą standardowym wejściem, więc serwer nie potrzebuje tools, a token nie trafia do listy procesów.

Konfiguracja: `infrastructure/deploy.env` (dodaj do `.gitignore`; w projekcie wzór `deploy.env.example`), format
`KLUCZ=wartość` bez uruchamiania powłoki — wartości ze spacjami w cudzysłowie `"…"` (`\"` i `\\` jako znaki) albo
`'…'` (dosłownie), komentarz po `#`, opcjonalny `export`.

| Klucz | Znaczenie |
| --- | --- |
| `DEPLOY_SSH` | konto SSH serwera (`login@serwer` albo alias z `~/.ssh/config`), wymagane |
| `DEPLOY_PATH` | katalog aplikacji na serwerze (repozytorium projektu z `.env`), wymagane; `~/` = katalog domowy |
| `DEPLOY_PHP` | polecenie PHP 8.4 na serwerze (domyślnie `php84`) |
| `REMOTE_COMPOSER` | plik Composera na serwerze (np. `composer.phar`); pusty = `~/bin/composer` |
| `LOCAL_COMPOSER` | polecenie Composera na komputerze (np. `php composer.phar`); pusty = `composer` |
| `GITHUB_TOKEN` | token tylko do odczytu do repozytoriów paczek; zapisywany w konfiguracji Composera na serwerze |

```bash
vendor/bin/simplysmart deploy --dry-run                    # konfiguracja, repozytoria i plan; nic nie wysyła
vendor/bin/simplysmart deploy
vendor/bin/simplysmart deploy --config=infrastructure/test.env
```

### Onboarding

`vendor/bin/simplysmart deploy:onboarding` robi to samo wdrożenie, a na końcu zakłada konta pierwszej instalacji
z `infrastructure/deploy.onboarding.env` (dodaj do `.gitignore`; inny plik: `--onboarding=<plik>`):

```bash
ONBOARD_ACCOUNT_1_ZONE=admiro                # strefa: polecenie <strefa>:account:create, wymagane
ONBOARD_ACCOUNT_1_EMAIL=admin@firma.pl       # wymagane
ONBOARD_ACCOUNT_1_NAME="Jan Kowalski"        # wymagane
ONBOARD_ACCOUNT_1_ROLES=hero                 # role po przecinku, opcjonalnie
ONBOARD_ACCOUNT_1_ORGANIZATION=              # token organizacji ról, opcjonalnie
```

Każde konto to `php artisan <strefa>:account:create <e-mail> <imię> --role=… --organization=… --skip-existing`,
więc istniejący adres jest pomijany i onboarding można powtórzyć. Nieznany klucz, brak wymaganej wartości albo
nazwa strefy spoza `[a-z0-9_-]` to błąd przed połączeniem z serwerem. `--dry-run` pokazuje konta bez wysyłania.

## Skille AI

Pakiet dostarcza wspólne skille dla agentów AI w `resources/boost/skills/`: `solid-dry` (SOLID, DRY, standardy PHP 8.4), `docs-after-tests` (dokumentacja po testach) i `changelog-before-commit` (CHANGELOG i wydania z `--dry-run`). W projekcie Laravel z [Laravel Boost](https://laravel.com/docs/boost) dopisz pakiet do `boost.json`, a Boost skopiuje skille do `.claude/skills/`:

```json
{ "packages": ["simplysmart/tools"] }
```

```bash
php artisan boost:update
```

## Rozwój

PSR-4 (autoload: jedna klasa na plik, namespace odpowiada katalogowi) i styl PER Coding Style (następca PSR-12) pilnowany przez Laravel Pint z presetem `per` (`pint.json`), który wymaga też `declare(strict_types=1);` w każdym pliku PHP.

```bash
composer install
composer test      # PHPUnit
composer lint      # sprawdzenie stylu i strict types
composer format    # poprawienie stylu
composer analyse   # PHPStan, poziom max
```

## Licencja

Simplysmart Restricted Source License 1.0 — © 2026 Marcin Ulezalka. Do celów niekomercyjnych
kod możesz bezpłatnie przeglądać, kopiować, uruchamiać i modyfikować. Użycie komercyjne (także
w projektach dla klientów) jest dozwolone po wpisaniu deklaracji do pliku `THIRD_PARTY.md` w
Twoim projekcie — wzór wpisu jest w [THIRD_PARTY.md](THIRD_PARTY.md). Sprzedaż,
sublicencjonowanie i komercyjna redystrybucja kodu jako produktu wymagają pisemnej zgody autora.
Kopie muszą zachować informację o prawach autorskich i treść licencji. Wiążąca jest angielska
treść pliku [LICENSE](LICENSE).
