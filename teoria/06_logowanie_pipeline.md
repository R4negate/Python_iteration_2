# 06. Logowanie: co zrobil pipeline i gdzie sie zatrzymal?

[Zadanie: zbuduj pipeline](../zadania/06_pipeline_logowanie.md)

W [lekcji 04](04_etl_nasa_postgresql.md) program na koncu wypisywal liczbe zapisanych zdarzen. Gdy przerwal prace wczesniej, tego komunikatu nie bylo. Chcemy wiedziec, czy wlasnie czekal na NASA, zapisywal JSON, czy laczyl sie z PostgreSQL. **Log** to zapis tego, co dzialo sie podczas pracy programu.

Pracujemy w tym samym projekcie `nasa_pipeline`. Nadal uruchamiamy `python main.py` i pobieramy dane z NASA. Nie zmieniamy serwera FastAPI ani nie dodajemy drugiego zrodla danych. Wbudowany modul `logging` nie wymaga instalacji.

## 1. Od `print()` do pierwszego logu

`print()` jest w sam raz, gdy chcemy pokazac wynik osobie uruchamiajacej program. W logu oprocz tekstu chcemy miec **czas, wage komunikatu i nazwe modulu**. Chcemy tez miec mozliwosc ukrycia szczegolow bez usuwania ich z kodu.

W `main.py` dopisz na gorze `import logging`. Na poczatku `main()`, przed `load_config()`, dodaj:

```python
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s - %(message)s",
    )
    logging.getLogger("nasa_etl").info("ETL: start")
```

Uruchom `python main.py`. Najpierw pojawi sie wpis w rodzaju `2026-01-10 12:00:00,123 INFO nasa_etl - ETL: start`. Stare podsumowanie nadal moze sie pojawic, bo jeszcze go nie zastepowalismy.

| Poziom | Przyklad z naszego pipeline'u |
|---|---|
| `DEBUG` | Gdzie dokladnie zapisano JSON? Przydaje sie podczas szukania problemu. |
| `INFO` | Ile zdarzen pobrano? Czy zapis do bazy sie zakonczyl? |
| `WARNING` | Odrzucono jeden rekord, ale reszte mozna zapisac. |
| `ERROR` | Nie udalo sie wykonac potrzebnego kroku. |

`level=logging.INFO` pokazuje komunikaty od `INFO` wzwyz. `DEBUG` jest wtedy ukryty. `%(asctime)s` to czas, `%(levelname)s` to poziom, `%(name)s` to nazwa loggera, a `%(message)s` to nasz tekst.

## 2. Dodaj logi przed waznymi krokami i po nich

Zastap `pipeline.py` ponizszym kodem. Nie zmieniamy kolejnosci krokow ani wyniku zwracanego przez funkcje.

```python
import json
import logging
from pathlib import Path

from config import DatabaseConfig
from ingestion import load_payload
from loading import load_events
from transformations import transform_events


logger = logging.getLogger(__name__)


def run_pipeline(data_dir: Path, config: DatabaseConfig) -> tuple[int, list[dict]]:
    logger.info("Extract: pobieram zdarzenia z NASA EONET")
    payload = load_payload()
    events = payload["events"]
    logger.info("Extract: pobrano %s zdarzen", len(events))

    logger.info("Raw: zapisuje odpowiedz")
    raw_path = data_dir / "raw" / "eonet_events.json"
    raw_path.parent.mkdir(parents=True, exist_ok=True)
    with raw_path.open("w", encoding="utf-8") as file:
        json.dump(payload, file, ensure_ascii=False, indent=2)
    logger.debug("Raw: zapisano plik %s", raw_path)

    logger.info("Transform: sprawdzam zdarzenia")
    clean, rejected = transform_events(events)
    logger.info("Transform: poprawne=%s, odrzucone=%s", len(clean), len(rejected))
    for row in rejected:
        logger.warning("Odrzucone zdarzenie: pozycja=%s, powod=%s",
                       row["position"], row["reason"])

    if clean:
        logger.info("Load: zapisuje %s zdarzen do PostgreSQL", len(clean))
        load_events(clean, config)
        logger.info("Load: zapis zatwierdzony")
    else:
        logger.warning("Load: brak poprawnych zdarzen do zapisu")

    return len(clean), rejected
```

`logging.getLogger(__name__)` daje logger nazwany tak jak modul, tutaj `pipeline`. Konfiguracja w `main.py` nadal obowiazuje: nie ustawiamy formatu na nowo w kazdym pliku. W `logger.info("... %s", len(events))` podajemy tekst i wartosc osobno; biblioteka wstawi liczbe w miejsce `%s`.

Log przed `load_events()` oznacza "zaczynam zapis". Dopiero log po wywolaniu oznacza "baza zapis potwierdzila". Gdy operacja sie nie uda, nie zobaczymy drugiego wpisu. Odrzucenie wiersza to ostrzezenie. Awaria bazy przerywa cale uruchomienie. Nie wypisujemy hasla, calego `.env`, tokenow ani calej odpowiedzi NASA. Wystarcza nazwa kroku, liczniki i krotkie powody odrzucenia.

## 3. Gdy cos sie zepsuje: log i kod wyjscia

Zastap `main.py`. Teraz podsumowanie jest logiem, a kod zakonczenia programu mowi terminalowi, czy pipeline sie udal.

