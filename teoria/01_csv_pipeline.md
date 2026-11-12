# 01. CSV - budujemy pipeline zamowien

Zbudujesz pipeline, ktory odczyta `orders.csv`, zwaliduje rekordy i zapisze poprawne oraz odrzucone zamowienia do dwoch plikow.

## 1. Co to jest CSV i jakie dane przechowuje?

CSV to tekstowy format zapisu danych tabelarycznych. Rekord ma pola oddzielone uzgodnionym separatorem. Pierwszy rekord czesto zawiera nazwy kolumn, czyli naglowek.

CSV nie jest pythonowym typem takim jak list albo dict. To sposob zapisania danych poza programem. Po odczycie dopiero utworzymy z niego struktury Pythona.

Przyklad z pliku, na ktorym zbudujemy caly pipeline:

```csv
order_id;customer_name;status;amount;country;note
1001; Anna Kowalska ; PAID ;120,50; pl ;"Promocja; kod ""START"""
```

Mamy szesc kolumn. Srednik rozdziela pola, ale srednik w notatce nalezy do jej tresci. Cudzyslowy pozwalaja zapisac go bez tworzenia dodatkowej kolumny. Podwojny cudzyslow wewnatrz pola oznacza jeden cudzyslow w wartosci.

Po poprawnym odczycie note ma zawierac:

```text
Promocja; kod "START"
```

Dlatego nie rozbijamy pliku przez split(";"). Wykorzystamy parser, ktory zna reguly CSV.

### Separator, naglowek i konce linii

Nazwa CSV kojarzy sie z przecinkiem, ale eksport moze uzywac srednika lub innego separatora. W naszym pliku separator to srednik. Przecinek w 120,50 jest separatorem dziesietnym kwoty, nie granica kolumny.

Naglowek jest opisem kolumn, nie zamowieniem. Nie wliczamy go do liczby rekordow sprzedazy.

Rekord CSV nie zawsze jest jedna fizyczna linia tekstu. Pole w cudzyslowach moze zawierac nowa linie. Parser obsluguje to zgodnie z formatem; nie musimy samodzielnie odtwarzac tej logiki.

### Co z typami?

W uzytym przez nas odczycie CSV wartosci komorek staja sie stringami. ID 1001 nie bedzie jeszcze intem, a kwota nie bedzie Decimal.

```text
plik CSV -> lista slownikow ze stringami -> poprawny model zamowienia
```

Parser zna skladnie pliku, ale nie zna zasad sklepu. Potrafi odczytac bad jako tekst kwoty. To my ustalimy, ze taka wartosc nie jest poprawna.

## 2. Historia biznesowa i cel pipeline'u

DataShop codziennie przekazuje eksport zamowien. Ksiegowosc potrzebuje poprawnych rekordow, a dzial operacyjny listy odrzucen, zeby poprawic dane u zrodla.

Jeden rekord to jedno zamowienie.

Zbudujemy jeden proces:

```text
orders.csv
    -> odczyt i kontrola kolumn
    -> normalizacja oraz walidacja zamowien
    -> clean_orders.csv i rejected_orders.csv
```

## 3. Projekt i plik wejsciowy

**Plik cwiczeniowy:** [orders.csv](../materialy/01_csv/orders.csv).

Skopiuj go do projektu bez zmiany jego zawartosci.

**Umiesc w swoim projekcie:** `datashop_csv/data/raw/orders.csv`.

Otworzmy nowy, pusty projekt Python o nazwie datashop_csv. Potrzebujemy Pythona 3.10 lub nowszego. Csv, pathlib i decimal pochodza z biblioteki standardowej.

Utworz katalog `data/raw` w projekcie `datashop_csv` i skopiuj tam plik `orders.csv`. 

### Wejscie do calej lekcji

```csv
order_id;customer_name;status;amount;country;note
1001; Anna Kowalska ; PAID ;120,50; pl ;"Promocja; kod ""START"""
1002;Jan Nowak;pending;80.00;DE;
1003;Ola Lis;paid;0;;Zamowienie bezplatne
1004;Ewa Mazur;paid;bad;FR;Blad kwoty
1005;Anna Kowalska;cancelled;20.00;PL;
1006;Jan Nowak;paid;50,25; de ;Druga platnosc
1007;Piotr Kot;paid;NaN;PL;
```

Dwie kwoty sa celowo niepoprawne: bad i NaN. Jedno paid ma poprawna kwote zero. Te wartosci zostaja w raw; nie poprawiamy ich recznie.

### Docelowy projekt

