# 07. Testy od podstaw: po co, co i jak testujemy w pipeline

[Zadanie: zbuduj pipeline](../zadania/07_pipeline_testy.md)

Wyobraz sobie, ze zmieniasz jedna linie w `normalize_event()`. Pipeline nadal sie uruchamia, ale nagle zaczyna odrzucac otwarte zdarzenia, bo nie maja daty zamkniecia. Nie chcemy sprawdzac tego recznie po kazdej zmianie. **Test automatyczny** uruchamia kod z przygotowanymi danymi i porownuje wynik z tym, czego oczekujemy.

Pracujemy w projekcie `nasa_pipeline` po lekcji 06. Dane w testach to krotkie, wymyslone slowniki. To **nie jest tryb offline ETL**: normalne `python main.py` nadal pobiera dane z NASA. Testy podstawowe nie lacza sie z NASA i nie pisza do bazy.


## 1. Po co sa testy?

Program moze zakonczyc sie bez wyjatku, a mimo to zwrocic zly wynik. Wyobraz sobie, ze funkcja ma oczyscic tytul `" Pozar lasu "`. Jesli odda go razem ze spacjami, Python sie nie zatrzyma. Z punktu widzenia interpretera to nadal poprawny tekst. Z punktu widzenia naszej reguly danych wynik jest niepoprawny.

**Test to kod, ktory uruchamia wybrany fragment programu i sprawdza konkretne oczekiwanie.** Oczekiwanie bierze sie z wymagania, a nie z tego, co akurat zwraca funkcja. Jesli ustalilismy, ze usuwamy spacje z brzegow, test powinien wymagac `"Pozar lasu"`.

Testy przydaja sie w trzech sytuacjach:

- Piszesz nowa funkcje: sprawdzasz, czy robi to, co zaplanowales.
- Zmieniasz istniejacy kod: sprawdzasz, czy nie zepsules wczesniej dzialajacych przypadkow.
- Trafiasz na blad: dodajesz przypadek, ktory go odtwarza, poprawiasz kod i zostawiasz test jako zabezpieczenie.

Na tym polega wykrywanie **regresji**: zmiana rozwiazala jeden problem, ale zepsula cos, co wczesniej dzialalo. Np. poprawienie odczytu daty nie powinno spowodowac odrzucania otwartych zdarzen z `closed=None`.

### Test reczny a automatyczny

Recznie wywolujesz funkcje, patrzysz na wynik i sam oceniasz, czy jest dobry. To przydatne przy poznawaniu danych. Test automatyczny zawiera juz zapisane oczekiwanie: komputer moze je sprawdzic za kazdym uruchomieniem, bez ponownego ogladania wynikow przez czlowieka.

`print(event.title)` tylko pokazuje wartosc. Nie decyduje, czy ta wartosc jest poprawna. `assert event.title == "Pozar lasu"` sprawdza ustalony warunek i sygnalizuje jego niespelnienie.

### Dlaczego logi nie zastepuja testow?

Log z lekcji 06 moze powiedziec: „zapisano 5 zdarzen”. Nie dowodzi, ze kazde ma poprawny tytul. Log opisuje przebieg uruchomienia. Test sprawdza ustalone zachowanie. Potrzebujemy obu: testow przed zmiana i wdrozeniem oraz logow podczas rzeczywistej pracy.

Zielone testy nie sa dowodem, ze program nie ma zadnych bledow. Oznaczaja, ze **sprawdzane przez te testy przypadki** spelnily oczekiwania. Jesli nie sprawdzilismy pustej listy, sam zielony wynik nie mowi, jak program ja obsluguje.

## 2. Jakie sa rodzaje testow?

Najpierw podzielmy je wedlug tego, **jak duzy fragment systemu uruchamiamy**.

| Rodzaj | Co sprawdza? | Przyklad w naszym projekcie |
| --- | --- | --- |
| Jednostkowy, ang. unit test | Mala jednostke logiki w odizolowaniu od zewnetrznych systemow. | `normalize_event()` usuwa spacje i rozpoznaje `closed=None`. |
| Integracyjny | Wspolprace rzeczywistych czesci programu lub programu z zaleznoscia. | `load_events()` wykonuje SQL na osobnej bazie testowej; ponowny zapis ID aktualizuje jeden wiersz. |
| End-to-end, skrot E2E | Caly wybrany przeplyw przez rzeczywiste warstwy, od wejscia do wyniku. | Uruchomienie ETL z prawdziwym API i testowa baza, potem sprawdzenie wyniku w SQL. |

