# 10. Duzy CSV od podstaw: iteratory, generatory i partie danych

[Zadanie: przebuduj pipeline CSV](../zadania/10_pipeline_csv_strumieniowy.md)

Wracamy do projektu `datashop_csv` z [lekcji 01](01_csv_pipeline.md), najlepiej na jego kopii. Nie zmieniamy NASA ani PostgreSQL. Korzystamy z tego samego [orders.csv](../materialy/01_csv/orders.csv), tych samych regul zamowien i dwoch plikow wynikowych. Zmieni sie **sposob przeplywu danych**, nie wynik biznesowy.

W srodowisku projektu potrzebny jest **Python 3.12 lub nowszy**, poniewaz korzystamy z `itertools.batched`. Samo zainstalowanie nowego Pythona na komputerze nie przelacza starej .venv: sprawdz interpreter PyCharma i wynik `python --version` w terminalu projektu. Jesli potrzeba, utworz nowe srodowisko z wybranym interpreterem i zainstaluj python-dotenv z 01. Reszta kodu pipeline'u korzysta z biblioteki standardowej.


## 1. Po co zmieniac pipeline, ktory juz dziala?

W lekcji 01 odczyt konczyl sie zbudowaniem listy wszystkich rekordow. Transformacja budowala kolejne listy: poprawnych i odrzuconych. Dopiero potem zaczynal sie zapis.

```text
wczytaj wszystkie -> przygotuj wszystkie -> zapisz wszystkie
```

Przy siedmiu rekordach to czytelne i wystarczajace. Przy duzym pliku listy moga zajac duzo pamieci. Surowe stringi, slowniki, listy i obiekty w Pythonie maja swoj koszt, wiec plik o rozmiarze 100 MB nie musi zajmowac w RAM dokladnie 100 MB.

Chcemy rozpoczac przetwarzanie, zanim caly plik zostanie wczytany. Zamiast zbierac wszystkie rekordy, bedziemy przekazywac kolejne i zapisywac ich niewielkie grupy:

```text
odczytaj rekord -> przeksztalc rekord -> dolacz do biezacej partii
                                             |
                                  partia gotowa? zapisz ja
                                             |
                                  pobierz kolejne rekordy
```

To nazywamy tu **odczytem strumieniowym**. Nie oznacza to aplikacji dzialajacej bez konca ani osobnego systemu do zdarzen na zywo. Nasz plik nadal ma poczatek i koniec; po jego przetworzeniu ETL sie zakonczy.

### Czy lista jest zla?

Nie. Lista daje prosty dostep po indeksie, latwo policzyc jej dlugosc i mozna przejsc po niej kilka razy. Dla malej, ograniczonej dostawy poprzednia wersja nadal jest poprawnym wyborem.

Generator wybieramy, gdy chcemy przetwarzac po kolei i nie potrzebujemy pelnej listy. Cena jest koniecznosc zrozumienia jednorazowego odczytu, zamykania pliku i reakcji na blad, ktory moze pojawic sie dopiero pod koniec wejscia.

## 2. Iterable i iterator: co robi for?

**Obiekt iterowalny** (ang. iterable) to taki, po ktorym mozemy przechodzic petla for, np. lista, string lub slownik. Funkcja `iter(...)` pobiera z niego **iterator**, czyli obiekt pamietajacy stan kolejnego odczytu.

```python
order_ids = [1001, 1002, 1003]
iterator = iter(order_ids)

print(next(iterator))  # 1001
print(next(iterator))  # 1002
print(next(iterator))  # 1003
print(next(iterator, "Koniec"))  # Koniec
```

`next(iterator)` prosi o nastepny element. Iterator pamieta, ktore juz oddal. Po wyczerpaniu zwykle zglosilby `StopIteration`; w ostatniej linii podalismy wartosc domyslna, wiec oddaje napis Koniec.

Petla for sama obsluguje ten koniec. Dlatego nie piszemy w niej wlasnego while, indeksu ani except StopIteration. W uproszczeniu prosi iterator o kolejne elementy i konczy, gdy ich zabraknie.

