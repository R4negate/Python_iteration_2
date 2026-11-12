# 07. Testy pipeline'u w Pythonie: od pierwszego assert do pytest

[Zadanie: zbuduj pipeline](../zadania/07_pipeline_testy.md)

Wyobraz sobie, ze zmieniasz jedna linie w `normalize_event()`. Pipeline nadal sie uruchamia, ale nagle zaczyna odrzucac otwarte zdarzenia, bo nie maja daty zamkniecia. Nie chcemy sprawdzac tego recznie po kazdej zmianie. **Test automatyczny** uruchamia kod z przygotowanymi danymi i porownuje wynik z tym, czego oczekujemy.

Pracujemy w projekcie `nasa_pipeline` po lekcji 06. Dane w testach to krotkie, wymyslone slowniki. To **nie jest tryb offline ETL**: normalne `python main.py` nadal pobiera dane z NASA. Testy podstawowe nie lacza sie z NASA i nie pisza do bazy.

## 1. Napisz i uruchom pierwszy test

W katalogu projektu zainstaluj `pytest`:

```powershell
python -m pip install pytest
```

Utworz katalog `tests` obok `main.py`. Dodaj `tests/test_transformations.py`:

```python
from transformations import normalize_event


def test_normalize_event_strips_text():
    raw = {
        "id": " TEST_01 ",
        "title": " Pozar lasu ",
        "categories": [{"title": " Wildfires "}],
        "closed": None,
    }

    event = normalize_event(raw)

    assert event.event_id == "TEST_01"
    assert event.title == "Pozar lasu"
    assert event.categories == ["Wildfires"]
    assert event.closed_at is None
```

To trzy kroki: **przygotuj dane** (Arrange), **uruchom funkcje** (Act), **sprawdz wynik** (Assert). `assert` podaje warunek, ktory musi byc prawdziwy. Nie piszemy tylko `assert event`: wtedy test nie zauwazylby np. zlego tytulu. W JSON `null` oznacza brak wartosci, w slowniku Pythona uzywamy `None`.

```powershell
python -m pytest -v
```

Uruchom komende z katalogu projektu. `pytest` sam szuka m.in. plikow `test_*.py` i funkcji `test_*`; nie wywolujemy ich w `main.py`. Powinno wyjsc `1 passed`. Zmien na chwile oczekiwany tytul na `"Inny tytul"`. Uruchom test ponownie, przeczytaj porownanie wartosci przy `AssertionError` i przywroc prawdziwe oczekiwanie. Nie poprawiaj kodu pipeline'u tylko po to, zeby pasowal do celowo blednego testu.

Mozesz uruchomic tylko plik (`python -m pytest tests/test_transformations.py -v`), wyszukac testy po nazwie (`python -m pytest -k normalize`), zatrzymac sie na pierwszym bledzie (`-x`) albo skrocic opis bledu (`--tb=short`).

## 2. Wspolne dane, bledy i kilka przypadkow

Ten sam poprawny slownik bedzie potrzebny w kilku testach. Zamiast go kopiowac, tworzymy **fixture**, czyli funkcje przygotowujaca dane do testu. Dodaj `tests/conftest.py`:

```python
import pytest


@pytest.fixture
def raw_event():
    return {
        "id": " TEST_01 ",
        "title": " Pozar lasu ",
        "categories": [{"title": " Wildfires "}],
        "closed": None,
    }
```

`pytest` automatycznie znajduje `conftest.py`. Gdy test ma argument `raw_event`, `pytest` uruchomi fixture i poda jej wynik. Nie importujemy tego pliku. Domyslnie fixture powstaje od nowa dla kazdego testu. Dzieki temu zmiana tytulu w jednym tescie nie zmieni danych nastepnego.

Zakres fixture (ang. `scope`) mowi, jak dlugo mozna uzywac jednego wyniku. `function` to jeden test, `module` to jeden plik testow, `session` to cale uruchomienie `pytest`. Na razie uzywamy domyslnego `function`, bo slownik mozna zmieniac. Fixture z `yield` moze przygotowac zasob przed testem i posprzatac po nim; przy zwyklym slowniku wystarcza `return`.