Granice nazw moga roznic sie miedzy zespolami. Wazniejsze jest umiec dokladnie wskazac, co test uruchamia naprawde, a co zastepuje. Test naszego pipeline'u z prawdziwa transformacja i zapisem do tymczasowego pliku, ale bez HTTP i SQL, **nie jest pelnym testem E2E**.

Test jednostkowy zwykle jest szybki i latwo wskazuje przyczyne bledu. Test integracyjny wymaga przygotowania zaleznosci, np. tabeli. E2E sprawdza wiecej naraz, ale moze upasc przez niedostepnosc API, konfiguracje bazy lub blad kodu. Nie zaczynamy od samych E2E. Najpierw testujemy reguly, potem wspolprace warstw, a dopiero na koncu calosc.

### A testy danych i testy regresji?

To inny sposob podzialu: wedlug **celu**, nie wielkosci.

- **Test jakosci danych** sprawdza np. unikalnosc ID, wymagane pola lub liczbe odrzucen. Zapytania kontrolne z lekcji 06 tez sprawdzaly jakosc danych.
- **Test regresji** pilnuje, zeby dawniej poprawne zachowanie nadal dzialalo po zmianie.

Test jednostkowy moze byc jednoczesnie testem regresji. Nie sa to rozlaczne szufladki.

## 3. Zanim napiszemy kod testu: ustalamy przypadki

Test musi odpowiadac na pytanie. Dla `normalize_event()` zapisujemy wymagania i na ich podstawie dobieramy dane:

| Przypadek | Oczekiwany wynik |
| --- | --- |
| Poprawny tytul ze spacjami | Tytul bez brzegowych spacji. |
| Pole `closed` o wartosci `None` | Poprawne otwarte zdarzenie, `closed_at is None`. |
| Brak klucza `closed` | `ValueError`; brak pola nie oznacza automatycznie otwartego zdarzenia. |
| Brak, `None` lub same spacje w tytule | `ValueError` dotyczacy tytulu. |
| Pusta lista kategorii | `ValueError` dotyczacy kategorii. |
| Pusta partia zdarzen | Dwie puste listy: poprawne i odrzucone. |
| Dwa poprawne rekordy z tym samym ID w partii | Pierwszy zostaje, drugi jest odrzucony. |

Przyklad z tytulem to **typowy przypadek**. Pusta partia to **przypadek brzegowy**. Brak wymaganego pola to **przypadek niepoprawnych danych**. Wszystkie sa potrzebne. Testowanie tylko poprawnego zdarzenia nie sprawdzi, czy walidacja odrzuca zle dane.

Nie wyliczamy oczekiwanego wyniku przez ponowne wywolanie testowanej funkcji. `expected = normalize_event(raw)` i porownanie go z `normalize_event(raw)` nie sprawdzi reguly oczyszczania. Oczekiwany tytul zapisujemy niezaleznie: `"Pozar lasu"`.

## 4. Co robi assert?

`assert` jest instrukcja Pythona. Sprawdza warunek:

```python
actual = 2 + 3
assert actual == 5
```

Gdy warunek jest prawdziwy, program przechodzi dalej. Nie wypisuje sam z siebie „OK”. Gdy warunek jest falszywy, Python zglasza `AssertionError`:

```python
actual = 2 + 3
assert actual == 6
```

Drugi przyklad **ma zakonczyc sie bledem**. Jest tu po to, zeby zobaczyc, czym rozni sie pokazanie wyniku od sprawdzenia wyniku.

W testach uzywamy np. `assert event.closed_at is None` albo `assert clean == []`. Sprawdzamy wartosc, ktora ma znaczenie dla wymagania. Samo `assert event` nie sprawdzi tytulu ani kategorii.

`assert` jest tu narzedziem testu, nie zamiennikiem walidacji danych przychodzacych z API. W kodzie ETL pozostawiamy `if ...: raise ValueError(...)`. Asercje Pythona moga byc wylaczone przez uruchomienie z optymalizacja `-O`; testy uruchamiamy normalnie, bez tej opcji.

## 5. Given - When - Then: jak ulozyc test?

Kazdy test powinien miec czytelne trzy czesci:

1. **Given**: przygotuj dane i warunki poczatkowe.
2. **When**: wykonaj operacje, ktora sprawdzasz.
3. **Then**: sprawdz oczekiwany wynik.

