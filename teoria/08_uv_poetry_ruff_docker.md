# 08. Narzedzia projektu: uv, Poetry, Ruff i Docker

[Zadanie: zbuduj pipeline](../zadania/08_pipeline_narzedzia_docker.md)

Do tej pory `nasa_pipeline` dzialal na naszym komputerze. Teraz wyobraz sobie, ze inna osoba dostaje repozytorium. Nie wie, jakiej wersji Pythona uzyc, co zainstalowac ani jak uruchomic baze. W tej lekcji przygotujemy projekt tak, zeby mogla go odtworzyc bez zgadywania.

| Narzedzie | Jaki problem rozwiazuje? |
|---|---|
| `uv` albo Poetry | Ustala, jakie biblioteki i w jakich wersjach zainstalowac dla projektu. |
| Ruff | Wskazuje wybrane problemy w kodzie i porzadkuje jego wyglad. |
| Docker | Uruchamia nasz kod i osobna baze w kontenerach. |

**`uv` i Poetry to dwie alternatywy.** Nie uzywamy obu do zarzadzania ta sama kopia projektu. Glowne cwiczenie i Docker uzywaja `uv`, a Poetry sprawdzimy osobno.

Punktem wyjscia jest `nasa_pipeline` po [06](06_logowanie_pipeline.md) i [07](07_testy_pipeline_pytest.md): z plikiem `storage.py` oraz testami w `tests/`. Nie zmieniamy transformacji, SQL-a ani zrodla danych.

## 1. `pyproject.toml`, lock i `.venv`: co jest czym?

`requests` jest nam potrzebny do pobrania zdarzen z NASA. Ale sam napis "zainstaluj requests" nie mowi, ktora wersje dostaniemy ani jakie dodatkowe biblioteki zainstaluja sie razem z nim. Dlatego projekt przechowuje dwie rozne informacje:

- **`pyproject.toml`:** czego potrzebujemy, np. "Python 3.12 i biblioteka `requests`". Mozemy wpisac zakres dozwolonych wersji.
- **Plik lock (`uv.lock` albo `poetry.lock`):** jakie konkretne wersje narzedzie dobralo, takze dla zaleznosci posrednich. Zapisujemy go w Git obok `pyproject.toml`, zeby kolejna osoba uzyla tej samej decyzji.
- **`.venv`:** lokalne srodowisko, czyli miejsce na interpreter srodowiska i zainstalowane biblioteki. Kazdy odtwarza je u siebie; nie wrzucamy go do repozytorium.

`requests` i sterownik PostgreSQL sa potrzebne podczas pracy ETL. `pytest` i Ruff sa potrzebne do pracy nad kodem, ale nie do uruchomienia pipeline'u. Dlatego zapisujemy je w osobnej grupie `dev`.

## 2. Glowna sciezka: uporzadkuj projekt z `uv`