Zastap `tests/test_transformations.py`:

```python
from datetime import datetime, timezone

import pytest

from transformations import normalize_event, transform_events


def test_normalize_event_strips_text(raw_event):
    event = normalize_event(raw_event)

    assert event.event_id == "TEST_01"
    assert event.title == "Pozar lasu"
    assert event.categories == ["Wildfires"]
    assert event.closed_at is None
    assert raw_event["title"] == " Pozar lasu "


def test_normalize_event_requires_title(raw_event):
    del raw_event["title"]

    with pytest.raises(ValueError, match="title:"):
        normalize_event(raw_event)


@pytest.mark.parametrize("bad_categories", [[], None], ids=["empty", "null"])
def test_normalize_event_rejects_bad_categories(raw_event, bad_categories):
    raw_event["categories"] = bad_categories

    with pytest.raises(ValueError, match="categories:"):
        normalize_event(raw_event)


def test_normalize_event_parses_closed_date(raw_event):
    raw_event["closed"] = "2024-08-09T00:00:00Z"

    event = normalize_event(raw_event)

    assert event.closed_at == datetime(2024, 8, 9, tzinfo=timezone.utc)


def test_transform_events_keeps_valid_rows_and_rejects_others(raw_event):
    broken = {**raw_event, "id": "TEST_02", "title": ""}
    next_event = {**raw_event, "id": "TEST_03"}

    clean, rejected = transform_events([raw_event, broken, raw_event, next_event])

    assert [event.event_id for event in clean] == ["TEST_01", "TEST_03"]
    assert [row["position"] for row in rejected] == [2, 3]
    assert "title:" in rejected[0]["reason"]
    assert "Duplicate id" in rejected[1]["reason"]
```

`pytest.raises(ValueError)` oznacza, ze **blad jest oczekiwanym wynikiem**. Test nie przejdzie, jesli kod w ogole nie zglosi bledu. `match` pilnuje dodatkowo, ze komunikat dotyczy tego pola, o ktore nam chodzilo. `parametrize` uruchamia ten sam test osobno dla kazdej wartosci. Tu dostajemy dwa przypadki: pusta lista i `None`. W sumie po tym kroku powinno przejsc **6 testow**.

## 3. Sprawdz zapis pliku bez dotykania `data/raw`

W 06 zapis JSON jest w srodku `run_pipeline()`. Chcemy go sprawdzic bez bazy i bez sieci, wiec przeniesiemy zapis do osobnej funkcji. Dodaj `storage.py` obok `pipeline.py`:

```python
import json
from pathlib import Path


def save_raw_snapshot(payload: dict, data_dir: Path) -> Path:
    raw_path = data_dir / "raw" / "eonet_events.json"
    raw_path.parent.mkdir(parents=True, exist_ok=True)
    with raw_path.open("w", encoding="utf-8") as file:
        json.dump(payload, file, ensure_ascii=False, indent=2)
    return raw_path
```

W `pipeline.py` usun `import json` i dodaj `from storage import save_raw_snapshot` przy pozostalych importach projektu. Zastap fragment od `raw_path = data_dir / ...` do `json.dump(...)` jedna linia `raw_path = save_raw_snapshot(payload, data_dir)`, na tym samym wcieciu. Log `logger.debug(...)` pod nia zostaje. Plik nadal ma te sama nazwe i zawartosc.

Dodaj `tests/test_storage.py`:

```python
import json

from storage import save_raw_snapshot


def test_save_raw_snapshot_writes_original_payload(tmp_path, raw_event):
    payload = {"events": [raw_event], "title": "Test"}

    saved = save_raw_snapshot(payload, tmp_path)

    assert saved == tmp_path / "raw" / "eonet_events.json"
    assert json.loads(saved.read_text(encoding="utf-8")) == payload
```

`tmp_path` to gotowa fixture z `pytest`. Kazdy test, ktory o nia poprosi, dostaje wlasny katalog tymczasowy. Sprawdzamy nie tylko istnienie pliku, ale tez to, czy po odczytaniu ma te same dane. Nie zalezymy od pliku z poprzedniego przebiegu.