```text
datashop_csv/
    ingestion.py
    transformations.py
    exporting.py
    pipeline.py
    reporting.py
    .env               # wartosci ustawien; dodajemy po pierwszym udanym odczycie
    config.py          # odczytuje .env
    main.py
    data/
        raw/
            orders.csv
        out/
            clean_orders.csv
            rejected_orders.csv
```

### Odpowiedzialnosci plikow od poczatku lekcji

| Plik | Za co odpowiada? |
|---|---|
| .env | zawiera sciezke wejsciowa i katalog wynikowy dla lokalnego uruchomienia |
| config.py | odczytuje .env, sprawdza ustawienia i przekazuje je jako obiekt PipelineConfig |
| main.py | pobiera konfiguracje, uruchamia proces i zleca pokazanie wyniku |
| pipeline.py | ustala kolejnosc etapow i przekazuje miedzy nimi dane |
| ingestion.py | przyjmuje i odczytuje plik, sprawdza strukture dostawy |
| transformations.py | normalizuje i waliduje rekordy |
| exporting.py | przygotowuje rekordy do formatu wyjscia i zapisuje pliki |
| reporting.py | formatuje podsumowanie w konsoli |


## 4. Kontrakt wejscia: co przyjmujemy?

Zanim napiszemy kod, uzgadniamy format z dostawca.

| Element | Nasza umowa |
|---|---|
| kodowanie | UTF-8, ewentualnie z poczatkowym BOM |
| separator | srednik |
| naglowek | dokladnie order_id/customer_name/status/amount/country/note, w tej kolejnosci |
| liczba komorek | szesc w kazdym rekordzie |
| country i note | komorka musi istniec, ale moze byc pusta |
| rozmiar | do 5 000 000 bajtow w tym cwiczeniu |
| gotowosc pliku | dostawca zakonczyl jego zapis |

Pusta komorka kraju to pusty string. Brak szostej komorki to cos innego: rekord ma zla strukture. Nie wypelniamy automatycznie brakujacych kolumn, zeby ukryc uszkodzony eksport.

Sam naglowek bez danych jest poprawnym pustym eksportem. Pusty plik bez naglowka nie spelnia tej umowy.

Limit rozmiaru jest zalozeniem tego malego projektu, a nie uniwersalna wartoscia dla kazdego procesu DE. Dzieki niemu jawnie mowimy, dla jakich dostaw wybralismy liste rekordow w pamieci.

## 5. Warstwa odczytu budowana krok po kroku


### Krok 1. Najpierw tylko odczytujemy otrzymany plik

Chcemy otrzymac liste rekordow.

Plik `ingestion.py` - krok 1:

```python
import csv
from pathlib import Path


def read_orders(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8", newline="") as file:
        reader = csv.DictReader(file, delimiter=";")
        rows = list(reader)

    return rows
```

#### Jak dziala ta pierwsza wersja?

Import csv udostepnia parser. Path jest klasa opisujaca sciezke.

Read_orders otrzymuje sciezke jako argument. Nie ustala samodzielnie, w ktorym katalogu jest plik. To decyzja main.

Open w trybie r otwiera plik do odczytu. Newline="" pozostawia interpretacje koncow linii parserowi CSV, a nie recznej obsludze tekstu.

With zamknie plik przy opuszczaniu bloku, takze przy wyjatku.

DictReader bierze nazwy z pierwszego rekordu i laczy je z komorkami kolejnych. Delimiter=";" ustala separator; przecinek w kwocie pozostaje czescia tekstu komorki.

List(reader) pobiera wszystkie rekordy, poki plik jest otwarty. Rows jest gotowa lista slownikow. Return poza with zwraca te liste juz po zamknieciu pliku. Nie zwracamy readera zaleznego od otwartego zasobu.

#### Oddzielamy przebieg procesu od prezentacji

Zanim uruchomimy odczyt, nadajmy plikom stale odpowiedzialnosci. Main nie bedzie recznie odczytywalo komorek ani budowalo komunikatow.

**Miejsce pracy:** tworzymy pipeline.py oraz reporting.py obok ingestion.py.

Plik `pipeline.py` - etap 1, sam odczyt:

```python
from pathlib import Path

from ingestion import read_orders


def run_pipeline(input_path: Path, output_dir: Path) -> dict:
    raw_orders = read_orders(input_path)
    return {
        "source_file": input_path.name,
        "input_rows_count": len(raw_orders),
    }
```

Run_pipeline jest orkiestratorem procesu: w tym momencie ma tylko jeden etap, read_orders.