Zainstaluj `uv` zgodnie z [instrukcja dla Windows](https://docs.astral.sh/uv/getting-started/installation/). Sprawdz `uv --version` w nowym terminalu. Otworz katalog `nasa_pipeline`, w ktorym jest `main.py`.

Jezeli masz aktywne stare `.venv`, otworz terminal bez jego aktywacji. Najbezpieczniej pracowac na kopii kodu bez `.venv`, bo synchronizacja moze usunac biblioteki, ktorych nie wpisalismy do projektu. Nie kopiuj hasel do nowego repozytorium. Jezeli jest juz `pyproject.toml`, nie uruchamiaj `uv init` ponownie; sprawdz jego zawartosc.

```powershell
uv python install 3.12
uv init --bare --python 3.12
uv python pin 3.12
uv add requests python-dotenv "psycopg[binary]" fastapi uvicorn pydantic
uv add --dev pytest pytest-cov pytest-mock ruff
uv sync --locked
uv run --locked python --version
uv run --locked python -m pytest -q tests
```

Co sie dzieje po kolei?

1. `uv python install` udostepnia Pythona 3.12. `init --bare` tworzy konfiguracje bez generowania nowego `main.py`. `pin` zapisuje wersje w `.python-version`.
2. `uv add` wpisuje biblioteki do `pyproject.toml`, zapisuje dobrane wersje w `uv.lock` i synchronizuje srodowisko. `--dev` oznacza grupe narzedzi do pracy nad kodem.
3. `uv sync --locked` instaluje wedlug locka i przerywa, jezeli konfiguracja wymaga jego aktualizacji. Na drugim komputerze nie wykonujemy ponownie `uv add`: po pobraniu repozytorium uzywamy `uv sync --locked`.
4. `uv run --locked` uruchamia polecenie w srodowisku projektu. Nie musisz recznie aktywowac `.venv`. W PyCharm mozesz wskazac `.venv/Scripts/python.exe` jako interpreter projektu.

Podstawowe testy z 07 powinny przejsc bez NASA i bazy. Uruchamiamy `python -m pytest` z katalogu projektu, zeby Python widzial moduly lezace obok `main.py`. Jesli dodales opcjonalne testy, niektore moga miec wynik `skipped`. Do repozytorium dodaj `pyproject.toml`, `uv.lock` i `.python-version`. W `.gitignore` uwzglednij `.venv/`, `.env`, `.env.api`, `.env.docker` i `data/raw/`.

## 3. Poetry: ten sam problem, inne polecenia

Poetry tez umie zapisac wymagania, dobrac wersje i uruchomic polecenie we wlasciwym srodowisku. Przecwiczymy to w osobnym katalogu `nasa_pipeline_poetry`, obok glownego projektu. Skopiuj tam pliki `.py` i katalog `tests/`, a jezeli masz `pytest.ini`, skopiuj rowniez ten plik. Nie kopiuj `.venv`, `uv.lock`, `pyproject.toml` ani plikow z haslami.

Zainstaluj Poetry wedlug [oficjalnej instrukcji](https://python-poetry.org/docs/#installation) i sprawdz `poetry --version`. Alternatywnie `uv tool install poetry` instaluje samo polecenie Poetry w osobnym srodowisku narzedzia; nie oznacza to, ze `uv` zaczyna zarzadzac projektem Poetry.

Utworz w kopii nowy `pyproject.toml`:

```toml
[project]
name = "nasa-pipeline-poetry"
version = "0.1.0"
requires-python = ">=3.12,<3.13"

[tool.poetry]
package-mode = false
```

`package-mode = false` mowi: instalujemy zaleznosci, ale nie probujemy budowac i instalowac naszego ETL jako paczki. Mamy zwykly projekt uruchamiany przez `main.py`. Otworz katalog kopii w terminalu bez aktywnego srodowiska z projektu `uv`.

```powershell
poetry env use 3.12
poetry add requests python-dotenv "psycopg[binary]" fastapi uvicorn pydantic
poetry add --group dev pytest pytest-cov pytest-mock ruff
poetry install
poetry run python -m pytest -q tests
```

Jesli Poetry nie znajdzie interpretera 3.12, podaj pelna sciezke do niego w `poetry env use`. Jesli zainstalowales go w kroku 2, `uv python find --managed-python 3.12` wyswietli te sciezke. `poetry env info` pozwala sprawdzic, gdzie Poetry utworzylo swoje srodowisko.

| Co chcesz zrobic? | Projekt z `uv` | Projekt z Poetry |
|---|---|---|
| Dodac biblioteke | `uv add requests` | `poetry add requests` |
| Dodac narzedzie do pracy nad kodem | `uv add --dev pytest` | `poetry add --group dev pytest` |
| Zainstalowac projekt z zapisanego locka | `uv sync --locked` | `poetry install`, gdy `poetry.lock` jest juz w repozytorium |
| Uruchomic testy | `uv run --locked python -m pytest` | `poetry run python -m pytest` |

Nie przenosimy `poetry.lock` do glownego projektu. Po cwiczeniu wroc do katalogu `nasa_pipeline` z `uv.lock`.

## 4. Ruff: znajdz problem, a potem sprawdz format

Ruff ma dwa rozne polecenia. **Linter** (`ruff check`) szuka m.in. importow, z ktorych kod nie korzysta. **Formatter** (`ruff format`) ustala m.in. spacje, wciecia i podzial dlugich linii. Zaden z nich nie sprawdzi, czy dane naprawde zapisaly sie do PostgreSQL. Do tego mamy testy i kontrole z 06-07.

Na koncu `pyproject.toml` glownego projektu dopisz (nie dubluj istniejacych sekcji):

```toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["E4", "E7", "E9", "F"]
```

```powershell
uv run --locked ruff check .
uv run --locked ruff format --check .
```

W `storage.py` dodaj na chwile `import csv`, chociaz plik go nie uzywa. Uruchom `ruff check storage.py` przez `uv run --locked`. Ruff powinien wskazac `F401` i linie importu. Usun import i sprawdz jeszcze raz. Format sprawdzamy oddzielnie: `--check` niczego nie zmienia, tylko wypisuje pliki do sformatowania.

```powershell
uv run --locked ruff format .
uv run --locked ruff check .
uv run --locked ruff format --check .
uv run --locked python -m pytest -q tests
```

Po zmianie formatu przejrzyj diff i ponownie uruchom testy. W kopii Poetry odpowiednikiem byloby `poetry run ruff check .`. Ruff korzysta z konfiguracji projektu, ale nie jest zalezny od tego, kto zainstalowal biblioteki.

## 5. Docker: obraz, kontener i dane

`.venv` rozwiazuje problem bibliotek Pythona. Nie uruchamia PostgreSQL i nie pakuje calego srodowiska z systemem, w ktorym startuje kod. W Dockerze budujemy **obraz** z Pythonem, bibliotekami i plikami projektu. **Kontener** jest uruchomieniem tego obrazu. **Docker Compose** pozwala opisac kilka kontenerow, ktore maja razem pracowac.

W tym cwiczeniu uruchomimy jednorazowy ETL i osobny kontener PostgreSQL. Nie potrzebujemy FastAPI ani dostepu do PostgreSQL z komputera. Nie zmieniamy lokalnej bazy z lekcji 04. Zainstaluj i uruchom Docker Desktop z kontenerami Linux. W terminalu sprawdz `docker version` i `docker compose version`.

Dodaj w glownym katalogu plik `Dockerfile`:

```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:0.11.29 /uv /bin/uv

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev

COPY main.py pipeline.py config.py ingestion.py loading.py db.py storage.py transformations.py ./
ENTRYPOINT ["/app/.venv/bin/python", "main.py"]
```

`FROM` wybiera obraz z Pythonem, a pierwsze `COPY` dodaje `uv`. Najpierw kopiujemy tylko pliki zaleznosci i instalujemy wedlug locka. Docker moze dzieki temu wykorzystac poprzednia instalacje, gdy zmieni sie tylko plik `.py`. `--no-dev` pomija narzedzia takie jak Ruff i pytest. `ENTRYPOINT` mowi, co uruchomic w kontenerze.

Dodaj `.dockerignore`:

```text
.env
.env.*
.venv/
.git/
data/
__pycache__/
.pytest_cache/
```

Nie chcemy wysylac hasel ani lokalnych danych do procesu budowania obrazu. Nie kopiujemy tez `.venv` z Windows: kontener tworzy wlasne srodowisko pod Linuxem.

Zbuduj obraz i sprawdz go bez laczenia z baza:

```powershell
docker build -t nasa-etl .
docker run --rm --entrypoint /app/.venv/bin/python nasa-etl --version
```

Drugie polecenie na chwile zastepuje domyslne uruchomienie i pokazuje wersje Pythona z obrazu. `--rm` usuwa ten jednorazowy kontener po wykonaniu polecenia, ale nie usuwa obrazu.

## 6. Uruchom ETL i osobna baze przez Compose

Utworz `sql/init.sql` z **`CREATE SCHEMA` i `CREATE TABLE`** z [lekcji 04](04_etl_nasa_postgresql.md). Nie kopiuj `CREATE DATABASE`: kontener utworzy baze z konfiguracji.

Obok `Dockerfile` utworz `.env.docker`. Przykladowe haslo zastap swoim, uzywanym tylko w tym cwiczeniu. Tego pliku nie dodawaj do Git.

```dotenv
DB_HOST=db
DB_PORT=5432
DB_NAME=nasa_events_lab
DB_USER=nasa
DB_PASSWORD=change_me_locally
```

W Compose `db` jest nazwa uslugi z PostgreSQL. Z perspektywy kontenera ETL `127.0.0.1` oznaczaloby sam kontener ETL, nie baze. Zapisz `compose.yaml` obok `Dockerfile`:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_DB: nasa_events_lab
      POSTGRES_USER: nasa
      POSTGRES_PASSWORD: ${DB_PASSWORD:?Set DB_PASSWORD in .env.docker}
    healthcheck:
      test: ["CMD", "pg_isready", "-h", "127.0.0.1", "-U", "nasa", "-d", "nasa_events_lab"]
      interval: 2s
      timeout: 3s
      retries: 15
    volumes:
      - pgdata:/var/lib/postgresql
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
  etl:
    build: .
    image: nasa-etl
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./.env.docker:/app/.env:ro
      - ./data:/app/data

volumes:
  pgdata:
```

- Compose odczytuje `DB_PASSWORD` z `--env-file .env.docker` i przekazuje je do PostgreSQL. Ten sam plik podlaczamy tylko do odczytu (`:ro`) jako `/app/.env`, bo takiego pliku szuka nasz `config.py`.
- `healthcheck` sprawdza, czy PostgreSQL przyjmuje polaczenia. `init.sql` wykona sie przy pierwszym uruchomieniu z pustym wolumenem. Na istniejacej bazie zmiana tego pliku nie uruchomi go ponownie.
- `pgdata` zachowuje dane bazy poza kontenerem. `./data:/app/data` oznacza, ze surowy JSON pojawi sie tez w katalogu projektu. Nie publikujemy portu PostgreSQL na komputerze.

```powershell
New-Item -ItemType Directory -Path data -Force | Out-Null
docker compose --env-file .env.docker up -d --wait db
docker compose --env-file .env.docker run --rm --build etl
docker compose --env-file .env.docker exec db psql -U nasa -d nasa_events_lab -c "SELECT COUNT(*) FROM eonet.events;"
docker compose --env-file .env.docker down
```

Po udanym przebiegu zobaczysz logi z 06, plik `data/raw/eonet_events.json` i liczbe wierszy z `psql`. `down` zatrzymuje projekt i usuwa kontenery oraz jego siec. W tej komendzie celowo nie ma `-v`, zeby nie usuwac danych z wolumenu.

### Samodzielnie

Przed `down` uruchom ETL drugi raz i ponow zapytanie `COUNT(*)`. Porownaj ID z raw przed i po uruchomieniu. Jesli NASA zwrocila te same ID, liczba wierszy nie powinna wzrosnac. W `README.md` projektu zapisz komendy do synchronizacji, lintowania, testow i uruchomienia Compose. Podaj nazwy potrzebnych zmiennych, nie ich wartosci.

## Dokumentacja

- [uv: praca z projektem](https://docs.astral.sh/uv/guides/projects/) i [uv w Dockerze](https://docs.astral.sh/uv/guides/integration/docker/)
- [Poetry: podstawy](https://python-poetry.org/docs/basic-usage/) i [grupy zaleznosci](https://python-poetry.org/docs/managing-dependencies/)
- [Ruff: konfiguracja](https://docs.astral.sh/ruff/configuration/) i [formatowanie](https://docs.astral.sh/ruff/formatter/)
- [Docker Compose: kolejnosc startu](https://docs.docker.com/compose/how-tos/startup-order/) i [oficjalny obraz PostgreSQL](https://hub.docker.com/_/postgres)