### Dlaczego drugi for moze nic nie wypisac?

```python
iterator = iter([1001, 1002])

for order_id in iterator:
    print(order_id)

for order_id in iterator:
    print(order_id)  # Ten blok nie wykona sie ani razu
```

Iterator zostal zuzyty przez pierwsza petle. Lista nadal istnieje i da sie z niej utworzyc nowy iterator, ale ten konkretny iterator nie przewija sie sam. To wazne przy pliku: ponowne przejscie wymaga nowego odczytu, a nie drugiego for po tym samym zuzytym obiekcie.

## 3. Generator i yield: oddaj jeden wynik, potem wroc

**Funkcja generatorowa** zawiera instrukcje yield. Jej wywolanie tworzy generator, ktory jest rodzajem iteratora. Nie wykonuje od razu calego ciala funkcji.

```python
def order_numbers():
    print("Przygotowuje pierwsze ID")
    yield 1001
    print("Przygotowuje drugie ID")
    yield 1002
    print("Koniec pracy")


numbers = order_numbers()
print("Generator utworzony")

print(next(numbers))
print(next(numbers))
print(next(numbers, "Brak kolejnego ID"))
```

Wynik:

```text
Generator utworzony
Przygotowuje pierwsze ID
1001
Przygotowuje drugie ID
1002
Koniec pracy
Brak kolejnego ID
```

Pierwsze next uruchamia funkcje do pierwszego yield. Funkcja oddaje 1001 i **wstrzymuje wykonanie**. Drugie next wznawia ja po tym yield i dochodzi do kolejnego. Nie zaczyna calej funkcji od poczatku. Trzecie next pozwala dojsc do konca; domyslny tekst zastepuje StopIteration.

### Yield a return

- `yield value` przekazuje kolejny element i zachowuje stan funkcji, zeby mozna bylo kontynuowac.
- `return` konczy funkcje. W generatorze oznacza koniec dostepnych elementow; nie sluzy do oddawania nastepnego rekordu w petli.
- Generator nie jest watkiem ani procesem w tle. Pracuje wtedy, gdy odbiorca prosi o nastepny element.

Generator expressions, np. `(row for row in batch if row["status"] == "paid")`, tez dostarczaja elementy stopniowo. Nawiasy okragle nie tworza listy jak list comprehension. W tej lekcji uzyjemy takiego zapisu przy eksporcie.


## 4. Leniosc musi dotyczyc calego przeplywu

**Leniwe przetwarzanie** oznacza obliczanie kolejnych wynikow dopiero wtedy, gdy sa potrzebne. Samo uzycie yield nie wystarczy, jesli za chwile ponownie zbierzemy wszystko do listy.

```python
rows = iter_orders(input_path)
all_rows = list(rows)  # Znowu wczytujemy caly wynik do pamieci
```

To antyprzyklad dla duzego wejscia. Tak samo `sorted(rows)` musi zebrac wszystkie elementy, zanim odda posortowana liste. Liczenie `len(list(rows))` zuzyje generator: pozniejszy odczyt tego samego obiektu nie da tych samych rekordow.

Nasz docelowy przeplyw bedzie mial trzy warstwy:

```text
iter_orders -> iter_transformed -> export_stream
kolejny raw    kolejny wynik       grupa wynikow i zapis
```

Export prosi o elementy. To powoduje prace transformacji, ktora prosi odczyt o kolejne rekordy. Utworzenie tych generatorow samo jeszcze nie przetwarza calego pliku. Plik CSV jest czytany z buforowaniem systemowym, wiec nie obiecujemy jednego fizycznego odczytu dysku na jeden rekord; unikamy pelnej listy slownikow w programie.

## 5. Najpierw generator odczytu CSV

Zachowaj z lekcji 01 config.py, main.py, normalize_order w transformations.py i wejscie `data/raw/orders.csv`. Zmieniamy ingestion.py.

### Kontrakt pozostaje jawny