W sygnaturze mamy juz output_dir, poniewaz umowa calego procesu obejmuje wejscie i katalog wynikowy. Na tym etapie ten argument jeszcze nie jest wykorzystywany. Nie tworzy katalogu i nie uruchamia zapisu. W dalszej czesci dodamy eksport bez zmieniania sposobu wywolania procesu z main.

Plik `reporting.py` - etap 1, podsumowanie odczytu:

```python
def print_summary(result: dict) -> None:
    print(f"Plik: {result['source_file']}")
    print(f"Odczytane rekordy: {result['input_rows_count']}")
```

Print_summary dostaje gotowy wynik i pokazuje go w konsoli. Nie otwiera CSV ani JSON i nie sprawdza danych. Tutaj jest miejsce na tekst komunikatow i f-stringi, nie w punkcie wejscia.

#### Cienki punkt wejscia main.py

**Miejsce pracy:** tworzymy main.py w katalogu glownym projektu. Na razie proces ma tylko przyjac i odczytac nasze rzeczywiste wejscie.

Plik `main.py` - pierwsza wersja, ze sciezkami do pierwszego uruchomienia:

```python
from pathlib import Path

from pipeline import run_pipeline
from reporting import print_summary


def main() -> None:
    base_dir = Path(__file__).resolve().parent
    input_path = base_dir / "data" / "raw" / "orders.csv"
    output_dir = base_dir / "data" / "out"

    result = run_pipeline(input_path, output_dir)
    print_summary(result)


if __name__ == "__main__":
    main()
```

#### Budowanie sciezki linia po linii

`from pathlib import Path` udostepnia klase do pracy ze sciezkami. `from pipeline import run_pipeline` i `from reporting import print_summary` udostepniaja dwie operacje punktu wejscia. Nie importujemy tutaj read_orders ani orders.csv.

`def main() -> None:` definiuje punkt wejscia programu. Samo zdefiniowanie funkcji nie uruchamia odczytu. Szczegolowa kolejnosc etapow znajduje sie w run_pipeline.

W `base_dir = Path(__file__).resolve().parent`:

| Fragment | Znaczenie |
|---|---|
| __file__ | sciezka do pliku main.py, w ktorym jest instrukcja |
| Path(__file__) | obiekt opisujacy te sciezke |
| resolve() | sciezka bezwzgledna z rozwiazanymi m.in. .. i dowiazaniami |
| parent | katalog zawierajacy main.py |

Base_dir jest zwykla nasza zmienna, nie slowem kluczowym. Parent jest atrybutem, wiec nie ma nawiasow; resolve jest metoda, wiec je ma.

Dalej `base_dir / "data" / "raw" / "orders.csv"` dolacza kolejne czesci sciezki.


Wynik:

```text
Plik: orders.csv
Odczytane rekordy: 7
```

**Punkt kontrolny:** program naprawde odczytal nasz plik, ale jeszcze nie stwierdzil, czy siedem zamowien jest poprawnych. Nie nazywamy tego siedmioma poprawnymi sprzedazami.

### Krok 1a. Przenosimy sciezki do konfiguracji

Program odczytal siedem rekordow. Ale od teraz sciezki trzymamy w `.env`, zeby oddzielic konfiguracje od kodu. Dzieki temu `main.py` nie musi znac katalogow projektu, a dane wejsciowe i raporty maja osobne miejsca: `data/raw` i `data/out`.

**1. Tworzymy plik `.env` obok `main.py`** w glownym katalogu `datashop_csv`. W PyCharm: prawy przycisk na katalogu projektu, **New > File**, nazwa `.env`. Zaczyna sie od kropki i nie ma koncowki `.txt`. To zwykly plik tekstowy, nie skrypt Pythona ani katalog `.venv`.

```dotenv
INPUT_PATH=data/raw/orders.csv
OUTPUT_DIR=data/out
```

Kazda linia ma postac `NAZWA=wartosc`. Po lewej jest nazwa ustawienia, po prawej sciezka zapisana jako tekst. Sciezki sa wzgledne wobec katalogu projektu, a zapis z `/` dziala takze na Windows. W projekcie przechowywanym w Git dopisujemy `.env` do `.gitignore`. Do repozytorium mozemy dodac `.env.example` z przykladowymi wartosciami bez sekretow. Program odczytuje `.env`, a nie `.env.example`.

**2. Instalujemy biblioteke przez pip i tworzymy `config.py` obok `.env`.** W terminalu PyCharma, w srodowisku projektu:

```powershell
python -m pip install python-dotenv
```
Nastepnie tworzymy plik config.py
```python
from dataclasses import dataclass
from pathlib import Path

from dotenv import dotenv_values


@dataclass(frozen=True)
class PipelineConfig:
    input_path: Path
    output_dir: Path


def load_config() -> PipelineConfig:
    base_dir = Path(__file__).resolve().parent
    env_path = base_dir / ".env"

    with env_path.open(encoding="utf-8") as file:
        settings = dotenv_values(stream=file, interpolate=False)

    input_value = settings.get("INPUT_PATH")
    output_value = settings.get("OUTPUT_DIR")
    if input_value is None or not input_value.strip():
        raise ValueError("Set INPUT_PATH in .env")
    if output_value is None or not output_value.strip():
        raise ValueError("Set OUTPUT_DIR in .env")

    return PipelineConfig(
        input_path=base_dir / input_value,
        output_dir=base_dir / output_value,
    )
```

- `PipelineConfig` to typ z dwoma nazwanymi polami. `@dataclass` tworzy konstruktor, dzieki czemu mozemy przekazac `input_path=...` i `output_dir=...` bez pisania wlasnego `__init__`.
- `frozen=True` blokuje ponowne przypisanie pol do obiektu w trakcie uruchomienia. Nie blokuje edycji `.env`: przy kolejnym uruchomieniu program odczyta ten plik od nowa.
- `Path(__file__).resolve().parent` wskazuje katalog zawierajacy `config.py`, wiec `base_dir / ".env"` wskazuje nasz plik niezaleznie od katalogu roboczego terminala.
- `with` otwiera ten konkretny plik i zamyka go po odczycie. `dotenv_values(stream=file, interpolate=False)` czyta pary nazwa-wartosc do slownika. `interpolate=False` sprawia, ze wartosci traktujemy doslownie, bez podstawiania np. `${HOME}` z otoczenia. Brak pliku powoduje `FileNotFoundError` dla `.env`.
- `settings.get("INPUT_PATH")` daje np. tekst `"data/raw/orders.csv"`. Brak klucza daje `None`, a `INPUT_PATH=` daje pusty tekst. Odrzucamy oba przypadki przed odczytem zamowien. To wazne, bo dolaczenie pustego tekstu do `base_dir` daloby katalog projektu zamiast sciezki do pliku.
- `base_dir / input_value` tworzy obiekt `Path`. Analogicznie powstaje `output_dir`. Adnotacja `Path` nie konwertuje tekstu sama. Na koniec zwracamy oba obiekty w `PipelineConfig`; jeszcze nie otwieramy danych i nie tworzymy katalogu wynikowego.

**3. Zastepujemy caly `main.py`** ponizsza wersja. Nie zmieniamy `ingestion.py`, `pipeline.py` ani `reporting.py`.

```python
from config import load_config
from pipeline import run_pipeline
from reporting import print_summary


def main() -> None:
    config = load_config()
    result = run_pipeline(config.input_path, config.output_dir)
    print_summary(result)


if __name__ == "__main__":
    main()
```

`config = load_config()` pobiera ustawienia dla tego uruchomienia. `config.input_path` i `config.output_dir` pobieraja dwa pola obiektu. `run_pipeline` nadal dostaje dwa obiekty `Path`, tak jak przed zmiana. Nie musi importowac `PipelineConfig` ani wiedziec, skad przyszly ustawienia. `print_summary` nadal pokazuje wynik, a blok `if __name__ == "__main__"` uruchamia funkcje tylko wtedy, gdy wykonujemy ten plik jako skrypt.

**4. Sprawdzamy, ze program pobiera ustawienia z pliku.** W terminalu w katalogu projektu:

```powershell
python main.py
```

```text
Plik: orders.csv
Odczytane rekordy: 7
```

Wynik zostal taki sam, ale ustawienia pochodza juz z `.env`. Na tym etapie nadal nie powstaje `data/out`, bo pipeline jeszcze niczego nie zapisuje. Zostaw `INPUT_PATH=data/raw/orders.csv` i `OUTPUT_DIR=data/out` do konca lekcji.

### Krok 2. Dodajemy kontrole naglowka

Samo odczytanie czegos do slownikow nie oznacza, ze mamy oczekiwane kolumny. Plik z price zamiast amount tez moglby zostac odczytany, ale dalsze funkcje nie dostalyby potrzebnego pola.

Zapiszmy uzgodnione nazwy i porownajmy je z naglowkiem parsera.

Plik `ingestion.py` - krok 2, zastap poprzednia wersje:

```python
import csv
from pathlib import Path


EXPECTED_COLUMNS = ["order_id", "customer_name", "status", "amount", "country", "note"]


def read_orders(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8", newline="") as file:
        reader = csv.DictReader(file, delimiter=";")
        if reader.fieldnames != EXPECTED_COLUMNS:
            raise ValueError(f"Unexpected CSV header: {reader.fieldnames!r}")

        rows = list(reader)

    return rows
```

#### Co konkretnie sie zmienilo?

EXPECTED_COLUMNS jest lista nazw w uzgodnionej kolejnosci. Wielkie litery sa konwencja dla stalej konfiguracji, nie specjalna instrukcja.

Reader.fieldnames udostepnia naglowek. Porownanie dwoch list sprawdza wartosci i kolejnosc. To rygorystyczna umowa tego projektu, a nie wymog kazdego odczytu CSV.

Gdy naglowek jest inny, raise ValueError konczy ta probe odczytu. Nie przechodzimy do list(reader), nie zwracamy [] i nie udajemy pustej sprzedazy. With nadal zamknie plik.

#### Sprawdzamy na tym samym main

Po `python main.py` widzimy 7 rekordow. Naglowek `orders.csv` zgadza sie z `EXPECTED_COLUMNS`, wiec kontrola pozwala przejsc do odczytu danych.

### Krok 3. Sprawdzamy liczbe komorek kazdego rekordu

Poprawny naglowek nie gwarantuje, ze kazdy kolejny rekord ma szesc komorek. Parser moze uzupelnic brakujaca wartosc None albo umiescic nadmiarowe komorki pod dodatkowym kluczem None.

Potrzebujemy obejrzec kazdy surowy rekord przed dopisaniem go do wyniku.

Plik `ingestion.py` - krok 3:

```python
import csv
from pathlib import Path


EXPECTED_COLUMNS = ["order_id", "customer_name", "status", "amount", "country", "note"]


def read_orders(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8", newline="") as file:
        reader = csv.DictReader(file, delimiter=";")
        if reader.fieldnames != EXPECTED_COLUMNS:
            raise ValueError(f"Unexpected CSV header: {reader.fieldnames!r}")

        rows = []
        for record_number, row in enumerate(reader, start=1):
            if None in row or any([value is None for value in row.values()]):
                raise ValueError(f"Wrong number of cells at record {record_number}")
            rows.append(row)

    return rows
```

#### Dlaczego zamiast list(reader) mamy petle?

List(reader) od razu budowalo liste bez naszej kontroli pojedynczego rekordu. Teraz jawnie tworzymy rows, sprawdzamy row i dopiero potem wykonujemy append.

Enumerate(reader, start=1) daje pozycje danych od 1, bez naglowka. Nie jest to gwarantowany numer fizycznej linii, bo pole CSV moze zawierac nowa linie.

None in row sprawdza klucze slownika. Dodatkowy klucz None oznacza nadmiarowe komorki.

Comprehension sprawdza value is None dla kazdej wartosci, a any odpowiada, czy choc jedna komorka jest brakujaca. Pusty string "" nie jest None. Dlatego poprawny pusty kraj w ID 1003 nadal przejdzie.

To kontrola struktury, nie jeszcze reguly kraju UNKNOWN. Ta normalizacja bedzie nalezala do transformations.

#### Sprawdzenie kroku

Po `python main.py` nadal widzimy 7 rekordow. W zamowieniu 1003 kraj jest pusty, ale separator zachowuje miejsce tej komorki. To poprawna struktura, dlatego zamowienie przechodzi kontrole.

### Krok 4. Zaostrzamy reakcje parsera na uszkodzony zapis CSV

Do tej pory dodawalismy wlasne kontrole. Teraz zmienimy konfiguracje samego parsera.

Plik `ingestion.py` - krok 4:

```python
import csv
from pathlib import Path


EXPECTED_COLUMNS = ["order_id", "customer_name", "status", "amount", "country", "note"]


def read_orders(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8", newline="") as file:
        reader = csv.DictReader(file, delimiter=";", strict=True)
        if reader.fieldnames != EXPECTED_COLUMNS:
            raise ValueError(f"Unexpected CSV header: {reader.fieldnames!r}")

        rows = []
        for record_number, row in enumerate(reader, start=1):
            if None in row or any([value is None for value in row.values()]):
                raise ValueError(f"Wrong number of cells at record {record_number}")
            rows.append(row)

    return rows
```

#### Co daje strict=True?

Nowa czesc to tylko strict=True przy DictReader. Wlacza bardziej rygorystyczna reakcje parsera na wykrywane niepoprawne zapisy CSV.