Mozesz spotkac nazwy **Arrange - Act - Assert**. To podobny uklad: przygotuj, wykonaj, sprawdz. Sa to konwencje pisania kodu, nie slowa kluczowe Pythona.

Test funkcji `required_text()` z lekcji 04 wyglada tak:

```python
from transformations import required_text


def test_required_text_removes_surrounding_spaces():
    # Given
    text = "  Pozar lasu  "

    # When
    result = required_text(text, "title")

    # Then
    assert result == "Pozar lasu"
```

`def` definiuje zwykla funkcje bez parametru `self`, bo nie jest to metoda klasy. Nazwa zaczyna sie od `test_`, a dalsza czesc opisuje zachowanie. Komentarze oddzielaja etapy. `result` jest wynikiem rzeczywistego wywolania, a `"Pozar lasu"` naszym niezaleznym oczekiwaniem.

Na razie przeczytaj przyklad. W kolejnym kroku skonfigurujemy narzedzie, ktore znajdzie takie funkcje i je uruchomi. Testu nie wywolujemy z produkcyjnego `main.py`.

## 6. Czym jest pytest i gdzie trzymamy testy?

`pytest` to biblioteka do uruchamiania testow. Wyszukuje testy, wykonuje je osobno i pokazuje podsumowanie oraz miejsce bledu. Nie musimy pisac wlasnej petli uruchamiajacej kazda funkcje.

Przyklady ponizej odpowiadaja **ukladowi z lekcji 04 i 06**: moduly sa obok `main.py`, a nie wewnatrz drugiego folderu `nasa_pipeline`.

```text
nasa_pipeline/
    main.py
    config.py
    db.py
    ingestion.py
    loading.py
    pipeline.py
    transformations.py
    tests/
        conftest.py
        test_transformations.py
        test_storage.py
        test_ingestion.py
        test_pipeline.py
        test_postgres_integration.py
```

W tej lekcji utworzymy te pliki po kolei. `storage.py` pojawi sie pozniej, kiedy wydzielimy zapis raw.

**Jesli masz juz podfolder-pakiet `nasa_pipeline/`**, nie przenos go z powrotem. Trzymaj `tests/` w katalogu glownym obok `main.py`. W testach stosuj wtedy np. `from nasa_pipeline.transformations import normalize_event`, a podmiany kieruj do `nasa_pipeline.ingestion.requests.get` i `nasa_pipeline.pipeline.load_events`. Importy wewnatrz pakietu musza byc poprawne, np. `from .loading import load_events`. Dodaj `__init__.py` w podfolderze pakietu. Zmieniasz sciezke importu, nie logike testu. Nie dodajemy `sys.path.append(...)`, zeby maskowac niepoprawne importy.

**Jesli na zajeciach masz `load_payload(url, params)`**, w testach tez przekazujesz `url` i `params`. Przyklad ponizej ma bezargumentowa wersje z lekcji 03. Podobnie `run_pipeline(data_dir, config)` zwraca tu liczbe i odrzucenia, a nie liste `clean`. Test musi odpowiadac faktycznemu kontraktowi funkcji. Nie zmieniamy dzialajacego ETL tylko po to, zeby pasowal do innej sygnatury z przykladu.

## 7. Co robi dobry test i czego unikamy?

Dobry test ma jasny cel, male dane, konkretne oczekiwanie i wynik niezalezny od innych testow. Zwykle jedna funkcja testowa opisuje jedno zachowanie; moze miec kilka asercji, jesli razem sprawdzaja ten sam wynik.

Nie opieramy podstawowych testow na aktualnej odpowiedzi NASA, bo jutro dane moga byc inne. Nie czekamy w nich na prawdziwe `time.sleep()`, nie zapisujemy do szkoleniowego `data/raw` i nie korzystamy ze szkoleniowej tabeli jako miejsca na dane testowe. W dalszych krokach poznasz zastepstwa odpowiedzi HTTP oraz katalog tymczasowy.

Nie ukrywamy blednego testu przez `try/except: pass`. Gdy oczekujemy wyjatku, sprawdzimy go przez `pytest.raises`. Nie zmieniamy tez oczekiwania na dowolna wartosc tylko po to, zeby test byl zielony: najpierw ustalamy, czy blad jest w kodzie, tescie czy nieaktualnym wymaganiu.

## 8. Napisz i uruchom pierwszy test