Nadal wymagamy UTF-8 z opcjonalnym BOM, srednika, dokladnych nazw i kolejnosci kolumn oraz szesciu komorek w kazdym rekordzie. Country i note moga byc puste, ale ich komorki musza istniec.

Dla tego cwiczenia podnosimy limit dostawy z 5 MB do **50 000 000 bajtow**, zeby zmiescil sie pozniej wygenerowany przyklad. To swiadoma zmiana limitu, nie skutek yield. Zakladamy gotowy plik, ktorego dostawca nie dopisuje w trakcie odczytu. Strumieniowy odczyt nie oznacza przyjmowania plikow dowolnej wielkosci.

### Kod ingestion.py

```python
import csv
from collections.abc import Generator
from pathlib import Path


EXPECTED_COLUMNS = ["order_id", "customer_name", "status", "amount", "country", "note"]


def iter_orders(path: Path, max_bytes: int = 50_000_000) -> Generator[tuple[int, dict], None, None]:
    if type(max_bytes) is not int or max_bytes < 1:
        raise ValueError("max_bytes must be a positive integer")
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

### Wyjasnienie krok po kroku

1. Adnotacja `Generator[tuple[int, dict], None, None]` opisuje, ze generator oddaje pare: pozycje i slownik. Dwa None mowia, ze nie projektujemy wysylania wartosci do generatora ani koncowego wyniku przez return. Sama adnotacja nie odczytuje ani nie waliduje danych.
2. Sprawdzamy, czy limit jest dodatnia liczba calkowita. `type(...) is int` odrzuca tez bool, ktory Python traktuje jako szczegolny rodzaj liczby.
3. `path.stat().st_size` pobiera rozmiar pliku w bajtach bez wczytywania go do listy. Nieistniejaca sciezka zglosi FileNotFoundError.
4. Plik otwieramy **wewnatrz generatora**. `utf-8-sig` obsluguje ewentualny BOM, a `newline=""` zostawia obsluge koncow linii parserowi CSV.
5. `DictReader` odczytuje komorki, a fieldnames daje naglowek. Porownanie list pilnuje tez kolejnosci. `strict=True` zaostrza reakcje parsera na bledny zapis cytowania.
6. `enumerate(reader, start=1)` nadaje numery rekordom danych, bez naglowka. Numer nie musi byc numerem fizycznej linii: notatka w cudzyslowach moze miec nowa linie.
7. `None in row` wykrywa nadmiarowe komorki zapisane przez DictReader pod dodatkowym kluczem None. `any(value is None for value in row.values())` wykrywa brakujace komorki. Pusty tekst nie jest None.
8. `yield position, row` oddaje jedna krotke i wstrzymuje odczyt. Nie dopisujemy row do listy.

### Sprawdzenie dwoch rekordow i zamkniecia pliku

W tymczasowym `check_reading.py` obok main.py uruchom:

```python
from contextlib import closing
from pathlib import Path

from ingestion import iter_orders


input_path = Path(__file__).resolve().parent / "data" / "raw" / "orders.csv"
with closing(iter_orders(input_path)) as rows:
    print(next(rows))
    print(next(rows))
