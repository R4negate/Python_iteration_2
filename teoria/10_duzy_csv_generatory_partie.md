# 10. Duzy CSV: iteratory, yield i zapis partiami

W [lekcji 01](01_csv_pipeline.md) zapisywalismy wszystkie rekordy do listy, a dopiero pozniej je czyscilismy. Dla siedmiu zamowien to wystarcza. Ale przy wiekszym pliku Python trzymalby naraz surowe rekordy i wyniki. Chcemy, zeby potrafil rozpoczac prace bez czekania na wczytanie calego CSV.

Wroc do projektu `datashop_csv` z 01. Najlepiej pracuj na kopii, zeby porownac oba rozwiazania. Zachowaj `config.py`, `main.py`, `normalize_order()` w `transformations.py` i plik [orders.csv](../materialy/01_csv/orders.csv) w `data/raw/`. Uzyj **Pythona 3.12 lub nowszego**, bo w tej wersji pojawilo sie `itertools.batched`. To osobny projekt; nie zmieniamy pipeline'u NASA.

[Zadanie: przerob pipeline CSV na wersje strumieniowa](../zadania/10_pipeline_csv_strumieniowy.md)

## 1. Odczyt po jednym rekordzie

Lista odpowiada na pytanie "jakie sa wszystkie elementy?". Iterator odpowiada "jaki jest nastepny element?". `next(iterator)` bierze jeden element, a `for` prosi o kolejne, az sie skoncza.

Generator to sposob napisania takiego iteratora. `yield` oddaje jeden wynik i zatrzymuje funkcje w tym miejscu. Przy nastepnym `next()` Python wraca do miejsca zatrzymania. `return` konczy prace funkcji, a nie tylko oddaje jej kolejny element.

Zastap `ingestion.py`. Zachowujemy reguly struktury z 01 i jawnie ustawiamy limit rozmiaru pliku na 50 MB na potrzeby cwiczenia. To osobna decyzja: odczyt strumieniowy nie oznacza przyjmowania dowolnie duzego pliku.

```python
import csv
from collections.abc import Generator
from pathlib import Path


EXPECTED_COLUMNS = ["order_id", "customer_name", "status", "amount", "country", "note"]


def iter_orders(path: Path, max_bytes: int = 50_000_000) -> Generator[tuple[int, dict], None, None]:
    if path.stat().st_size > max_bytes:
        raise ValueError("Input file exceeds the agreed size limit")

    with path.open("r", encoding="utf-8-sig", newline="") as file:
        reader = csv.DictReader(file, delimiter=";", strict=True)
        if reader.fieldnames != EXPECTED_COLUMNS:
            raise ValueError(f"Unexpected CSV header: {reader.fieldnames!r}")

        for position, row in enumerate(reader, start=1):
            if None in row or any(value is None for value in row.values()):
                raise ValueError(f"Wrong number of cells at record {position}")
            yield position, row
```

Uruchom w katalogu projektu, zanim polaczysz etapy:

```powershell
python -c "from pathlib import Path; from ingestion import iter_orders; rows = iter_orders(Path('data/raw/orders.csv')); print(next(rows)); print(next(rows)); rows.close()"
```

Zobaczysz tylko pozycje 1 i 2. Samo `iter_orders(...)` jeszcze nie otwiera pliku: funkcja rusza przy pierwszym `next()`. Plik otwieramy **wewnatrz** generatora i pozostaje otwarty w trakcie odczytu. W tej krotkiej probie nie dochodzimy do konca, wiec `rows.close()` zamyka generator i plik. W pelnym przebiegu generator zostanie zuzyty, a `with` zamknie plik.

## 2. Transformacja tez oddaje jeden wynik naraz

Nie zmieniaj `normalize_order()`. Zamiast starego `transform_orders()`, ktore zbieralo dwie listy, dodaj w `transformations.py` ponizsza funkcje. Import `Iterator` dodaj na gorze pliku.

```python
from collections.abc import Iterator
```

```python
def iter_transformed(
    rows: Iterator[tuple[int, dict]], source_file: str
) -> Iterator[tuple[bool, dict]]:
    accepted_ids = set()

    for position, raw in rows:
        try:
            order = normalize_order(raw)
            if order["order_id"] in accepted_ids:
                raise ValueError("Duplicate order_id")
        except (KeyError, TypeError, ValueError) as error:
            yield False, {
                "source_file": source_file,
                "record_number": position,
                "raw_order_id": raw.get("order_id"),
                "error_type": type(error).__name__,
                "reason": str(error),
            }
        else:
            accepted_ids.add(order["order_id"])
            yield True, order
```