W katalogu projektu zainstaluj `pytest`:

```powershell
python -m pip install pytest
```

Utworz katalog `tests` obok `main.py`. Dodaj `tests/test_transformations.py`:

```python
from transformations import normalize_event


def test_normalize_event_strips_text():
    # Given: poprawne zdarzenie ze spacjami
    raw = {
        "id": " TEST_01 ",
        "title": " Pozar lasu ",
        "categories": [{"title": " Wildfires "}],
        "closed": None,
    }

    # When: wykonanie prawdziwej funkcji
    event = normalize_event(raw)

    # Then: porownanie z wymaganym wynikiem
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


### Jak czytac wynik pytest?

- `PASSED`: asercje i wykonanie tego przypadku zakonczyly sie poprawnie.
- `FAILED`: przypadek nie spelnil oczekiwania albo podczas jego wykonania wystapil nieoczekiwany blad.
- `ERROR`: np. nie udalo sie zaimportowac modulu lub przygotowac fixture; test mogl nawet nie dojsc do wywolania sprawdzanej funkcji.
- `SKIPPED`: przypadek zostal pominiety. Nie traktujemy tego jako potwierdzenia poprawnosci.

Gdy widzisz `assert 'Pozar lasu' == 'Inny tytul'`, czytasz porownanie wyniku z oczekiwaniem zapisanym w tescie. Najpierw znajdz nazwe testu i linie bledu, potem sprawdz Given, When i Then. Blad importu nie oznacza, ze funkcja zle czysci dane: trzeba najpierw poprawic sposob uruchomienia lub import.

Opcja `-v` pokazuje nazwy poszczegolnych przypadkow. Bez niej pytest czesto wypisuje kropki dla poprawnych testow i litere `F` dla nieudanych. Jesli kolekcja zwraca `collected 0 items`, sprawdz nazwe pliku i funkcji oraz katalog, z ktorego uruchamiasz komende. Zero znalezionych testow nie oznacza, ze kod zostal sprawdzony.

## 9. Wspolne dane, bledy i kilka przypadkow

Przy pierwszym tescie dane byly wewnatrz funkcji. Teraz rozdzielimy trzy nowe mechanizmy: przygotowanie danych przez fixture, sprawdzanie wyjatkow i uruchamianie jednej logiki dla kilku wejsc. Najpierw powstaje fixture, potem korzystajace z niej testy. Nie zmieniamy funkcji ETL.

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


### Jak czytac test bledu bez zgadywania?

W `test_normalize_event_requires_title` przygotowanie polega na usunieciu klucza `title` z poprawnego slownika. Usuwamy tylko jedna rzecz, zeby wiedziec, ktora regula jest sprawdzana. Blok `with pytest.raises(...)` obejmuje wywolanie funkcji, czyli krok When. Sam kontekst sprawdza Then: funkcja ma zglosic wskazany typ wyjatku.

Gdy wyjatku nie bedzie, zobaczysz `DID NOT RAISE`: kod przepuscil zle dane. Gdy pojawi sie inny wyjatek, test tez sie nie powiedzie. `match="title:"` sprawdza fragment komunikatu jako wyrazenie regularne. W tym prostym wzorcu dwukropek jest zwyklym znakiem.

### Parametryzacja zamiast trzech kopii tego samego testu

Dekorator `@pytest.mark.parametrize` podaje kolejne wartosci argumentu testu. To nie jest petla napisana wewnatrz testu: pytest tworzy osobny przypadek dla kazdej wartosci. Dzieki temu od razu widac, ktory wariant zawiodl. `ids` ustala czytelne etykiety zamiast technicznych nazw.

Dopisz do tego samego `tests/test_transformations.py`:

```python
@pytest.mark.parametrize("title", ["", "   ", None], ids=["empty", "spaces", "null"])
def test_normalize_event_rejects_empty_title(raw_event, title):
    # Given
    raw_event["title"] = title

    # When / Then: oczekujemy odrzucenia rekordu
    with pytest.raises(ValueError, match="title:"):
        normalize_event(raw_event)


@pytest.mark.parametrize(
    "closed",
    ["not-a-date", "", "2024-08-09T00:00:00"],
    ids=["invalid", "empty", "no-timezone"],
)
def test_normalize_event_rejects_invalid_closed(raw_event, closed):
    raw_event["closed"] = closed

    with pytest.raises(ValueError, match="closed:"):
        normalize_event(raw_event)