## 4. Sprawdz klienta HTTP bez wysylania zapytania

Chcemy uruchomic prawdziwa funkcje `load_payload()`, ale z gory wybrac, co "odpowie NASA". Na czas testu zastapimy tylko `requests.get` w miejscu, w ktorym uzywa go nasz kod. Taka podmiane nazywamy **mockowaniem**.

Dodaj `tests/test_ingestion.py`:

```python
from unittest.mock import MagicMock, patch

import pytest
from requests import HTTPError

from ingestion import load_payload


def test_load_payload_reads_events():
    response = MagicMock()
    response.json.return_value = {"events": []}

    with patch("ingestion.requests.get", return_value=response) as get:
        result = load_payload()

    assert result == {"events": []}
    get.assert_called_once_with(
        "https://eonet.gsfc.nasa.gov/api/v3/events",
        params={"status": "all", "start": "2024-08-01", "end": "2024-08-31", "limit": 5},
        timeout=10,
    )
    response.raise_for_status.assert_called_once_with()


def test_load_payload_does_not_parse_http_error():
    response = MagicMock()
    response.raise_for_status.side_effect = HTTPError("503 from source")

    with patch("ingestion.requests.get", return_value=response):
        with pytest.raises(HTTPError):
            load_payload()

    response.json.assert_not_called()
```

`MagicMock` to zastepczy obiekt, ktory moze oddac ustalona wartosc i zapamietac wywolania. `return_value` mowi "zwroc to", a `side_effect` w tym przykladzie mowi "zglos ten blad". `patch` wstawia zastepstwo na czas bloku `with`, a potem przywraca prawdziwa funkcje. Wybieramy nazwe tak, jak widzi ja testowany modul: w `ingestion.py` to `requests.get`, wiec uzywamy `ingestion.requests.get`. Nie podmieniamy samego `load_payload()`, bo wtedy nie sprawdzilibysmy jego pracy.

Gdy nie potrzebujesz sprawdzac wywolan, czasem wystarczy samodzielnie napisana funkcja albo klasa z kilkoma metodami, czyli **fake**. Dodatek `pytest-mock` daje fixture `mocker`. Po `python -m pip install pytest-mock` mozna w dodatkowym tescie uzyc `get = mocker.patch("ingestion.requests.get")`, ustawic `get.return_value.json.return_value` i wykonac te same sprawdzenia. Fixture sama cofa podmiane po tescie. Nie ma potrzeby pisac trzech kopii kazdego testu.

## 5. Sprawdz, czy kroki pipeline'u wspolpracuja

W tym tescie uzyjemy prawdziwego czyszczenia i prawdziwego zapisu pliku do `tmp_path`. Zastapimy tylko pobranie z sieci i zapis do PostgreSQL. Dodaj `tests/test_pipeline.py`:

```python
import json

from config import DatabaseConfig
from pipeline import run_pipeline


def test_pipeline_saves_raw_and_sends_only_clean_rows(tmp_path, raw_event, monkeypatch):
    broken = {**raw_event, "id": "TEST_02", "title": ""}
    payload = {"events": [raw_event, broken]}
    sent_ids = []
    config = DatabaseConfig("unused", 5432, "unused", "unused", "")

    def fake_load_events(events, db_config):
        assert db_config == config
        sent_ids.extend(event.event_id for event in events)

    monkeypatch.setattr("pipeline.load_payload", lambda: payload)
    monkeypatch.setattr("pipeline.load_events", fake_load_events)

    loaded, rejected = run_pipeline(tmp_path, config)

    assert loaded == 1
    assert [row["position"] for row in rejected] == [2]
    assert sent_ids == ["TEST_01"]
    raw_path = tmp_path / "raw" / "eonet_events.json"
    assert json.loads(raw_path.read_text(encoding="utf-8")) == payload
```