Kazde `yield` oddaje pare: `True` i oczyszczony rekord albo `False` i opis odrzucenia. Nie gubimy pozycji rekordu. Zle zapisanej kwoty nie mylimy z uszkodzona struktura CSV: kwota odrzuca jeden rekord, a uszkodzony plik przerywa przebieg.

`accepted_ids` nalezy do **calego przebiegu**, nie do pojedynczej partii. Dzieki temu duplikat zostanie odrzucony nawet po zapisaniu poprzedniej partii. Ten zbior moze rosnac wraz z liczba unikalnych zamowien. Zapis partiami ogranicza pamiec na rekordy, ale **nie daje tu stalego zuzycia pamieci**. Jesli sam zbior przestanie sie miescic, osobnym rozwiazaniem jest przechowywanie kluczy na dysku albo w bazie.

## 3. Zapis partiami i dwa pliki wynikowe

Partia to mala grupa kolejnych elementow. Dla siedmiu rekordow i rozmiaru 3 dostaniemy partie **3, 3 i 1**. `batched` bierze tylko tyle, ile potrzeba na kolejna grupe. Zapiszemy ja i wrocimy po nastepna.

Zastap `exporting.py`. Utrzymujemy format plikow z 01. Zapis najpierw idzie do katalogu tymczasowego. Po pelnym odczycie i zamknieciu plikow publikujemy wyniki pod docelowymi nazwami.

```python
import csv
from collections.abc import Iterator
from itertools import batched
from pathlib import Path
from tempfile import TemporaryDirectory


CLEAN_COLUMNS = ["order_id", "customer_name", "status", "amount", "country", "note"]
REJECTED_COLUMNS = ["source_file", "record_number", "raw_order_id", "error_type", "reason"]


def export_stream(
    output_dir: Path, rows: Iterator[tuple[bool, dict]], batch_size: int
) -> dict[str, int]:
    if batch_size < 1:
        raise ValueError("Batch size must be positive")

    output_dir.mkdir(parents=True, exist_ok=True)
    clean_count = 0
    rejected_count = 0
    batch_count = 0

    with TemporaryDirectory(dir=output_dir) as temp_dir:
        clean_path = Path(temp_dir) / "clean_orders.csv"
        rejected_path = Path(temp_dir) / "rejected_orders.csv"

        with (
            clean_path.open("w", encoding="utf-8", newline="") as clean_file,
            rejected_path.open("w", encoding="utf-8", newline="") as rejected_file,
        ):
            clean_writer = csv.DictWriter(clean_file, fieldnames=CLEAN_COLUMNS, delimiter=";")
            rejected_writer = csv.DictWriter(
                rejected_file, fieldnames=REJECTED_COLUMNS, delimiter=";"
            )
            clean_writer.writeheader()
            rejected_writer.writeheader()

            for batch in batched(rows, batch_size):
                clean_writer.writerows(row for accepted, row in batch if accepted)
                rejected_writer.writerows(row for accepted, row in batch if not accepted)

                accepted_count = sum(accepted for accepted, _ in batch)
                clean_count += accepted_count
                rejected_count += len(batch) - accepted_count
                batch_count += 1

        clean_path.replace(output_dir / clean_path.name)
        rejected_path.replace(output_dir / rejected_path.name)

    return {
        "input_rows_count": clean_count + rejected_count,
        "clean_count": clean_count,
        "rejected_count": rejected_count,
        "batch_count": batch_count,
    }
```

Kazdy naglowek zapisujemy raz, **przed petla**. W `writerows(row for ... if ...)` zamiast budowac kolejna liste, podajemy generator wybierajacy rekordy z biezacej partii. `sum(accepted for ...)` liczy prawdziwe flagi: `True` w sumie to 1, `False` to 0.

Przy bledzie odczytu przed `replace()` poprzednie wyniki zostana na swoim miejscu. `replace()` wykonujemy dopiero po zamknieciu obu plikow, co jest wazne m.in. na Windowsie. Dwie podmiany to jednak nie jedna transakcja: awaria miedzy nimi moze pozostawic rozne wersje plikow. Na tym etapie uczymy sie chronic wynik przed uszkodzonym wejsciem, nie publikowac zestawu plikow jak transakcje SQL.