```

```powershell
python check_reading.py
```

Zobaczysz pozycje 1 i 2 oraz surowe slowniki. Kwota pozostaje tekstem, bo transformacja jeszcze nie nastapila. Ten plik sprawdza tylko odczyt: nie uruchamia eksportu i nie zastepuje docelowego main.py.

### Kiedy plik jest otwarty i co go zamyka?

Samo `rows = iter_orders(path)` tworzy generator. Kontrola rozmiaru, otwarcie pliku i odczyt naglowka nastapia dopiero przy pierwszym next. Dlatego nawet blad zlej sciezki pojawi sie podczas odczytu, nie przy samym tworzeniu obiektu.

With wewnatrz iter_orders pozostaje aktywny pomiedzy yield. Po calkowitym zuzyciu generatora blok sie konczy i zamyka plik. Jesli jednak zrobimy tylko dwa next albo odbiorca przerwie prace, trzeba zamknac generator. `closing(...)` zapewnia wywolanie `rows.close()` na koncu bloku, rowniez przy wyjatku. Samo break w dowolnej petli nie daje takiej gwarancji.

Nie zwracamy DictReader spoza zakonczonego with: wtedy iterator probowalby czytac juz zamkniety plik. Nie polegamy tez na tym, ze Python „kiedys sam posprzata”.


## 6. Transformacja jednego rekordu naraz

Nie zmieniamy normalize_order z 01. Nadal oczyszcza teksty, waliduje status, tworzy Decimal i odrzuca niepoprawne kwoty. Kwota zero jest poprawna, a statusy pending i cancelled nadal sa dozwolone: to nie raport tylko paid.

Do transformations.py dodaj import i funkcje:

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

### Co znaczy wynik generatora?

`yield True, order` oddaje krotke z flaga przyjecia i oczyszczonym slownikiem. `yield False, {...}` oddaje flage odrzucenia i slownik z powodem. Nie zwracamy dwoch pelnych list, jak poprzednio.

Adnotacja Iterator opisuje sposob pobierania wynikow, a nie ich gotowa liste. Mozna rozpakowac kolejny wynik przez `accepted, row = next(transformed)`. Znaczenie slownika zalezy od flagi: to albo zamowienie, albo opis odrzucenia.

### Dlaczego except nie obejmuje calego odczytu?

Blad kwoty nalezy do jednego poprawnie odczytanego rekordu. Zapisujemy powod i przechodzimy dalej. Bledna liczba komorek, uszkodzone cytowanie albo blad dysku oznaczaja problem z dostawa. Zgloszone podczas pobierania kolejnego elementu petli **nie sa** tu zamieniane na pojedyncze odrzucenie.

Nie lapalismy Exception i nie zwracamy pustego wyniku po kazdej awarii. Granica miedzy zlym rekordem i zlym plikiem jest taka sama jak w 01.

### Dlaczego accepted_ids jest poza petla i poza partia?

Ten set pamieta ID **poprawnie przyjetych** zamowien w calym przebiegu. Po zapisaniu pierwszej partii nie zapominamy jej identyfikatorow. Duplikat z kolejnej partii tez musi byc odrzucony.

ID dodajemy do setu dopiero po poprawnej walidacji. Jesli pierwsze wystapienie mialo bledna kwote, nie blokuje pozniejszego poprawnego zamowienia z tym samym ID.

To nowy set dla kazdego uruchomienia generatora, nie globalny stan pozostajacy po poprzednim imporcie.


## 7. Co to jest partia i jak dziala batched?

Partia (ang. batch) to grupa elementow przetwarzanych razem. Tutaj grupujemy **wyniki transformacji dla kolejnych rekordow wejsciowych**, zarowno poprawnych, jak i odrzuconych.

```python
from itertools import batched


for batch in batched(range(1, 8), 3):
    print(batch)
