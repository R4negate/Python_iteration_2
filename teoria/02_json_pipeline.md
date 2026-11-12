# 02. JSON - budujemy pipeline zagniezdzonych zamowien

[Zadanie: zbuduj pipeline](../zadania/02_pipeline_json.md)

Zbudujesz pipeline, ktory odczyta `orders.json`, zwaliduje zamowienia i zapisze poprawne oraz odrzucone rekordy do dwoch plikow.

## 1. Co to jest JSON i jakie typy danych opisuje?

JSON jest tekstowym formatem wymiany danych. Pozwala zapisac obiekt z nazwanymi polami, liste elementow i wartosci roznych typow. W przeciwienstwie do tabelarycznego CSV moze bezposrednio opisac klienta wewnatrz zamowienia albo liste tagow tego zamowienia.

JSON nie jest pythonowym slownikiem na dysku. To osobny zapis tekstowy, z ktorego parser dopiero utworzy struktury Pythona.

| Element JSON | Przyklad w naszej dostawie | Po odczycie w Pythonie |
|---|---|---|
| obiekt | customer z jego polami | dict |
| tablica | orders albo tags | list |
| string | "paid", "120,50" | str |
| liczba calkowita | 1001 | int |
| liczba z czescia dziesietna | 120.50 bez cudzyslowow | domyslnie float |
| wartosc logiczna | true albo false w is_gift | True albo False |
| brak wartosci | null w country lub tags | None |

Nawiasy klamrowe oznaczaja obiekt, a kwadratowe tablice. Nazwy pol i stringi maja podwojne cudzyslowy. True i None sa zapisem Pythona, nie zamiennikiem true i null w JSON.

W poprawnym JSON nie dopisujemy komentarzy ani przecinka po ostatnim elemencie. Wciecia i nowe linie poprawiaja czytelnosc, ale nie wyznaczaja rekordow jak w prostym pliku tekstowym.

### Liczba i string z liczba to rozne wartosci

`1001` stanie sie intem. `"1001"` pozostanie stringiem. `"false"` jest tekstem, a `false` wartoscia logiczna. Parser zachowuje te roznice; nie zgaduje, co autor mial na mysli.

Dla kwot w tym projekcie dostawca przekazuje string, np. "120,50". Potem jawnie utworzymy Decimal. Nie pozwalamy, zeby cena najpierw stala sie przyblizonym floatem.

To umowa naszej dostawy, a nie zasada, ze kazdy JSON musi miec ceny w stringach. Przy liczbach dziesietnych bez cudzyslowow mozna np. skonfigurowac parse_float=Decimal. W tej lekcji nie zmieniamy jednak kontraktu w polowie procesu.

### Czy kazdy JSON jest lista rekordow?

Nie. Caly dokument moze miec inna glowna wartosc. Nasz pipeline wymaga konkretnej struktury: obiektu z waluta i lista orders. Poprawna skladnia JSON nie zapewnia takiego schematu automatycznie.

## 2. Historia biznesowa i jeden model zamowienia

DataShop przekazuje zamowienia w zagniezdzonym JSON-ie. Dane klienta sa w obiekcie `customer`, a tagi w tablicy `tags`. Ksiegowosc potrzebuje poprawnych rekordow, a dzial operacyjny listy odrzucen z powodami.

```text
orders.json
    -> odczyt i kontrola struktury dokumentu
    -> normalizacja oraz walidacja zamowien
    -> clean_orders.json i rejected_orders.json
```

## 3. Projekt i plik wejsciowy

**Plik cwiczeniowy:** [orders.json](../materialy/02_json/orders.json).

To jedyny plik wejsciowy potrzebny w tej lekcji. Skopiuj go do projektu bez zmiany zawartosci.

**Umiesc w swoim projekcie:** `datashop_json/data/raw/orders.json`.

Otworzmy nowy projekt Python datashop_json. Wybieramy interpreter Python 3.10 lub nowszy. JSON, pathlib i decimal sa w bibliotece standardowej.

Uruchamiaj `python main.py` w terminalu PyCharma, w katalogu `datashop_json`. Nie potrzebujesz gotowego projektu CSV.