def test_normalize_event_requires_closed_key(raw_event):
    del raw_event["closed"]

    with pytest.raises(ValueError, match="Missing closed"):
        normalize_event(raw_event)


def test_transform_events_handles_empty_batch():
    clean, rejected = transform_events([])

    assert clean == []
    assert rejected == []
```

Teraz plik transformacji obejmuje **14 przypadkow testowych**, mimo ze definicji funkcji jest mniej. Test brakujacego `closed` i test `closed=None` sa celowo osobne: „brak pola” i „znana informacja, ze zdarzenie nie jest zamkniete” nie znacza tego samego.

## 10. Sprawdz zapis pliku bez dotykania `data/raw`

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

W tym tescie **naprawde zapisujemy plik**, a potem go odczytujemy. Nie potrzebujemy mocka, bo zapis do malego katalogu tymczasowego jest szybki i mozna go odizolowac od danych ETL. To test wspolpracy naszego zapisu z systemem plikow. Nie dowodzi jeszcze, ze mamy uprawnienia do prawdziwego katalogu docelowego.

`tmp_path` to gotowa fixture z `pytest`. Kazdy test, ktory o nia poprosi, dostaje wlasny katalog tymczasowy. Sprawdzamy nie tylko istnienie pliku, ale tez to, czy po odczytaniu ma te same dane. Nie zalezymy od pliku z poprzedniego przebiegu.

## 11. Sprawdz klienta HTTP bez wysylania zapytania

Chcemy uruchomic prawdziwa funkcje `load_payload()`, ale z gory wybrac, co "odpowie NASA". Na czas testu zastapimy tylko `requests.get` w miejscu, w ktorym uzywa go nasz kod. Taka podmiane nazywamy **mockowaniem**.

### Co zastepujemy i dlaczego?

`load_payload()` zawiera dwie rzeczy: kontakt z siecia i decyzje, co zrobic z odpowiedzia. Chcemy sprawdzic decyzje naszego kodu, nie sprawnosc NASA. Dlatego funkcja pozostaje prawdziwa, a jej zaleznosc HTTP dostaje zastepstwo.

`response = MagicMock()` tworzy obiekt udajacy odpowiedz. `response.json.return_value = {"events": []}` oznacza: wywolanie `response.json()` ma oddac ten slownik. `response.raise_for_status()` domyslnie niczego nie zglasza; w drugim tescie celowo ustawiamy mu wyjatek. Mock zapamietuje tez wywolania, wiec mozemy sprawdzic URL, parametry i timeout.

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

`get.assert_called_once_with(...)` sprawdza, ze GET wywolano dokladnie raz i z podanymi argumentami. `response.json.assert_not_called()` pilnuje, ze po bledzie HTTP nie probowalismy czytac odpowiedzi jak poprawnych danych. Samo `assert result == ...` nie sprawdziloby tych decyzji.

`MagicMock` to zastepczy obiekt, ktory moze oddac ustalona wartosc i zapamietac wywolania. `return_value` mowi "zwroc to", a `side_effect` w tym przykladzie mowi "zglos ten blad". `patch` wstawia zastepstwo na czas bloku `with`, a potem przywraca prawdziwa funkcje. Wybieramy nazwe tak, jak widzi ja testowany modul: w `ingestion.py` to `requests.get`, wiec uzywamy `ingestion.requests.get`. Nie podmieniamy samego `load_payload()`, bo wtedy nie sprawdzilibysmy jego pracy.

Gdy nie potrzebujesz sprawdzac wywolan, czasem wystarczy samodzielnie napisana funkcja albo klasa z kilkoma metodami, czyli **fake**. Dodatek `pytest-mock` daje fixture `mocker`. Po `python -m pip install pytest-mock` mozna w dodatkowym tescie uzyc `get = mocker.patch("ingestion.requests.get")`, ustawic `get.return_value.json.return_value` i wykonac te same sprawdzenia. Fixture sama cofa podmiane po tescie. Nie ma potrzeby pisac trzech kopii kazdego testu.


### Czy brak events i pusta lista to to samo?

Nie. `{"events": []}` to poprawna odpowiedz bez zdarzen. `{}` to dokument bez wymaganej listy. Dodaj do `tests/test_ingestion.py`:

```python
@pytest.mark.parametrize(
    "payload",
    [{}, {"events": None}, {"events": {}}, {"events": "bad"}, []],
    ids=["missing", "null", "dict", "text", "not-object"],
)
def test_load_payload_rejects_invalid_structure(payload):
    response = MagicMock()
    response.json.return_value = payload

    with patch("ingestion.requests.get", return_value=response):
        with pytest.raises(ValueError, match="events list"):
            load_payload()