```python
import logging
from pathlib import Path

from config import load_config
from pipeline import run_pipeline


logger = logging.getLogger("nasa_etl")


def main() -> int:
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s - %(message)s",
    )
    logger.info("ETL: start")

    try:
        logger.info("Config: wczytuje ustawienia")
        config = load_config()
        data_dir = Path(__file__).resolve().parent / "data"
        loaded_count, rejected = run_pipeline(data_dir, config)
    except Exception:
        logger.exception("ETL: przerwano uruchomienie")
        return 1

    logger.info("ETL: zakonczono, zapisane=%s, odrzucone=%s",
                loaded_count, len(rejected))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

`logger.exception()` jest podobne do `logger.error()`, ale w bloku `except` dolacza **traceback**, czyli miejsca w kodzie, ktorymi Python doszedl do bledu. Lapiemy blad na samym koncu sciezki, w punkcie uruchomienia programu, i od razu konczymy z kodem `1`. Nie logujemy bledu po to, zeby isc dalej i udawac sukces. `raise SystemExit(main())` przekazuje terminalowi liczbe zwrocona przez `main()`; `0` oznacza sukces.

1. Uruchom `python main.py` z poprawnym `.env`. W PowerShellu sprawdz `$LASTEXITCODE`. Po udanym przebiegu to `0`.
2. Na probe ustaw w `.env` `DB_PORT=nie_liczba`. Uruchom skrypt jeszcze raz. Zobaczysz log `Config`, wpis `ERROR` i traceback. Nie zobaczysz logu sukcesu. `$LASTEXITCODE` to `1`.
3. Przywroc oryginalny port. Zmien na chwile `level=logging.INFO` na `level=logging.DEBUG` i sprawdz, czy pojawia sie wpis ze sciezka pliku. Przywroc `INFO`. W trybie `DEBUG` dodatkowe wpisy moga tez pochodzic z bibliotek.

## 4. Sam log sukcesu nie wystarczy

Po uruchomieniu sprawdzamy **wynik**, nie tylko terminal:

1. Otworz `data/raw/eonet_events.json`. Komenda `python -m json.tool data/raw/eonet_events.json` pozwala sprawdzic, czy plik da sie odczytac jako JSON. Policz zdarzenia w `events` i porownaj z logiem `Extract`.
2. W DBeaverze, w `nasa_events_lab`, sprawdz liczbe wierszy oraz to, czy zapisane rekordy maja wymagane dane:

```sql
SELECT
    COUNT(*) AS events_in_database,
    COUNT(DISTINCT event_id) AS distinct_ids,
    COUNT(*) FILTER (
        WHERE title IS NULL OR btrim(title) = ''
           OR categories IS NULL OR cardinality(categories) = 0
    ) AS invalid_rows
FROM eonet.events;
```

3. `invalid_rows` powinno wyniesc `0`. `events_in_database` ma byc rowne `distinct_ids`. Jezeli NASA zwrocila zdarzenia, ale po udanym zapisie tabela jest pusta, trzeba to sprawdzic. Brak daty zamkniecia nie jest bledem: dla otwartego zdarzenia `closed_at IS NULL` jest poprawne.
4. Uruchom pipeline drugi raz. Jesli NASA zwrocila te same ID, liczba wierszy nie powinna sie podwoic. Licznik `zapisane` z logu to liczba przetworzonych UPSERT-ow, a nie liczba nowo dodanych wierszy. Jesli chcesz sprawdzic konkretne ID, porownaj je z plikiem raw i wynikiem `SELECT event_id FROM eonet.events ORDER BY event_id;`.

To **kontrole jakosci danych**. Zalezne tabele mialyby tez kontrole relacji, np. czy kazda pozycja ma zamowienie. U nas kazde zdarzenie miesci sie w jednym wierszu, wiec nie dorabiamy takiego testu na sile. Testy automatyczne dopiszemy w [lekcji 07](07_testy_pipeline_pytest.md).

### Samodzielnie

Zmien na probe tytul zdarzenia w szkoleniowej bazie na pusty tekst, uruchom samo zapytanie kontrolne i przewidz wynik. Przywroc poprzedni tytul (najprosciej wykonac probe w transakcji i zakonczyc `ROLLBACK`). Nie modyfikuj danych pobranych z NASA i nie dodawaj nowego trybu pipeline'u.

## 5. Jak przekazac projekt innej osobie?

W `README.md` projektu opisz: wymagana wersje Pythona, sposob instalacji bibliotek, utworzenie tabeli z 04, potrzebne **nazwy** zmiennych z `.env` (bez hasel), polecenie `python main.py` oraz miejsca, w ktorych sprawdzamy wynik. Po lekcji 07 dodaj tam tez komende uruchamiajaca testy. `.env` i `.env.api` zostaja poza repozytorium.

Ruff, zarzadzanie zaleznosciami i Docker maja osobna [lekcje 08](08_uv_poetry_ruff_docker.md).

## Dokumentacja i inspiracja

- [Python: przewodnik po logging](https://docs.python.org/3/howto/logging.html)
- Zakres porownano z modulem Python for Data Engineering, Week 02, Day 4 w [DataCee](https://www.datacee.com/). Kod i dane dotycza naszego projektu NASA EONET.