```

Wynik:

```text
(1, 2, 3)
(4, 5, 6)
(7,)
```

Kazda partia jest krotka o rozmiarze najwyzej 3. Ostatnia moze byc niepelna; nie odrzucamy jej i nie dopelniamy sztucznymi rekordami. Przecinek w `(7,)` oznacza jednoelementowa krotke, nie zwykla liczbe w nawiasach.

Przy pustym wejsciu nie ma partii. Przy siedmiu rekordach partia 3 daje trzy partie, a partia 1000 daje jedna. Rozmiar partii musi byc dodatnia liczba calkowita, nie True ani 2.5.

### Jak partia ma sie do poprawnych zamowien?

W pierwszej partii sa pozycje 1, 2, 3. W drugiej 4, 5, 6: pozycja 4 ma bledna kwote, ale **nadal zajmuje miejsce w partii**. Trzecia partia to pozycja 7, odrzucona za NaN.

Partia rozmiaru 3 nie oznacza „czekaj, az zbierzesz trzy poprawne sprzedaze”. Inaczej liczba i granice partii zalezalaby od jakosci danych. Tu chcemy rozliczyc wszystkie rekordy wejscia.

Podobnie jak przy USGS z 04b pracujemy na ograniczonej grupie naraz. Ale tutaj nie ma stron HTTP ani COMMIT po kazdej partii. Przetwarzamy lokalny plik, a zapis CSV ma inne zasady niz transakcja bazy.

## 8. Export: partie zapisujemy do dwoch plikow

Zastap exporting.py. Kazda kolejna partia zostanie podzielona na poprawne rekordy i odrzucenia. Nazwy i kolumny plikow zostaja takie jak w 01.

### Dlaczego najpierw katalog tymczasowy?

Przy odczycie strumieniowym mozemy odkryc uszkodzony rekord strukturalny dopiero pod koniec pliku. Nie chcemy wtedy zastapic poprzednich raportow wynikiem zawierajacym tylko poczatek nowej dostawy.

Dlatego zapisujemy do plikow roboczych. Dopiero po pelnym odczycie i poprawnym zamknieciu obu plikow podmienimy nazwy docelowe. Pliki robocze znajduja sie w katalogu wynikowym, a nie w innym przypadkowym miejscu na dysku.

### Kod exporting.py

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
    if type(batch_size) is not int:
        raise TypeError("Batch size must be an integer, not bool")
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

### Czytamy przygotowanie plikow

- Kontrola batch_size dzieje sie przed utworzeniem katalogu i pobieraniem danych. Nie przyjmujemy bool jako liczby mimo pokrewienstwa z int w Pythonie.
- `output_dir.mkdir(...)` przygotowuje miejsce na wyniki. To katalog, a nie nazwa pliku.
- Liczniki zaczynaja sie od zera: nie przechowujemy list ID do podsumowania.
- `TemporaryDirectory(dir=output_dir)` tworzy tymczasowy podkatalog, ktory po wyjsciu z with zostanie posprzatany.
- Zapis `with (..., ...):` otwiera dwa pliki w jednym bloku. Wyjscie zamyka oba, takze przy bledzie.
- DictWriter dostaje nazwy kolumn i separator. Naglowek zapisujemy **raz przed petla**, nie w kazdej partii.

### Co robi srodek petli?

`batched(rows, batch_size)` pobiera tylko elementy potrzebne na biezaca partie. Writer poprawnych danych dostaje wyrazenie generatorowe `row for accepted, row in batch if accepted`. Rozpakowujemy kazda pare i wybieramy slowniki z flaga True. Drugi writer bierze flagi False.

Przechodzimy po batch kilka razy i to jest poprawne: batch jest mala **krotka**, nie zuzywanym generatorem. Nie probujemy kilka razy przejsc po calym rows.

`sum(accepted for accepted, _ in batch)` liczy przyjete rekordy, bo True zachowuje sie w sumie jak 1, a False jak 0. Podkreslenie jest zwykla nazwa zmiennej oznaczajaca tutaj: nie potrzebujemy drugiego elementu pary do tego liczenia.

`len(batch) - accepted_count` daje odrzucenia z tej partii. Zwiekszamy batch_count raz na partie, rowniez gdy wszystkie jej rekordy sa odrzucone.

### Zapis partii to nie transakcja SQL

Writerows zapisuje kolejne rekordy do pliku roboczego. Python i system moga buforowac zapis. Nie obiecujemy, ze cala partia jest jednym fizycznym zapisem na dysk albo ze kazda jest osobno zatwierdzona jak w bazie.

`Path.replace` wykonuje sie poza blokiem otwartych plikow, po ich zamknieciu. To wazne m.in. na Windowsie. Przy bledzie odczytu przed tym miejscem poprzednie dwa raporty pozostaja nienaruszone, a katalog roboczy jest sprzatany.

**Dwie podmiany nie sa jedna transakcja.** Awaria podczas publikacji, np. po podmianie pierwszego pliku, moze zostawic pliki z roznych przebiegow. Nie ukrywamy tego ograniczenia. W bardziej rozbudowanym systemie publikowalibysmy wersjonowany katalog i wskaznik kompletnego zestawu; tutaj chronimy wynik przed niepelnym odczytem, nie projektujemy transakcji wieloplikowej.


## 9. Laczymy odczyt, transformacje i eksport

Zastap pipeline.py:

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

### Co wykonuje sie od razu, a co dopiero w eksporcie?

Closing przygotowuje zamkniecie generatora raw_rows. Samo iter_orders nie czyta jeszcze CSV. Utworzenie transformed tez nie przetwarza wszystkich zamowien. Dopiero export_stream zaczyna pobierac elementy i uruchamia te poprzednie warstwy.

Gdy odczyt dojdzie do konca, generator opusci wewnetrzny with i zamknie plik. Gdy zapis przerwie sie wczesniej, zewnetrzny closing tez zamknie raw_rows. Dzieki temu awaria eksportu nie pozostawia otwartego wejscia tylko dlatego, ze generator nie doszedl do konca.

Pipeline oddaje nazwe pliku i liczniki. Nie zwraca wszystkich poprawnych i odrzuconych rekordow, bo to ponownie budowaloby wynik w pamieci. Reguly zamowien zostaly w transformations, a main nie dostalo petli ani walidacji.

### Zmieniamy reporting.py, nie dokladamy logiki do main

Poprzednia wersja podsumowania probowala wypisac wszystkie ID. Teraz podajemy liczniki:

```python
def print_summary(result: dict) -> None:
    print(f"Plik: {result['source_file']}")
    print(f"Odczytane: {result['input_rows_count']}")
    print(f"Poprawne: {result['clean_count']}, odrzucone: {result['rejected_count']}")
    print(f"Partie: {result['batch_count']}")