def test_load_payload_propagates_timeout():
    from requests import Timeout

    with patch("ingestion.requests.get", side_effect=Timeout("Test timeout")):
        with pytest.raises(Timeout):
            load_payload()
```

To kolejne **6 przypadkow**, nadal bez prawdziwego polaczenia z NASA. `side_effect=Timeout(...)` zglasza wyjatek od razu przy wywolaniu podmienionego GET. Ten test sprawdza przekazanie bledu, nie dostepnosc internetu.

Przyklady dotycza klienta z 03, przed dodaniem ponowien z 09. Jesli masz juz retry, podmien takze `ingestion.time.sleep` (albo odpowiednia nazwe modulu pakietu), zeby test nie czekal naprawde. Dla bledu przejsciowego sprawdz liczbe prob i opoznienia, dla np. 404 sprawdz brak ponowienia. Nie uzalezniaj czasu testu od rzeczywistego backoffu.

## 12. Sprawdz, czy kroki pipeline'u wspolpracuja

Najpierw ustalamy granice: wykonamy prawdziwe `run_pipeline`, transformacje i zapis raw. Nie otworzymy HTTP ani polaczenia z baza. To sprawdzenie kilku warstw naraz, ale nie pelne E2E.

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

`monkeypatch` to fixture z `pytest`, ktora po tescie cofa podmiany. `pipeline.py` robi `from loading import load_events`, wiec w tescie podmieniamy `pipeline.load_events`. Slownik z bledem ma byc w raw, ale nie ma trafic do funkcji zapisujacej. Ten test nie dowodzi, ze SQL dziala: nie wykonuje prawdziwego SQL. Po tym kroku podstawowy zestaw to **24 przypadki testowe** (bez opcjonalnego PostgreSQL i dodatkow DataFrame).

## 13. Opcjonalnie: jeden test z prawdziwym PostgreSQL

Test, ktory sprawdza wspolprace Pythona z baza, to **test integracyjny**. Przyda sie np. do sprawdzenia, czy dwa UPSERT-y tego samego ID zostawiaja jeden wiersz. Tego nie da sie sprawdzic przez samo zastapienie `load_events()`.

Utworz **osobna baze `nasa_events_test`** w srodowisku do cwiczen i wykonaj w niej DDL schematu `eonet` oraz tabeli z [04](04_etl_nasa_postgresql.md). Sprawdz `SELECT current_database();` w DBeaverze przed utworzeniem tabeli. Nie uruchamiaj testu na bazie szkoleniowego pipeline'u ani na produkcji.

Test **nie czyta zwyklego `.env` ETL-u**. Dostanie jawne ustawienia `TEST_DB_*` ze zmiennych srodowiskowych. Brak hosta, nazwy bazy lub loginu jest bledem konfiguracji, a inna nazwa niz `nasa_events_test` zostanie odrzucona **przed zapisem**. To ograniczenie dla tego cwiczenia, nie ogolna metoda rozpoznawania bezpiecznej bazy po nazwie. Haslo moze byc puste tylko wtedy, gdy tak skonfigurowano lokalne uwierzytelnianie.

W katalogu projektu dodaj `pytest.ini`:

```ini
[pytest]
markers =
    integration: test wymagajacy osobnej bazy PostgreSQL
```

Dodaj `tests/test_postgres_integration.py`:

```python
import os
from uuid import uuid4

import pytest

from config import DatabaseConfig
from db import open_connection
from loading import load_events
from transformations import Event


def database_config_for_test() -> DatabaseConfig:
    required = ("TEST_DB_HOST", "TEST_DB_NAME", "TEST_DB_USER")
    missing = [name for name in required if not os.environ.get(name, "").strip()]
    if missing:
        raise ValueError(f"Missing test settings: {', '.join(missing)}")
    if os.environ["TEST_DB_NAME"] != "nasa_events_test":
        raise ValueError("Only nasa_events_test is allowed for this test")

    return DatabaseConfig(
        host=os.environ["TEST_DB_HOST"],
        port=int(os.environ.get("TEST_DB_PORT", "5432")),
        dbname=os.environ["TEST_DB_NAME"],
        user=os.environ["TEST_DB_USER"],
        password=os.environ.get("TEST_DB_PASSWORD", ""),
    )