`monkeypatch` to fixture z `pytest`, ktora po tescie cofa podmiany. `pipeline.py` robi `from loading import load_events`, wiec w tescie podmieniamy `pipeline.load_events`. Slownik z bledem ma byc w raw, ale nie ma trafic do funkcji zapisujacej. Ten test nie dowodzi, ze SQL dziala: nie wykonuje prawdziwego SQL. Po tym kroku podstawowy zestaw to **10 testow**.

## 6. Opcjonalnie: jeden test z prawdziwym PostgreSQL

Test, ktory sprawdza wspolprace Pythona z baza, to **test integracyjny**. Przyda sie np. do sprawdzenia, czy dwa UPSERT-y tego samego ID zostawiaja jeden wiersz. Tego nie da sie sprawdzic przez samo zastapienie `load_events()`.

Utworz **osobna baze `nasa_events_test`** i wykonaj w niej DDL tabeli z [04](04_etl_nasa_postgresql.md). Test uzyje hosta i loginu z `.env`, ale zawsze podmieni nazwe bazy na `nasa_events_test`. Nie wlaczaj go, jesli nie przygotowales tej bazy. W katalogu projektu dodaj `pytest.ini`:

```ini
[pytest]
markers =
    integration: test wymagajacy osobnej bazy PostgreSQL
```

Dodaj `tests/test_postgres_integration.py`:

```python
import os
from dataclasses import replace
from uuid import uuid4

import pytest

from config import load_config
from db import open_connection
from loading import load_events
from transformations import Event


@pytest.mark.integration
@pytest.mark.skipif(os.getenv("RUN_DB_TESTS") != "1", reason="Requires nasa_events_test")
def test_upsert_updates_one_event():
    config = replace(load_config(), dbname="nasa_events_test")
    event_id = f"TEST_{uuid4().hex}"

    try:
        load_events([Event(event_id, "Pierwszy tytul", ["Wildfires"], None)], config)
        load_events([Event(event_id, "Nowy tytul", ["Wildfires"], None)], config)

        with open_connection(config) as conn:
            rows = conn.execute(
                "SELECT title FROM eonet.events WHERE event_id = %s", (event_id,)
            ).fetchall()

        assert rows == [("Nowy tytul",)]
    finally:
        with open_connection(config) as conn:
            conn.execute("DELETE FROM eonet.events WHERE event_id = %s", (event_id,))
```

`integration` oznacza grupe. `skipif` pomija test, gdy nie wlaczylismy go jawnie. Losowe ID chroni przed pomieszaniem przebiegow, a `finally` usuwa tylko wiersz utworzony przez ten test, takze przy nieudanym `assert`. Baza musi istniec przed uruchomieniem.

```powershell
python -m pytest -m "not integration"
$env:RUN_DB_TESTS = "1"
python -m pytest -m integration -v
Remove-Item Env:RUN_DB_TESTS
```

`skip` pomija test zawsze, np. przez tymczasowo niedostepne srodowisko. `xfail` oznacza znany problem: test sie wykonuje, ale spodziewamy sie porazki. Przy `strict=True` niespodziewane przejscie testu jest sygnalem, by usunac oznaczenie i zweryfikowac, czy problem zniknal. Nie uzywaj `skip` ani `xfail` do chowania zwyklych bledow.

## 7. Pokrycie kodu i kontrola jakosci

```powershell
python -m pip install pytest-cov
python -m pytest --cov=transformations --cov=ingestion --cov=storage --cov=pipeline --cov-report=term-missing
```

Pokrycie kodu (ang. `coverage`) mowi, ktore linie zostaly wykonane podczas testow. Kolumna `Missing` pokazuje linie, ktorych testy nie odwiedzily. Nie jest to ocena poprawnosci: mozna wykonac linie bez sprawdzenia ich wyniku. Testuj decyzje w kodzie, nie dzialanie biblioteki standardowej. Np. warto sprawdzic, ze brak `closed` jest bledem, choc `closed=None` jest poprawne.

### Samodzielnie

Dopisz dwa testy: brak klucza `closed` ma zglosic `ValueError`, a odpowiedz NASA bez listy `events` ma zostac odrzucona przez `load_payload()`. Drugi test nie moze wykonywac prawdziwego `GET`. Uruchom testy i sprawdz ponownie raport pokrycia.