```

Config.py i main.py z 01 moga pozostac bez zmian. Domyslna partia w nowym run_pipeline to 1000. Na malym pliku sprawdzimy jednak 3, zeby zobaczyc niepelna ostatnia partie:

```powershell
python -c "from pathlib import Path; from pipeline import run_pipeline; from reporting import print_summary; print_summary(run_pipeline(Path('data/raw/orders.csv'), Path('data/out'), batch_size=3))"
```

Polecenie wykonaj z katalogu projektu. W wersji z kodem w podfolderze-pakiecie dopisz jego nazwe do importow, np. `from datashop_csv.pipeline import run_pipeline`, zgodnie z faktyczna nazwa pakietu.

Wynik:

```text
Plik: orders.csv
Odczytane: 7
Poprawne: 5, odrzucone: 2
Partie: 3
```

Poprawne ID to 1001, 1002, 1003, 1005, 1006. Odrzucenia sa na pozycjach 4 i 7. Kazdy plik zachowuje kolejnosc rekordow swojego typu. Na koncu CSV poprawnych danych nadal mamy kwote zero dla 1003.


## 10. Testujemy zachowanie, nie tylko liczniki

Zainstaluj pytest **w srodowisku datashop_csv**, jesli go tam nie ma. Instalacja w projekcie NASA nie dodala go automatycznie do tego projektu:

```powershell
python -m pip install pytest
```

Utworz tests/test_streaming.py. Testy kopiuja maly plik z data/raw do tmp_path, wiec nie modyfikuja oryginalnej dostawy. Kod z 07 przydaje sie tutaj wprost: fixture przygotowuje wejscie, a asercje sprawdzaja wynik.

```python
import csv
from pathlib import Path
from shutil import copyfile

import pytest

from pipeline import run_pipeline


@pytest.fixture
def input_path(tmp_path):
    source = Path(__file__).resolve().parents[1] / "data" / "raw" / "orders.csv"
    target = tmp_path / "orders.csv"
    copyfile(source, target)
    return target


def read_rows(path):
    with path.open(encoding="utf-8", newline="") as file:
        return list(csv.DictReader(file, delimiter=";"))