Utworz `data/raw` w projekcie `datashop_json` i skopiuj tam plik `orders.json`. Nie wklejamy tekstu dokumentu do kodu Pythona i nie zmieniamy null na None.

Pozostale pliki beda powstawaly na kolejnych etapach:

```text
datashop_json/
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
            orders.json
        out/
            clean_orders.json
            rejected_orders.json
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

## 4. Jak czytamy rzeczywisty dokument?

Przyklad struktury z pierwszym zamowieniem. Dolaczony `orders.json` ma siedem zamowien; do projektu skopiuj caly plik.

```json
{
  "source": "datashop",
  "currency": "PLN",
  "orders": [
    {
      "order_id": 1001,
      "customer": {
        "customer_id": 7,
        "name": " Anna Kowalska ",
        "country": " pl "
      },
      "status": " PAID ",
      "amount": "120,50",
      "is_gift": false,
      "tags": [" promo ", "PROMO", "sql"]
    }
  ]
}
```

Patrzac na ten dokument, rozdzielamy poziomy:

```text
obiekt glowny
    source: opis zrodla
    currency: waluta calej dostawy
    orders: lista
        jedno zamowienie
            order_id
            customer: obiekt
                customer_id
                name
                country
            status
            amount
            is_gift
            tags: lista
```

Source i currency nie sa zamowieniami. Nie zwiekszaja liczby sprzedazy.

Po odczycie payload["orders"] bedzie lista, payload["orders"][0] pierwszym slownikiem, a payload["orders"][0]["customer"] slownikiem klienta.

Dostep do kraju oznacza przejscie po kluczach orders, potem elemencie listy i kluczach customer oraz country. Nie laczymy tych nazw w jeden klucz "customer.country".

NaN w amount jest tutaj stringiem. Dokument daje sie odczytac, ale taka kwota zostanie odrzucona na etapie danych biznesowych.

## 5. Warstwa odczytu JSON budowana krok po kroku

Najpierw odczytujemy `orders.json`, potem dokladamy kolejne kontrole. Po kazdym kroku uruchom `python main.py` na tym samym pliku.

**Miejsce pracy:** ingestion.py, a w kroku 1 takze pipeline.py, reporting.py i main.py w glownym katalogu datashop_json.

Kazdy blok `ingestion.py` zastepuje poprzednia wersje tego pliku.

### Krok 1. Odczytujemy otrzymany dokument

Na poczatku korzystamy z wiedzy o naszym orders.json: jest obiektem z lista orders. Jeszcze tego nie sprawdzamy wlasnymi warunkami. Parser ma odczytac dokument, a main policzyc otrzymane elementy.

Plik `ingestion.py` - krok 1:

```python
import json
from pathlib import Path