@pytest.mark.integration
@pytest.mark.skipif(os.getenv("RUN_DB_TESTS") != "1", reason="Requires nasa_events_test")
def test_upsert_updates_one_event():
    config = database_config_for_test()
    with open_connection(config) as conn:
        actual_database = conn.execute("SELECT current_database()").fetchone()[0]
    if actual_database != "nasa_events_test":
        raise ValueError("Connected to an unexpected database")

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

`database_config_for_test()` jest zwykla funkcja pomocnicza, nie testem: jej nazwa nie zaczyna sie od `test_`. Najpierw sprawdza wymagane ustawienia, a potem buduje `DatabaseConfig`. Dodatkowe sprawdzenie `current_database()` potwierdza, gdzie naprawde sie polaczylismy. `try/finally` zaczyna sie dopiero po potwierdzeniu bazy; sprzatanie nie dotyka calej tabeli.

`integration` oznacza grupe. `skipif` pomija test, gdy nie wlaczylismy go jawnie. Losowe ID chroni przed pomieszaniem przebiegow, a `finally` usuwa tylko wiersz utworzony przez ten test, takze przy nieudanym `assert`. Baza musi istniec przed uruchomieniem.

```powershell
python -m pytest -m "not integration"
$env:TEST_DB_HOST = "127.0.0.1"
$env:TEST_DB_PORT = "5432"
$env:TEST_DB_NAME = "nasa_events_test"
$env:TEST_DB_USER = "postgres"
$env:TEST_DB_PASSWORD = ""  # ustaw lokalnie, jesli baza wymaga hasla
$env:RUN_DB_TESTS = "1"
python -m pytest -m integration -v
Remove-Item Env:RUN_DB_TESTS
```

Powyzej jest skladnia PowerShella. W terminalu macOS/Linux ustawiasz te same zmienne przez `export TEST_DB_NAME=nasa_events_test` itd., a flage usuwasz przez `unset RUN_DB_TESTS`. Hasla ustawiaj lokalnie, nie wstawiaj ich do repozytorium i nie pokazuj na wspoldzielonym ekranie.

Bez flagi test integracyjny ma status `skipped`, nie `passed`. Samo `-m integration` wybiera grupe, ale nie wlacza flagi. Uzycie `RUN_DB_TESTS=1` z brakujaca konfiguracja ma dac czytelny blad, a nie przejsc na domyslna baze.

`skip` pomija test zawsze, np. przez tymczasowo niedostepne srodowisko. `xfail` oznacza znany problem: test sie wykonuje, ale spodziewamy sie porazki. Przy `strict=True` niespodziewane przejscie testu jest sygnalem, by usunac oznaczenie i zweryfikowac, czy problem zniknal. Nie uzywaj `skip` ani `xfail` do chowania zwyklych bledow.

## 14. Pokrycie kodu i kontrola jakosci

W tym miejscu mamy testy roznych warstw. Coverage pomaga znalezc miejsca, ktore nie zostaly wykonane. Nie odpowie jednak, czy asercje byly sensowne. Najpierw sprawdzamy wymagania z tabeli przypadkow, potem raport.

```powershell
python -m pip install pytest-cov
python -m pytest --cov=transformations --cov=ingestion --cov=storage --cov=pipeline --cov-report=term-missing
```

Pokrycie kodu (ang. `coverage`) mowi, ktore linie zostaly wykonane podczas testow. Kolumna `Missing` pokazuje linie, ktorych testy nie odwiedzily. Nie jest to ocena poprawnosci: mozna wykonac linie bez sprawdzenia ich wyniku. Testuj decyzje w kodzie, nie dzialanie biblioteki standardowej. Np. warto sprawdzic, ze brak `closed` jest bledem, choc `closed=None` jest poprawne.

### Samodzielnie

Dopisz dwa testy przeplywu: dla `{"events": []}` wynik to zero zapisanych i brak odrzucen, plik raw powstaje, a `load_events()` nie jest wywolane; gdy `load_payload()` zglasza wyjatek, ten sam blad opuszcza `run_pipeline()`, a zapis do bazy nie jest uruchamiany. Uzyj `tmp_path` i podmian. Nie lacz sie z NASA ani baza. Uruchom testy i sprawdz ponownie raport pokrycia.

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