def test_small_file_counts_and_zero_amount(input_path, tmp_path):
    result = run_pipeline(input_path, tmp_path / "out", batch_size=3)
    rows = read_rows(tmp_path / "out" / "clean_orders.csv")

    assert result == {
        "source_file": "orders.csv",
        "input_rows_count": 7,
        "clean_count": 5,
        "rejected_count": 2,
        "batch_count": 3,
    }
    assert [row["order_id"] for row in rows] == [
        "1001", "1002", "1003", "1005", "1006"
    ]
    assert next(row for row in rows if row["order_id"] == "1003")["amount"] == "0"


def test_batch_size_does_not_change_output(input_path, tmp_path):
    first = tmp_path / "batch_1"
    second = tmp_path / "batch_3"

    run_pipeline(input_path, first, batch_size=1)
    run_pipeline(input_path, second, batch_size=3)

    for name in ("clean_orders.csv", "rejected_orders.csv"):
        assert (first / name).read_bytes() == (second / name).read_bytes()


def test_duplicate_in_later_batch_is_rejected(input_path, tmp_path):
    with input_path.open("a", encoding="utf-8", newline="") as file:
        file.write("1001;Other customer;paid;10;PL;duplicate\n")

    result = run_pipeline(input_path, tmp_path / "out", batch_size=3)
    rejected = read_rows(tmp_path / "out" / "rejected_orders.csv")

    assert result["input_rows_count"] == 8
    assert result["clean_count"] == 5
    assert result["rejected_count"] == 3
    assert rejected[-1]["record_number"] == "8"
    assert rejected[-1]["reason"] == "Duplicate order_id"


def test_late_structure_error_preserves_previous_outputs(input_path, tmp_path):
    output = tmp_path / "out"
    run_pipeline(input_path, output, batch_size=3)
    before = {
        name: (output / name).read_bytes()
        for name in ("clean_orders.csv", "rejected_orders.csv")
    }
    with input_path.open("a", encoding="utf-8", newline="") as file:
        file.write("1008;Missing cells\n")

    with pytest.raises(ValueError, match="Wrong number of cells"):
        run_pipeline(input_path, output, batch_size=3)

    for name, previous_content in before.items():
        assert (output / name).read_bytes() == previous_content
    assert sorted(path.name for path in output.iterdir()) == [
        "clean_orders.csv", "rejected_orders.csv"
    ]