def read_orders(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8") as file:
        payload = json.load(file)

    return payload["orders"]
```

#### Jak czytamy te linie?

Import json udostepnia parser formatu, a Path klase reprezentujaca sciezke.

Read_orders otrzymuje lokalizacje od main. Open w trybie r otwiera plik do odczytu. Utf-8 jest kodowaniem naszego pierwszego wejscia. Nie zmieniamy danych raw.

With zamyka plik przy opuszczaniu bloku, takze gdy json.load zglosi blad. Nie potrzebujemy recznego close na koncu sciezki sukcesu.

Json.load(file) zamienia zapis dokumentu na struktury Pythona. Nie wykonuje go jak kodu. Do payload trafia w tym przypadku slownik z source, currency i orders.

Return payload["orders"] wybiera liste z obiektu glownego i przekazuje ja dalej. Nie zwracamy otwartego pliku. Dane sa juz w pamieci.

Adnotacja list[dict] opisuje oczekiwany wynik, ale nie sprawdza typow podczas wykonania. Pierwsza wersja zaklada ksztalt znanego wejscia; zaraz uszczelnimy te umowe.

Na naszym wejsciu `payload` jest slownikiem z trzema kluczami, a `payload["orders"]` lista siedmiu zamowien. `payload["orders"][0]["customer"]["country"]` prowadzi kolejno od calego dokumentu do listy, pierwszego zamowienia, slownika klienta i wreszcie tekstu `" pl "`. Po odczycie ten tekst nadal ma spacje, a `amount` pierwszego rekordu nadal jest stringiem `"120,50"`. Parser odtwarza zapisane dane, ale ich nie oczyszcza.

#### Load a loads w tym konkretnym procesie

Load dostaje obiekt pliku. Loads dostaloby string z JSON. Litera s nie oznacza wielu rekordow.

Mamy juz otwarty plik, dlatego wybieramy load. Nie odczytujemy najpierw file.read(), zeby zaraz ponownie czytac ten sam zasob przez load. Pozycja odczytu moglaby byc na koncu.

Nie musimy tez tworzyc dodatkowego stringa przez read tylko po to, zeby przekazac go do loads.

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

Run_pipeline jest orkiestratorem procesu: w tym momencie ma tylko jeden etap, read_orders. Zbiera nazwe dostawy i liczbe rekordow do wyniku uruchomienia. Nie zamienia komorek ani nie formatuje tekstu dla odbiorcy.

W sygnaturze mamy juz output_dir, poniewaz umowa calego procesu obejmuje wejscie i katalog wynikowy. Na tym etapie ten argument jeszcze nie jest wykorzystywany. Nie tworzy katalogu i nie uruchamia zapisu. W dalszej czesci dodamy eksport bez zmieniania sposobu wywolania procesu z main.

Plik `reporting.py` - etap 1, podsumowanie odczytu:

```python
def print_summary(result: dict) -> None:
    print(f"Plik: {result['source_file']}")
    print(f"Odczytane rekordy: {result['input_rows_count']}")
```

Print_summary dostaje gotowy wynik i pokazuje go w konsoli. Nie otwiera CSV ani JSON i nie sprawdza danych. Tutaj jest miejsce na tekst komunikatow i f-stringi, nie w punkcie wejscia.

#### Cienki punkt wejscia main.py

**Miejsce pracy:** tworzymy main.py obok ingestion.py.

Plik `main.py` - pierwsza wersja, ze sciezkami do pierwszego uruchomienia:

```python
from pathlib import Path

from pipeline import run_pipeline
from reporting import print_summary


def main() -> None:
    base_dir = Path(__file__).resolve().parent
    input_path = base_dir / "data" / "raw" / "orders.json"
    output_dir = base_dir / "data" / "out"

    result = run_pipeline(input_path, output_dir)
    print_summary(result)


if __name__ == "__main__":
    main()
```

Sciezka ma ta sama odpowiedzialnosc co w poprzedniej lekcji. __file__ oznacza main.py, resolve wyznacza bezwzgledna lokalizacje, a parent wybiera jego katalog. Przez / dolaczamy data/raw/orders.json.

To nie jest odczyt sciezki customer z dokumentu. Lokalizacja pliku i struktura danych wewnatrz pliku to dwa rozne zagadnienia.

Input_path.name daje orders.json. Pipeline.py zbiera nazwe i liczbe elementow orders, nie kluczy w obiekcie glownym. Reporting.py wypisuje te informacje. Main jedynie przekazuje sciezki do run_pipeline i wynik do print_summary. Output_dir na tym etapie nie powoduje jeszcze zapisu.

Terminal w katalogu datashop_json:

```powershell
python main.py
```

Wynik:

```text
Plik: orders.json
Odczytane rekordy: 7
```

**Punkt kontrolny:** poprawny odczyt siedmiu obiektow nie oznacza siedmiu poprawnych zamowien. Najpierw sprawdzimy strukture dostawy, a dopiero przy transformacji sprawdzimy wartosci poszczegolnych zamowien.

### Krok 1a. Przenosimy sciezki do konfiguracji

Program odczytal siedem rekordow. Przenosimy sciezki do `.env`, zeby oddzielic konfiguracje od kodu. Przez cala lekcje pozostajemy przy `orders.json`.

**1. Tworzymy plik `.env` obok `main.py`** w glownym katalogu `datashop_json`. W PyCharm: prawy przycisk na katalogu projektu, **New > File**, nazwa `.env`. Zaczyna sie od kropki i nie ma koncowki `.txt`. To zwykly plik tekstowy, nie skrypt Pythona ani katalog `.venv`.

```dotenv
INPUT_PATH=data/raw/orders.json
OUTPUT_DIR=data/out
```

Kazda linia ma postac `NAZWA=wartosc`. Po lewej jest nazwa ustawienia, po prawej sciezka zapisana jako tekst. Sciezki sa wzgledne wobec katalogu projektu, a zapis z `/` dziala takze na Windows. W projekcie przechowywanym w Git dopisujemy `.env` do `.gitignore`. Do repozytorium mozemy dodac `.env.example` z przykladowymi wartosciami bez sekretow. Program odczytuje `.env`, a nie `.env.example`.

**2. Instalujemy czytnik i tworzymy `config.py` obok `.env`.** W terminalu PyCharma, w srodowisku projektu:

```powershell
python -m pip install python-dotenv
```

`python -m pip` uruchamia `pip` w interpreterze wskazywanym przez polecenie `python`. `install` dodaje pakiet do tego srodowiska. To polecenie terminala, a nie linia kodu w `config.py`. Pakiet do instalacji nazywa sie `python-dotenv`, a w kodzie importujemy modul `dotenv`. [DataCEE, Project 02: Define Configuration](https://www.datacee.com/#step-4-define-configuration) pokazuje `load_dotenv()` i odczyt ze zmiennych srodowiskowych. Tutaj korzystamy z [`dotenv_values()`](https://bbc2.github.io/python-dotenv/#load-configuration-without-altering-the-environment), aby odczytac wartosci prosto do slownika i nie zmieniac srodowiska procesu.

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
- `settings.get("INPUT_PATH")` daje np. tekst `"data/raw/orders.json"`. Brak klucza daje `None`, a `INPUT_PATH=` daje pusty tekst. Odrzucamy oba przypadki przed odczytem zamowien. To wazne, bo dolaczenie pustego tekstu do `base_dir` daloby katalog projektu zamiast sciezki do pliku.
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
Plik: orders.json
Odczytane rekordy: 7
```

Wynik zostal taki sam: 7 rekordow. `data/out` jeszcze nie powstaje, bo na tym etapie tylko odczytujemy dane. Zostaw `INPUT_PATH` i `OUTPUT_DIR` bez zmian do konca lekcji.

### Krok 2. Wymagamy obiektu z lista orders

Pierwszy reader moglby zwrocic string z pola orders. Len takiego stringa policzyloby znaki i main pokazaloby mylacy licznik zamiast zamowien.

Dlatego teraz kontrolujemy glowna strukture dokumentu.

Plik `ingestion.py` - krok 2:

```python
import json
from pathlib import Path


def read_orders(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8") as file:
        payload = json.load(file)

    if not isinstance(payload, dict):
        raise ValueError("JSON root must be an object")
    if "orders" not in payload or not isinstance(payload["orders"], list):
        raise ValueError("orders must be a list")

    return payload["orders"]
```

#### Co sie zmienilo?

Najpierw isinstance(payload, dict) sprawdza typ glownej wartosci JSON. Sam standard JSON nie wymaga, zeby caly dokument byl obiektem; to my ustalamy taka umowe dla eksportu.

Dopiero dla slownika sprawdzamy obecnosc orders i typ jego wartosci. Operator or nie odczyta payload["orders"], jezeli klucza nie ma, bo pierwsza czesc juz wykryla blad.

Orders: [] jest poprawna pusta lista. Orders: "brak danych" nie jest pusta dostawa. To inny typ, ktorego nie wolno przekazac do transformacji zamowien.

Raise konczy odczyt z opisem problemu. Nie zastapi danych wymyslona lista.

#### Sprawdzenie tej kontroli

Po `python main.py` nadal widzisz 7 rekordow: glowna wartosc dokumentu jest slownikiem, a `orders` jest lista.

### Krok 3. Kazdy element orders musi byc obiektem

Lista moze zawierac null, liczby i stringi. Sam fakt, ze orders jest lista, nie daje jeszcze listy slownikow zamowien.

Plik `ingestion.py` - krok 3:

```python
import json
from pathlib import Path


def read_orders(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8") as file:
        payload = json.load(file)

    if not isinstance(payload, dict):
        raise ValueError("JSON root must be an object")
    if "orders" not in payload or not isinstance(payload["orders"], list):
        raise ValueError("orders must be a list")

    for record_number, row in enumerate(payload["orders"], start=1):
        if not isinstance(row, dict):
            raise ValueError(f"Order at record {record_number} must be an object")

    return payload["orders"]
```

#### Co dodaje petla?

Enumerate przechodzi po elementach orders i daje ich pozycje od 1. Nie numeruje fizycznych linii pliku, bo formatowanie JSON nie wyznacza rekordow.

Isinstance(row, dict) sprawdza ksztalt pojedynczego elementu. Dla null dostalibysmy None, a nie slownik.

Jeszcze nie sprawdzamy, czy customer istnieje ani czy amount jest poprawna cena. To reguly jednego rekordu na pozniejszym etapie. Tutaj pilnujemy, zeby nastepna funkcja w ogole dostala rekordy.

#### Sprawdzenie kroku

Po `python main.py` nadal widzisz 7 rekordow. Kazdy element `orders` jest slownikiem.

### Krok 4. Sprawdzamy walute calego eksportu

`currency` dotyczy calego dokumentu. Nasz pipeline przyjmuje kwoty w PLN, dlatego sprawdzamy walute przed przetwarzaniem rekordow.

Plik `ingestion.py` - krok 4:

```python
import json
from pathlib import Path


def read_orders(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8") as file:
        payload = json.load(file)

    if not isinstance(payload, dict):
        raise ValueError("JSON root must be an object")
    if payload.get("currency") != "PLN":
        raise ValueError("Expected currency PLN")
    if "orders" not in payload or not isinstance(payload["orders"], list):
        raise ValueError("orders must be a list")

    for record_number, row in enumerate(payload["orders"], start=1):
        if not isinstance(row, dict):
            raise ValueError(f"Order at record {record_number} must be an object")

    return payload["orders"]
```

#### Co oznacza ten nowy if?

Payload.get("currency") pobiera walute. Brak pola daje None, ktore rowniez nie spelnia wymagania PLN.

Nie przeliczamy automatycznie EUR i nie podpisujemy takich kwot etykieta PLN. Na tej lekcji nie budujemy przelicznika walut; wymagamy uzgodnionej dostawy.

Kontrola jest za sprawdzeniem typu payload, zeby nie wywolywac get na liscie lub liczbie.

Source nadal jest opisem zrodla. Przy diagnozie wskazujemy nazwe pliku. Nie kopiujemy metadanych do listy zamowien przez przypadek.

#### Sprawdzenie kroku

Po `python main.py` nadal widzisz 7 rekordow. `orders.json` deklaruje `PLN`, wiec przechodzi kontrole.

### Krok 5. Ograniczamy wielkosc przyjmowanego dokumentu

Json.load nie jest automatycznym przetwarzaniem kolejnych zamowien bez przechowywania dokumentu. Parser tworzy struktury w pamieci. Potrzebujemy jawnej decyzji o rozmiarze tego wejscia.

Plik `ingestion.py` - krok 5, wersja docelowa:

```python
import json
from pathlib import Path


def read_orders(path: Path, max_bytes: int = 5_000_000) -> list[dict]:
    if path.stat().st_size > max_bytes:
        raise ValueError("Input file exceeds the agreed size limit")

    with path.open("r", encoding="utf-8-sig") as file:
        payload = json.load(file)

    if not isinstance(payload, dict):
        raise ValueError("JSON root must be an object")
    if payload.get("currency") != "PLN":
        raise ValueError("Expected currency PLN")
    if "orders" not in payload or not isinstance(payload["orders"], list):
        raise ValueError("orders must be a list")

    for record_number, row in enumerate(payload["orders"], start=1):
        if not isinstance(row, dict):
            raise ValueError(f"Order at record {record_number} must be an object")

    return payload["orders"]
```

#### Co dodalismy?

Opcjonalny max_bytes okresla dopuszczalny rozmiar w bajtach. Wczesniejsze read_orders(input_path) nadal dziala dzieki wartosci domyslnej 5_000_000.

Path.stat().st_size sprawdza rozmiar przed open. Przekroczenie limitu daje ValueError, a brak pliku moze juz tu zglosic FileNotFoundError.

To limit rozmiaru pliku, nie gwarancja zuzycia pamieci. `utf-8-sig` odczytuje UTF-8 i pomija opcjonalny znacznik BOM na poczatku.

#### Sprawdzenie limitu na tym samym pliku

W `pipeline.py` zmien na probe wywolanie na `raw_orders = read_orders(input_path, max_bytes=1)`. Po `python main.py` oczekuj `ValueError: Input file exceeds the agreed size limit`. Przywroc `raw_orders = read_orders(input_path)` i uruchom ponownie: wynik wraca do 7.

## 6. Punkt kontrolny: dokument gotowy do walidacji rekordow

Zostawiamy w ingestion.py **wersje z kroku 5**. Main.py pozostaje bez zmian. Pipeline.py na tym etapie tylko odczytuje dokument i zbiera licznik, a reporting.py prezentuje wynik.

Po python main.py otrzymujemy:

```text
Plik: orders.json
Odczytane rekordy: 7
```

Teraz wiemy, ze przyjelismy uzgodniony dokument PLN z lista slownikow. Nie wiemy jeszcze, czy kazda kwota, flaga, lista tagow i dane klienta sa poprawne. To odpowiedzialnosc nastepnego etapu.

Wszystkie dalsze czesci korzystaja z jednej funkcji read_orders. Kontrole dokumentu nie musza byc przepisywane razem z obliczeniami.

## 7. Jak ma wygladac poprawny rekord?

Zagniezdzenia wejscia mapujemy na jeden poprawny rekord zamowienia.

| Pole wyniku | Zrodlo i regula |
|---|---|
| order_id | dodatni int z order_id, nie bool |
| customer_id | dodatni int z customer.customer_id, nie bool |
| customer_name | niepusty string z customer.name, bez skrajnych spacji |
| status | wymagany tekst, po oczyszczeniu paid/pending/cancelled |
| amount | wymagany tekst, po konwersji skonczony nieujemny Decimal |
| country | customer.country; brak, null i pusty tekst daja UNKNOWN |
| is_gift | opcjonalny bool, przy braku False; jawne null nie jest bool |
| tags | opcjonalna lista stringow; brak lub null daja [], tagi normalizujemy |

Dla tagow przyjmujemy konkretny wybor biznesowy: usuwamy skrajne spacje, zmieniamy litery na male, pomijamy puste teksty, usuwamy duplikaty i zapisujemy alfabetycznie.

Nie oznacza to, ze JSON automatycznie usuwa duplikaty albo sortuje tablice. To nasza transformacja. Inna lista, np. kolejnych zdarzen klienta, moglaby wymagac zachowania kolejnosci.

Status, amount, customer, customer_id i name sa wymagane. Nie zamieniamy brakujacej kwoty na zero.

Identyfikatory wymagamy jako inty zgodnie ze schematem JSON tej dostawy. String "1001" nie zostanie tutaj automatycznie przyjety. To swiadoma roznica wobec tekstowych komorek z CSV.

### Transformacja: mapowanie i walidacja

**Miejsce pracy:** tworzymy transformations.py. Nie ma tu otwierania pliku, parsowania dokumentu ani wyboru katalogu.

Plik `transformations.py`:

```python
from decimal import Decimal, InvalidOperation


def normalize_order(raw: dict) -> dict:
    order_id = raw["order_id"]
    if type(order_id) is not int:
        raise TypeError("order_id must be int, not bool")
    if order_id <= 0:
        raise ValueError("order_id must be positive")

    customer = raw["customer"]
    if not isinstance(customer, dict):
        raise TypeError("customer must be an object")
    customer_id = customer["customer_id"]
    if type(customer_id) is not int:
        raise TypeError("customer_id must be int, not bool")
    if customer_id <= 0:
        raise ValueError("customer_id must be positive")
    if not isinstance(customer["name"], str):
        raise TypeError("customer name must be text")
    customer_name = customer["name"].strip()
    if customer_name == "":
        raise ValueError("customer name must not be empty")

    country = customer.get("country")
    if country is None:
        country = "UNKNOWN"
    elif not isinstance(country, str):
        raise TypeError("country must be text or None")
    else:
        country = country.strip().upper() or "UNKNOWN"

    if not isinstance(raw["status"], str):
        raise TypeError("status must be text")
    status = raw["status"].strip().lower()
    if status not in {"paid", "pending", "cancelled"}:
        raise ValueError("Unsupported status")

    if not isinstance(raw["amount"], str):
        raise TypeError("amount must be text")
    try:
        amount = Decimal(raw["amount"].strip().replace(",", "."))
    except InvalidOperation as exc:
        raise ValueError("Invalid amount") from exc
    if not amount.is_finite() or amount < 0:
        raise ValueError("Amount must be finite and non-negative")

    is_gift = raw.get("is_gift", False)
    if type(is_gift) is not bool:
        raise TypeError("is_gift must be bool")

    raw_tags = raw.get("tags")
    if raw_tags is None:
        raw_tags = []
    if not isinstance(raw_tags, list):
        raise TypeError("tags must be a list or None")
    tags = set()
    for tag in raw_tags:
        if not isinstance(tag, str):
            raise TypeError("Each tag must be text")
        normalized_tag = tag.strip().lower()
        if normalized_tag != "":
            tags.add(normalized_tag)

    return {
        "order_id": order_id,
        "customer_id": customer_id,
        "customer_name": customer_name,
        "status": status,
        "amount": amount,
        "country": country,
        "is_gift": is_gift,
        "tags": sorted(tags),
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

### Od zagniezdzen do pol rekordu

Raw["customer"] pobiera caly slownik klienta. Dopiero z niego odczytujemy customer_id, name i country. Najpierw sprawdzamy jego typ, zeby nie wywolywac operacji slownika na null albo stringu.

Customer_id nie staje sie drugim zamowieniem. Kopiujemy wybrane informacje do jednego poprawnego rekordu. Nie przechodzimy po tags, zeby powielic kwote zamowienia na kazdy tag.

Dokladny typ int odrzuca bool. W Pythonie True jest szczegolnie powiazane z int, ale w naszej domenie nie moze oznaczac numeru klienta ani zamowienia.

### Kwota i opcjonalny kraj

W amount wymagamy stringa, oczyszczamy go i tworzymy Decimal bez float. Sprawdzamy skonczonosc oraz nieujemnosc tak jak w poprzednim pipeline.

Country moze byc None. Nie wywolujemy na nim strip przed sprawdzeniem. Pusty tekst po oczyszczeniu tez daje UNKNOWN, ale liczba jako kraj jest bledem typu.

### Dlaczego nie bool(raw["is_gift"])?

Bool("false") daloby True, bo to niepusty string. Nie bylaby to poprawna interpretacja flagi od dostawcy.

Tutaj JSON-owe false zostalo juz sparsowane do False. Wymagamy prawdziwej wartosci bool. Brak pola ma swiadomie wartosc domyslna False; nie zgadujemy jej z dowolnego tekstu.

### Lista tagow i set

Jesli tags nie ma albo jest null, pracujemy na pustej liscie. String "promo" nie jest taka lista. Nie chcemy petli po jego literach i udawania, ze p/r/o/m/o sa tagami.

Kazdy element listy tez ma wymagany typ. Dobrej listy nie tworzy samo to, ze ma nawiasy kwadratowe. Liczba albo null jako jeden tag odrzuca rekord.

Roboczy set usuwa duplikaty. Sorted zamienia go na liste w ustalonej kolejnosci. Taki wynik mozna pozniej zapisac jako tablice JSON.

### Petla partii i odrzucenia

Transform_orders ma te sama odpowiedzialnosc co w poprzednim projekcie. Mozemy zachowac ten wzorzec: nowy set zaakceptowanych ID, osobna proba kazdego rekordu i odrzucenie z przyczyna.

Numerujemy pozycje w tablicy orders. To nie numery linii JSON, bo wciecia i formatowanie moga je dowolnie zmienic.

Nowe poprawne rekordy nie wspoldziela listy tags z oryginalnym rekordem. Nie zmieniamy dokumentu raw w pamieci ani na dysku.

### Sledzimy przejscie od dokumentu do poprawnego zamowienia

| Rekord | Co widzimy po odczycie | Co robi transformacja |
|---|---|---|
| 1001 (pozycja 1) | `customer` to slownik, kwota to `"120,50"`, a tagi to `[' promo ', 'PROMO', 'sql']` | wyciaga `customer_id=7` i `country='PL'`, tworzy `Decimal('120.50')`, normalizuje tagi do `['promo', 'sql']` i zwraca jeden slownik |
| 1003 (pozycja 3) | kraj i tagi maja `null`, flaga ma `true`, kwota to `"0"` | `None` w kraju daje `UNKNOWN`, `None` w tagach daje `[]`, flaga pozostaje `True`, a zero trafia do poprawnych zamowien |
| 1004 (pozycja 4) | dokument i obiekt zamowienia sa poprawnie zapisane, ale `amount` to `"bad"` | `normalize_order` zglasza `ValueError`; `transform_orders` zapisuje odrzucenie i przechodzi do nastepnego rekordu |

`clean, rejected = transform_orders(...)` rozpakowuje dwie zwrocone listy, w tej samej kolejnosci co `return clean, rejected`. `try` dotyczy jednej proby, a nie calej petli. `else` wykonuje sie tylko wtedy, gdy normalizacja i kontrola duplikatu nie zglosily wyjatku. Dlatego bledne wystapienie nie rezerwuje ID.

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

**Punkt kontrolny:** dla ID 1001 oczekujemy customer_id 7, country PL, is_gift False i tags ["promo", "sql"]. Dla ID 1003 kwota to Decimal("0"), kraj UNKNOWN, is_gift True, a tags []. Dane customer i tags nie tworza dodatkowych zamowien.

Rozszerzylismy proces bez dopisywania do main ani jednej reguly i bez przenoszenia tam printow. Main nadal pobiera ustawienia, uruchamia proces i zleca prezentacje.

## 9. Zapisujemy poprawne i odrzucone zamowienia

Transformacja zwraca dwie listy: `clean` i `rejected`. Kazda zapiszemy jako osobna tablice JSON.

Utworz `exporting.py` obok `pipeline.py`:

```python
import json
from pathlib import Path


def write_json(path: Path, rows: list[dict]) -> None:
    with path.open("w", encoding="utf-8") as file:
        json.dump(rows, file, ensure_ascii=False, indent=2, allow_nan=False)


def export_orders(output_dir: Path, clean: list[dict], rejected: list[dict]) -> None:
    output_dir.mkdir(parents=True, exist_ok=True)
    clean_rows = []
    for order in clean:
        row = order.copy()
        row["amount"] = str(order["amount"])
        clean_rows.append(row)

    write_json(output_dir / "clean_orders.json", clean_rows)
    write_json(output_dir / "rejected_orders.json", rejected)
```

`json.dump()` zapisuje liste do pliku, a `indent=2` dodaje wciecia. Standardowy zapis JSON w Pythonie nie obsluguje `Decimal` bez konwersji, dlatego w kopii rekordu zapisujemy kwote jako tekst, np. `"120.50"`. Oryginalny rekord zachowuje `Decimal`, a tagi nadal sa tablica. Tryb `"w"` nadpisuje wynik przy ponownym uruchomieniu.

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
| `clean_orders.json` | 5 rekordow: 1001, 1002, 1003, 1005 i 1006 |
| `rejected_orders.json` | 2 rekordy: 1004 (`Invalid amount`) i 1007 (`Amount must be finite and non-negative`) |

W rekordzie 1001 sprawdz `"amount": "120.50"` oraz `"tags": ["promo", "sql"]`. W `orders.json` dane zrodlowe pozostaja bez zmian.