## Dodatki: dane tabelaryczne i CI

### Gdy wynik jest DataFrame'em

W naszym pipeline wynikiem jest lista obiektow, wiec porownujemy ich pola. W przyszlym pipeline z Pandas albo Polars sprawdzisz takze kolumny, wiersze i typy. Ponizszy przyklad uzywa jednego oczyszczonego zdarzenia. To opcjonalny `tests/test_dataframes.py`; zainstaluj wybrana biblioteke, jesli chcesz go uruchomic.

```python
import pytest

from transformations import normalize_event


def test_pandas_event_row(raw_event):
    pd = pytest.importorskip("pandas")
    event = normalize_event(raw_event)
    actual = pd.DataFrame([{"event_id": event.event_id, "title": event.title}])
    expected = pd.DataFrame([{"event_id": "TEST_01", "title": "Pozar lasu"}])

    assert list(actual.columns) == ["event_id", "title"]
    pd.testing.assert_frame_equal(actual, expected)


def test_polars_event_row(raw_event):
    pl = pytest.importorskip("polars")
    from polars.testing import assert_frame_equal

    event = normalize_event(raw_event)
    actual = pl.DataFrame([{"event_id": event.event_id, "title": event.title}])
    expected = pl.DataFrame([{"event_id": "TEST_01", "title": "Pozar lasu"}])

    assert_frame_equal(actual, expected)
```

`importorskip()` pomija tylko test, ktory potrzebuje brakujacej biblioteki. Samo sprawdzenie kolumn lub liczby wierszy nie wykryje blednej wartosci. `assert_frame_equal` porownuje cala mala tabele. Tutaj to cwiczenie z porownywania wyniku, nie powod do zmiany pipeline'u na DataFrame.

### CI: uruchamianie testow przez GitHub

Gdy projekt jest w repozytorium GitHub, zapisz ponizszy przyklad jako `.github/workflows/tests.yml`. Workflow uruchamia testy na push i pull request, na osobnej maszynie. Nie przekazujemy mu hasla do bazy. Test integracyjny, jesli go dodano, sam sie pominie bez `RUN_DB_TESTS`. Gdy opanujesz [Ruff z lekcji 08](08_uv_poetry_ruff_docker.md), mozesz dolozyc go przed testami.

```yaml
name: Python tests
on: [push, pull_request]
permissions:
  contents: read
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: python -m pip install requests "psycopg[binary]" python-dotenv pytest
      - run: python -m pytest -v
```

W projekcie zespolowym zamiast listy pakietow w workflow lepiej uzyc tego samego pliku zaleznosci, z ktorego korzysta zespol lokalnie. Jezeli dodasz testy uzywajace `mocker`, dopisz do zaleznosci testowych `pytest-mock`. GitHub Actions wykonuje polecenia; nie zastapi uruchomienia i przeczytania testow podczas pracy.

## Dokumentacja i inspiracja

- [Pytest: pierwsze kroki](https://docs.pytest.org/en/stable/getting-started.html), [fixture](https://docs.pytest.org/en/stable/how-to/fixtures.html) i [parametryzacja](https://docs.pytest.org/en/stable/how-to/parametrize.html)
- [Python: unittest.mock](https://docs.python.org/3/library/unittest.mock.html), [pytest-mock](https://pytest-mock.readthedocs.io/en/latest/) i [pytest-cov](https://pytest-cov.readthedocs.io/en/latest/)
- [Pandas: assert_frame_equal](https://pandas.pydata.org/docs/reference/api/pandas.testing.assert_frame_equal.html) i [Polars: assert_frame_equal](https://docs.pola.rs/api/python/stable/reference/api/polars.testing.assert_frame_equal.html)
- [GitHub Actions: testowanie Pythona](https://docs.github.com/en/actions/tutorials/build-and-test-code/python)
- Zakres porownano z modulem Python for Data Engineering, Week 02, Day 3 w [DataCee](https://www.datacee.com/). Przyklady sprawdzaja nasze funkcje z projektu NASA EONET.