```

```powershell
python -m pytest tests/test_streaming.py -v
```

Oczekujemy **4 passed**. Pierwszy test pilnuje liczb, ID i zera. Drugi sprawdza, ze rozmiar partii nie zmienil zawartosci dwoch plikow. Trzeci odrzuca duplikat, mimo ze pierwsze wystapienie bylo w dawno zapisanej partii. Czwarty powoduje blad struktury na koncu i sprawdza, ze poprzednie raporty oraz porzadek w katalogu zostaly zachowane.

Uzycie listy w pomocniczym read_rows jest celowe: test czyta maly wynik, zeby go porownac. Nie jest to powrot do wczytywania wielkiej dostawy w produkcyjnym pipeline.

Test z bledna liczba komorek nie sprawdza jednoczesnej podmiany dwoch plikow podczas awarii publikacji. Potwierdza konkretnie ochrone przed bledem odczytu **przed** publikacja.


## 11. Wiekszy plik i co z pamiecia?

Najpierw potwierdzilismy poprawny wynik na malej dostawie. Teraz mozemy sprawdzic wieksza bez trzymania jej w repozytorium. Utworz generate_large_csv.py obok main.py:

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

### Dlaczego tu list(reader) jest dopuszczalne?

Czytamy tylko maly, znany plik wzorcowy z siedmioma rekordami, zeby moc przejsc po nim 20 000 razy. Nie tworzymy listy 140 000 wynikow. Duzy plik zapisujemy po jednym wierszu.

W naszym wzorcu ID to kolejne liczby 1001-1007. Przesuniecie `group * len(rows)` o wielokrotnosc siedmiu sprawia, ze nastepne kopie maja nowe ID. To generator danych do tego konkretnego wzorca, nie ogolna metoda nadawania kluczy dla dowolnego pliku.

`{**row, "order_id": order_id}` tworzy nowy slownik z nadpisanym ID; nie zmienia row we wzorcu. Nie poprawiamy bad i NaN, wiec proporcje poprawnych i odrzuconych rekordow zostaja te same.

### Uruchomienie wiekszej proby

1. Wykonaj `python generate_large_csv.py`. Nie edytuj recznie orders.csv.
2. W .env ustaw `INPUT_PATH=data/raw/orders_large.csv` i `OUTPUT_DIR=data/out_large`, zeby nie nadpisac wyniku malej proby.
3. Uruchom `python main.py`. Przy domyslnej partii 1000 oczekuj **140 000 odczytanych, 100 000 poprawnych, 40 000 odrzuconych i 140 partii**.
4. Sprawdz naglowki i dane wynikow. Potem przywroc ustawienia malego pliku.
5. W .gitignore wyklucz orders_large.csv i wygenerowane raporty; nie dodawaj duzej dostawy do historii Git.

### Czy generator gwarantuje stala pamiec?

Nie w tym projekcie. Nie trzymamy pelnych rekordow calego pliku, ale accepted_ids przechowuje kazde przyjete unikalne ID. Dla rosnacego wejscia ten set nadal rosnie. Dodatkowo trzymamy biezaca partie oraz bufory odczytu i zapisu.

Zmniejszenie partii ogranicza pamiec na rekordy w tej partii, ale nie zmniejsza zbioru kluczy potrzebnego do kontroli duplikatow. Jesli ten stan jest zbyt duzy, trzeba rozpatrzyc magazyn kluczy na dysku lub w bazie. Nie usuwamy kontroli duplikatow tylko po to, zeby uzyskac ladny wynik pamieci.

Nie twierdzimy tez, ze partia 1000 zawsze jest najszybsza. Rozmiar dobieramy do objetosci rekordow, limitow pamieci i sposobu zapisu. Tu jest ustawieniem technicznym: nie moze zmienic poprawnych zamowien ani powodow odrzucenia.


## 12. Co musi pozostac prawda po tej zmianie?

| Przypadek | Oczekiwane zachowanie |
| --- | --- |
| Tylko poprawny naglowek, bez rekordow | Dwa pliki z naglowkami, liczniki zero, zero partii. |
| Pusty plik bez naglowka | Blad struktury, nie „zero zamowien”. |
| Pusty kraj przy istniejacej komorce | Normalize_order ustawia UNKNOWN, rekord nadal moze byc poprawny. |
| Kwota zero | Nie odrzucamy jej z powodu falszywej wartosci logicznej. |
| Duplikat w nastepnej partii | Drugie poprawne wystapienie jest odrzucone. |
| Blad zapisu lub odczytu | Wyjatek wychodzi dalej; plik wejsciowy jest zamkniety. |
| Za duzy plik lub niepoprawny naglowek | Przerywamy, nie publikujemy pozornego sukcesu. |
| Zmiana partii z 1 na 3 | Zmienia sie licznik partii, nie rekordy wynikowe. |

### Samodzielnie

Dopisz testy pustej dostawy z samym naglowkiem, pustego pliku bez naglowka, niepoprawnego batch_size, przekroczenia max_bytes i przerwania eksportu po pierwszej partii. Sprawdz, ze plik zostaje zamkniety, a dotychczasowe raporty nie sa podmieniane przed zakonczonym odczytem.

Na koniec porownaj rozwiazanie z 01: **te same reguly i wynik, inny sposob dostarczania rekordow**. Nie przenosimy automatycznie tego rozwiazania na jedna wielka tablice JSON: zwykle json.load nadal wczytuje caly dokument. Potrzebny bylby format lub parser pozwalajacy na stopniowy odczyt.

## Dokumentacja

- [Python: iteratory i generatory](https://docs.python.org/3/tutorial/classes.html#iterators)
- [itertools.batched](https://docs.python.org/3/library/itertools.html#itertools.batched)
- [contextlib.closing](https://docs.python.org/3/library/contextlib.html#contextlib.closing)
- [Python: csv.DictReader i DictWriter](https://docs.python.org/3/library/csv.html)