Domyslny parser moze byc tolerancyjny. Nie chcemy opierac poprawnosci dostawy na tym, ze z niedomknietego pola udalo mu sie utworzyc jakis tekst.

Strict nie sprawdza za nas statusu, kwoty, naglowka ani calego schematu. Wczesniejsze kontrole zostaja.

#### Sprawdzenie kroku

Po `python main.py` nadal widzimy 7 rekordow. Notatka zamowienia 1001 ma srednik i wewnetrzne cudzyslowy, ale jest poprawnie zapisana. `strict=True` jej nie odrzuca.

### Krok 5. Dodajemy limit rozmiaru dostawy

Dzialajacy odczyt nadal tworzy cala liste w pamieci. Chcemy ograniczyc rozmiar przyjmowanego pliku przed jego otwarciem.

Plik `ingestion.py` - krok 5, wersja docelowa:

```python
import csv
from pathlib import Path


EXPECTED_COLUMNS = ["order_id", "customer_name", "status", "amount", "country", "note"]


def read_orders(path: Path, max_bytes: int = 5_000_000) -> list[dict]:
    if path.stat().st_size > max_bytes:
        raise ValueError("Input file exceeds the agreed size limit")

    with path.open("r", encoding="utf-8-sig", newline="") as file:
        reader = csv.DictReader(file, delimiter=";", strict=True)
        if reader.fieldnames != EXPECTED_COLUMNS:
            raise ValueError(f"Unexpected CSV header: {reader.fieldnames!r}")

        rows = []
        for record_number, row in enumerate(reader, start=1):
            if None in row or any([value is None for value in row.values()]):
                raise ValueError(f"Wrong number of cells at record {record_number}")
            rows.append(row)

    return rows
```

#### Co dodalismy?

Funkcja ma opcjonalny argument max_bytes z domyslnym limitem 5 000 000 bajtow. Wczesniejsze wywolanie read_orders(input_path) nadal dziala, bo drugi argument ma wartosc domyslna.

Path.stat().st_size odczytuje rozmiar pliku w bajtach. Gdy przekracza limit, raise konczy probe przed open i budowaniem listy rekordow. Brak pliku moze juz tutaj zglosic FileNotFoundError.

Podkreslenia w 5_000_000 sluza czytelnosci liczby. Nie zmieniaja jej wartosci.

To umowa dla naszego malego projektu, nie uniwersalna rekomendacja rozmiaru. Rozmiar tekstu nie jest scislym limitem pamieci struktur Pythona. Plik moze tez zmienic sie po stat, dlatego nadal wymagamy gotowej, niezmienianej dostawy.

`utf-8-sig` zachowuje zgodnosc z UTF-8 i pomija poczatkowy znacznik BOM, jesli wystepuje. Na naszym `orders.csv` nie zmienia odczytanych wartosci.

#### Sprawdzenie limitu na tym samym pliku

Na tym samym `orders.csv` zmieniamy jednorazowo w pipeline.py wywolanie na `raw_orders = read_orders(input_path, max_bytes=1)`. Main zostaje bez zmian.

Po python main.py oczekujemy ValueError: Input file exceeds the agreed size limit. To potwierdza, ze limit zostal uzyty.

W pipeline.py przywracamy `raw_orders = read_orders(input_path)`. Program znowu odczytuje 7 rekordow.

## 6. Punkt kontrolny: odczyt jest gotowy do dalszego pipeline'u

Uruchamiamy python main.py i widzimy:

```text
Plik: orders.csv
Odczytane rekordy: 7
```

Dalsze sekcje korzystaja z jednej stabilnej funkcji read_orders.

## 7. Model zamowienia i reguly transformacji

Teraz ustalamy, do jakich danych ma doprowadzic pipeline.

| Pole | Po normalizacji |
|---|---|
| order_id | dodatni int z tekstu akceptowanego przez int |
| customer_name | niepusty tekst bez skrajnych spacji |
| status | paid, pending albo cancelled, malymi literami |
| amount | skonczony nieujemny Decimal |
| country | wielkie litery; pusty tekst daje UNKNOWN |
| note | tekst bez skrajnych spacji; moze byc pusty |

Kwoty maja jeden separator dziesietny, kropke albo przecinek. Nie obslugujemy tu separatorow tysiecy ani zgadywania lokalnych formatow.

Zero jest poprawna kwota. Bad jest blednym zapisem. NaN da sie zamienic na Decimal, ale nie jest skonczona kwota. Nie wolno zamieniac tych bledow na zero.