## 4. Polacz etapy i porownaj wyniki

Zastap `pipeline.py`:

```python
from contextlib import closing
from pathlib import Path

from exporting import export_stream
from ingestion import iter_orders
from transformations import iter_transformed


def run_pipeline(input_path: Path, output_dir: Path, batch_size: int = 1_000) -> dict:
    with closing(iter_orders(input_path)) as raw_rows:
        transformed = iter_transformed(raw_rows, input_path.name)
        counts = export_stream(output_dir, transformed, batch_size)
    return {"source_file": input_path.name, **counts}
```

`closing` zamyka generator odczytu rowniez wtedy, gdy np. zapis przestanie dzialac w srodku przebiegu. Nie budujemy listy wszystkich surowych ani wszystkich oczyszczonych zamowien. Zwracamy tylko liczniki.

Zastap `reporting.py`, bo stare podsumowanie probowalo wypisac wszystkie ID:

```python
def print_summary(result: dict) -> None:
    print(f"Plik: {result['source_file']}")
    print(f"Odczytane: {result['input_rows_count']}")
    print(f"Poprawne: {result['clean_count']}, odrzucone: {result['rejected_count']}")
    print(f"Partie: {result['batch_count']}")
```

`main.py` moze zostac bez zmian. Na probe wywolaj ten sam pipeline z rozmiarem partii 3:

```powershell
python -c "from pathlib import Path; from pipeline import run_pipeline; from reporting import print_summary; print_summary(run_pipeline(Path('data/raw/orders.csv'), Path('data/out'), batch_size=3))"
```

Oczekuj **7 odczytanych, 5 poprawnych, 2 odrzuconych, 3 partii**. Wynikowe pliki maja zawierac te same rekordy co w 01, w tej samej kolejnosci w kazdym pliku. Rozmiar partii nie jest regula biznesowa.

## 5. Sprawdz wiekszy plik bez trzymania go w repozytorium

Wygeneruj wieksza dostawe z malego pliku. Utworz `generate_large_csv.py` obok `main.py`:

```python
import csv
from pathlib import Path


def main() -> None:
    raw_dir = Path(__file__).resolve().parent / "data" / "raw"
    with (raw_dir / "orders.csv").open(encoding="utf-8-sig", newline="") as file:
        reader = csv.DictReader(file, delimiter=";")
        rows = list(reader)
        columns = reader.fieldnames

    with (raw_dir / "orders_large.csv").open("w", encoding="utf-8", newline="") as file:
        writer = csv.DictWriter(file, fieldnames=columns, delimiter=";")
        writer.writeheader()
        for group in range(20_000):
            for row in rows:
                order_id = int(row["order_id"]) + group * len(rows)
                writer.writerow({**row, "order_id": order_id})


if __name__ == "__main__":
    main()
```

Tu `list(reader)` jest celowa: czytamy **tylko maly wzorzec**, zeby moc go uzyc wielokrotnie. Duzy plik powstaje po jednym wierszu. Nie dodawaj `orders_large.csv` ani wynikow do Git.

1. Wykonaj `python generate_large_csv.py`.
2. W `.env` ustaw `INPUT_PATH=data/raw/orders_large.csv` i np. `OUTPUT_DIR=data/out_large`, zeby nie nadpisywac wyniku malej proby.
3. Uruchom `python main.py`. Przy `batch_size=1_000` oczekuj **140 000 odczytanych, 100 000 poprawnych, 40 000 odrzuconych i 140 partii**.
4. Przywroc ustawienia malego pliku. Porownaj, co zmienia rozmiar partii, a co musi pozostac bez zmian.

Odczyt po rekordzie dziala dla tego CSV, bo `csv.DictReader` zwraca wiersze po kolei. Nie przenosimy tego automatycznie na plik z jedna wielka tablica JSON: `json.load()` dalej wczyta caly dokument.

## Dokumentacja

- [Python: iteratory i generatory](https://docs.python.org/3/tutorial/classes.html#iterators)
- [itertools.batched](https://docs.python.org/3/library/itertools.html#itertools.batched)
- [Python: csv.DictReader i DictWriter](https://docs.python.org/3/library/csv.html)