Identyfikatory maja byc unikalne wsrod zaakceptowanych zamowien. Zachowujemy pierwsze poprawne wystapienie ID. Bledny rekord nie rezerwuje ID, bo nie zostal jeszcze przyjety.

**Miejsce pracy:** tworzymy transformations.py. Funkcje otrzymuja slowniki; nie otwieraja plikow.

Plik `transformations.py`:

```python
from decimal import Decimal, InvalidOperation


def normalize_order(raw: dict) -> dict:
    for field in ("order_id", "customer_name", "status", "amount", "country", "note"):
        if not isinstance(raw[field], str):
            raise TypeError(f"{field} must be text")

    order_id = int(raw["order_id"].strip())
    if order_id <= 0:
        raise ValueError("order_id must be positive")

    customer_name = raw["customer_name"].strip()
    if customer_name == "":
        raise ValueError("customer_name must not be empty")

    status = raw["status"].strip().lower()
    if status not in {"paid", "pending", "cancelled"}:
        raise ValueError("Unsupported status")

    try:
        amount = Decimal(raw["amount"].strip().replace(",", "."))
    except InvalidOperation as exc:
        raise ValueError("Invalid amount") from exc
    if not amount.is_finite() or amount < 0:
        raise ValueError("Amount must be finite and non-negative")

    return {
        "order_id": order_id,
        "customer_name": customer_name,
        "status": status,
        "amount": amount,
        "country": raw["country"].strip().upper() or "UNKNOWN",
        "note": raw["note"].strip(),
    }


def transform_orders(raw_orders: list[dict], source_file: str) -> tuple[list[dict], list[dict]]:
    clean = []
    rejected = []
    accepted_ids = set()

    for record_number, raw in enumerate(raw_orders, start=1):
        try:
            order = normalize_order(raw)
            if order["order_id"] in accepted_ids:
                raise ValueError("Duplicate order_id")
        except (KeyError, TypeError, ValueError) as exc:
            rejected.append({
                "source_file": source_file,
                "record_number": record_number,
                "raw_order_id": raw.get("order_id"),
                "error_type": type(exc).__name__,
                "reason": str(exc),
            })
        else:
            accepted_ids.add(order["order_id"])
            clean.append(order)

    return clean, rejected
```

### Jak czytamy normalize_order?

Najpierw wymagamy stringow we wszystkich szesciu komorkach. To zgodne z kontraktem naszego CSV. Nie probujemy naprawiac struktury, ktora powinna zostac odrzucona w ingestion.

Int konwertuje ID; dodatni zakres sprawdzamy osobnym ifem. Skladnia akceptowana przez int nie jest tym samym co wymog samych cyfr ASCII, dlatego nasz kontrakt mowi wprost, jaka konwersje stosujemy.

Strip usuwa tylko brzegowe biale znaki. Nie wycina spacji z srodka nazwiska ani srednika z notatki. Lower i upper ujednolicaja wybrane pola, nie wszystko naraz.

Decimal powstaje z oczyszczonego tekstu, bez przechodzenia przez float. InvalidOperation zamieniamy na ValueError z czytelnym opisem i zachowaniem przyczyny przez from exc.

Najpierw sprawdzamy is_finite, potem nieujemnosc. Przy NaN pierwsza czesc or jest prawdziwa, wiec nie wykonujemy niepoprawnego porownania z zerem.

Return tworzy nowy slownik. Zrodlo nie zostaje nadpisane.

### Jak czytamy transform_orders?

Accepted_ids jest nowym setem dla kazdego uruchomienia, nie globalnym stanem pozostajacym z wczoraj.

W try normalizujemy jeden rekord i sprawdzamy jego ID. Oczekiwane bledy tego rekordu trafiaja do rejected. Else dopisuje poprawny rekord i dopiero wtedy rezerwuje ID.

Odrzucenie zawiera nazwe pliku, pozycje rekordu, oryginalne ID, typ bledu i przyczyne. Nie kopiujemy do komunikatu calego rekordu klienta.

Wyjatki obslugujemy przy pojedynczym rekordzie. Nie otaczamy calego programu except Exception, ktore mogloby ukryc awarie wejscia lub literowke w kodzie.

Return oddaje krotke dwoch list. Przy nieoczekiwanym bledzie implementacji nie obiecujemy automatycznej kontynuacji.

## 8. Dodajemy transformacje do pipeline'u, nie do main

**Miejsce pracy:** zapisujemy transformations.py. Zmieniamy pipeline.py i reporting.py. Main.py zostaje dokladnie takie samo.

Plik `pipeline.py` - etap 2, odczyt i transformacja:

```python
from pathlib import Path

from ingestion import read_orders
from transformations import transform_orders


def run_pipeline(input_path: Path, output_dir: Path) -> dict:
    raw_orders = read_orders(input_path)
    clean, rejected = transform_orders(raw_orders, input_path.name)
    return {
        "source_file": input_path.name,
        "input_rows_count": len(raw_orders),
        "clean_orders": clean,
        "rejected_orders": rejected,
    }
```

Run_pipeline najpierw wywoluje odczyt, potem transformacje i przekazuje ich wyniki dalej. Nie implementuje regul statusu, kwoty i duplikatow. To nadal odpowiedzialnosc transformations.py.

Wynik tego etapu zawiera dwie listy: clean_orders i rejected_orders. Chcemy je teraz obejrzec, zanim dodamy eksport.

Plik `reporting.py` - etap 2, podglad wyniku transformacji:

```python
def print_summary(result: dict) -> None:
    clean = result["clean_orders"]
    rejected = result["rejected_orders"]

    print(f"Poprawne ID: {[order['order_id'] for order in clean]}")
    print(f"Odrzucone rekordy: {[row['record_number'] for row in rejected]}")
    print(f"Poprawne: {len(clean)}, odrzucone: {len(rejected)}")
```

Wlasnie tutaj przygotowujemy liste ID do pokazania i wypisujemy pozycje odrzucen. Comprehension sluzy prezentacji wybranych pol. Nie zmienia rekordow, nie filtruje sprzedazy ani nie wylicza przychodu.

Len w tym podgladzie opisuje otrzymane listy. Reguly, ktore zdecydowaly o ich zawartosci, zostaly wykonane w transformacji, nie w reporting.

Uruchamiamy niezmienione main.py:

```powershell
python main.py
```

Wynik:

```text
Poprawne ID: [1001, 1002, 1003, 1005, 1006]
Odrzucone rekordy: [4, 7]
Poprawne: 5, odrzucone: 2
```

**Punkt kontrolny:** kwota zero ma pozostac poprawna. Pending i cancelled tez sa poprawnymi rekordami. Raw nadal zawiera bad i NaN; program tworzy osobny wynik, nie poprawia historii dostawy.

Rozszerzylismy proces bez dopisywania do main ani jednej reguly i bez przenoszenia tam printow. Main nadal pobiera ustawienia, uruchamia proces i zleca prezentacje.

## 9. Zapisujemy poprawne i odrzucone zamowienia

Transformacja zwraca juz dwie listy: `clean` i `rejected`. Zapiszemy kazda do osobnego pliku CSV. Odrzucenia maja inne kolumny, bo oprocz ID potrzebujemy powodu bledu.

Utworz `exporting.py` obok `pipeline.py`:

```python
import csv
from pathlib import Path


def write_csv(path: Path, rows: list[dict], columns: list[str]) -> None:
    with path.open("w", encoding="utf-8", newline="") as file:
        writer = csv.DictWriter(file, fieldnames=columns, delimiter=";")
        writer.writeheader()
        writer.writerows(rows)


def export_orders(output_dir: Path, clean: list[dict], rejected: list[dict]) -> None:
    output_dir.mkdir(parents=True, exist_ok=True)
    write_csv(
        output_dir / "clean_orders.csv",
        clean,
        ["order_id", "customer_name", "status", "amount", "country", "note"],
    )
    write_csv(
        output_dir / "rejected_orders.csv",
        rejected,
        ["source_file", "record_number", "raw_order_id", "error_type", "reason"],
    )
```

`mkdir` tworzy katalog wynikowy, jesli go jeszcze nie ma. `DictWriter` zapisuje slowniki w kolejnosci podanej w `columns`: najpierw naglowek, potem dane. Tryb `"w"` nadpisuje poprzedni wynik zamiast dopisywac kolejne wiersze.

W `pipeline.py` dodaj import:

```python
from exporting import export_orders
```

W `run_pipeline()` dodaj tylko wywolanie `export_orders()`, zaraz po transformacji i przed `return`:

```python
    clean, rejected = transform_orders(raw_orders, input_path.name)
    export_orders(output_dir, clean, rejected)
```

`main.py` i `reporting.py` pozostaja bez zmian. Uruchom:

```powershell
python main.py
```

W `data/out` sprawdz dwa pliki:

| Plik | Oczekiwany wynik |
|---|---|
| `clean_orders.csv` | 5 rekordow: 1001, 1002, 1003, 1005 i 1006 |
| `rejected_orders.csv` | 2 rekordy: 1004 (`Invalid amount`) i 1007 (`Amount must be finite and non-negative`) |
