# 03. Pobieranie danych z API NASA

W poprzedniej lekcji mielismy juz plik JSON. Tym razem Python sam poprosi NASA o dane. NASA prowadzi katalog zdarzen, takich jak pozary i burze. Nasz program pobierze odpowiedz z tego katalogu i zapisze ja w pliku.

```text
NASA EONET -> GET z Pythona -> data/raw/eonet_events.json
```

Plik nazywamy surowym (ang. `raw`), bo zostawiamy dane tak, jak je dostalismy. Na razie niczego nie poprawiamy i nie zapisujemy do bazy. Samo pobranie danych to krok `Extract`.

## 1. Wstep: API i HTTP

Wyobraz sobie, ze Python chce pobrac liste zdarzen. Wysyla prosbe do komputera NASA, a NASA odsyla odpowiedz. Python jest tu **klientem** (pyta), a NASA udostepnia **serwer** (odpowiada).

**API** mowi, o co mozna zapytac i jak to zrobic: np. pod jaki adres wyslac prosbe o liste zdarzen. **HTTP** to sposob, w jaki klient i serwer wymieniaja prosbe i odpowiedz. **JSON** to format, w ktorym moga byc zapisane same dane. To nie sa trzy nazwy tego samego.

Rozlozmy na czesci fikcyjne zapytanie: `GET https://api.example.com/events?limit=2`.

- `GET` to **metoda**: mowi, ze chcemy pobrac dane.
- `/events` to **sciezka**: mowi, ze chodzi o zdarzenia. `/events/DEMO_001` wskazywaloby jedno konkretne zdarzenie.
- `limit=2` to **parametr** po `?`: prosimy o najwyzej dwa wyniki.
- **Tresc** (ang. `body`) to same dane. Moze to byc lista zdarzen w odpowiedzi albo dane nowego zdarzenia wyslane przy dodawaniu. **Naglowek** to dodatkowa informacja o wiadomosci, np. `Content-Type: application/json` mowi, ze tresc zapisano jako JSON.

### Metody HTTP

| Metoda | Po ludzku | Przyklad |
|---|---|---|
| `GET` | Pobierz dane. Niczego nie zmieniaj. | `GET /events` pobiera liste zdarzen. |
| `POST` | Wyslij dane do serwera. Czesto uzywamy tego, by cos dodac. | `POST /events` moze dodac nowe zdarzenie. |
| `PUT` | Zastap caly zapis danego zdarzenia nowa wersja. | `PUT /events/DEMO_001` wysyla caly zapis od nowa. |
| `PATCH` | Zmien tylko to, co podajesz. | `PATCH /events/DEMO_001` moze zmienic sam tytul. |
| `DELETE` | Usun wskazane zdarzenie. | `DELETE /events/DEMO_001` usuwa to jedno zdarzenie. |

Przyklad: `DEMO_001` ma tytul "Pozar" i kategorie "Wildfires". Chcemy zmienic tytul na "Pozar lasu".

- Przy **PUT** wysylamy cala nowa wersje: `{"title": "Pozar lasu", "category": "Wildfires"}`. Podajemy tez kategorie, chociaz sie nie zmienia.
- Przy **PATCH** wysylamy tylko `{"title": "Pozar lasu"}`. Kategoria zostaje taka, jaka byla.

To przyklad API, w ktorym mozna edytowac zdarzenia. W tej lekcji **niczego nie zmieniamy na serwerze NASA**. Uzywamy tylko `GET`.

### Jak sprawdzic, czy sie udalo?

Serwer dolacza do odpowiedzi liczbe, czyli **kod HTTP**. Po niej mozemy szybko poznac wynik:

| Kod | Co to moze znaczyc? |
|---|---|
| `200` | Udalo sie. Jesli pytalismy o liste, moze byc pusta. |
| `201` / `204` | Utworzono nowy wpis / udalo sie i nie ma danych do odeslania. |
| `400` / `422` | Wyslalismy cos niepoprawnego, np. `limit=0`, gdy minimum to 1. |
| `401` / `403` | Trzeba sie zalogowac albo poprawic dane logowania / serwer nie daje nam dostepu. |
| `404` / `405` | Nie ma tego, o co pytamy / pod tym adresem nie wolno uzyc tej metody. |
| `429` | Wyslalismy za duzo zapytan w krotkim czasie. |
| `500` / `503` | Serwer ma blad albo jest chwilowo niedostepny. |

Na poczatek zapamietaj `200`, `404` i `500`. Jesli program przestaje czekac, bo serwer nie odpowiedzial na czas, to **timeout**, a nie kod HTTP. Z kolei `200` mowi, ze serwer odpowiedzial pomyslnie, ale Python nadal musi sprawdzic, czy otrzymal dane w formacie, ktorego potrzebuje.

### Paginacja, czyli pobieranie po kawalku

Zamiast pobierac cala liste naraz, mozemy prosic o po kilka rekordow. Zalozmy, ze lista ma piec zdarzen, a fikcyjne API obsluguje `limit` i `offset`:

| Zapytanie | Co dostaniemy? |
|---|---|
| `limit=2&offset=0` | Zdarzenia 1 i 2. Niczego nie pomijamy. |
| `limit=2&offset=2` | Zdarzenia 3 i 4. Pomijamy pierwsze dwa. |
| `limit=2&offset=4` | Tylko zdarzenie 5. Nie ma juz szostego. |

`limit` mowi, ile **najwyzej** zwrocic, a `offset`, ile pominac od poczatku listy. Nie kazde API dziala w ten sposob. W uzywanym przez nas adresie NASA mamy `limit`, ale dokumentacja nie podaje `offset`. `limit=5` daje najwyzej piec zdarzen, nie daje sposobu na pobranie nastepnej strony. Przecwiczymy to w lekcji 05.

## 2. Pierwsze zadanie do NASA

Otworz [przykladowe zapytanie EONET](https://eonet.gsfc.nasa.gov/api/v3/events?status=all&start=2024-08-01&end=2024-08-31&limit=5). Przegladarka zrobi `GET` i pokaze odpowiedz. Poszukaj klucza `events`: to lista. W kazdym jej elemencie mozesz zobaczyc np. `id` (identyfikator), `title` (tytul) i `categories` (kategorie).

Utworz projekt `nasa_pipeline`. W jego terminalu zainstaluj biblioteki:

```powershell
python -m pip install requests
```

Pierwsza wersja pliku `main.py`:

```python
import requests


response = requests.get(
    "https://eonet.gsfc.nasa.gov/api/v3/events",
    params={"status": "all", "start": "2024-08-01", "end": "2024-08-31", "limit": 5},
    timeout=10,
)
print(response.status_code)
response.raise_for_status()
payload = response.json()
print(f"Liczba zdarzen: {len(payload['events'])}")
```

Uruchom `python main.py`. Jesli pobranie sie uda, zobaczysz kod `200`, a w osobnej linii liczbe zdarzen. Np. `200` nie oznacza 200 zdarzen.

1. `requests.get(...)` wysyla prosbe do NASA. Slownik `params` trafia do adresu po `?`. Prosba dotyczy zdarzen otwartych i zamknietych (`status=all`), z wybranego zakresu dat, najwyzej pieciu.
2. `timeout=10` sprawia, ze program nie bedzie czekal bez konca na polaczenie lub kolejne dane. Nie mierzy lacznego czasu calego pobierania.
3. `response.raise_for_status()` przerywa prace z bledem, gdy serwer zwroci kod bledu (4xx albo 5xx). Dzieki temu nie probujemy czytac komunikatu o bledzie tak, jakby byl lista zdarzen.
4. `response.json()` zamienia tekst JSON na dane Pythona. `payload["events"]` wybiera liste zdarzen, a `len(...)` liczy jej elementy.

## 3. Przenies pobieranie do osobnej funkcji

Pierwszy `GET` juz dziala. Teraz przenosimy go do `ingestion.py`. W lekcji 04 skorzystamy z tej samej funkcji, zeby pobrac dane przed zapisem do bazy.

Plik `ingestion.py`:

```python
import requests


NASA_URL = "https://eonet.gsfc.nasa.gov/api/v3/events"
PARAMS = {"status": "all", "start": "2024-08-01", "end": "2024-08-31", "limit": 5}


def load_payload() -> dict:
    response = requests.get(NASA_URL, params=PARAMS, timeout=10)
    response.raise_for_status()
    payload = response.json()

    if not isinstance(payload, dict) or not isinstance(payload.get("events"), list):
        raise ValueError("Expected a JSON object with an events list")
    return payload
```

`load_payload()` wysyla `GET` i zwraca dane. Sprawdzamy, czy odpowiedz po odczytaniu JSON-a jest slownikiem z lista `events`. `{"events": []}` jest poprawne: NASA zwrocila liste bez zdarzen. Przy `{}` nie ma klucza `events`, wiec program zglasza blad. Na razie nie sprawdzamy zawartosci pojedynczych zdarzen.

## 4. Zapisz surowa odpowiedz

Plik `pipeline.py`:

```python
import json
from pathlib import Path

from ingestion import load_payload


def run_pipeline(data_dir: Path) -> tuple[Path, int]:
    payload = load_payload()

    output_path = data_dir / "raw" / "eonet_events.json"
    output_path.parent.mkdir(parents=True, exist_ok=True)
    with output_path.open("w", encoding="utf-8") as file:
        json.dump(payload, file, ensure_ascii=False, indent=2)

    return output_path, len(payload["events"])
```

W `run_pipeline()` najpierw pobieramy dane. Dopiero gdy je mamy, `mkdir(...)` tworzy brakujacy katalog, a `open("w", ...)` otwiera plik do zapisu. Gdy NASA nie odpowie, program nie dochodzi do `open` i poprzedni plik zostaje na dysku. Dlatego sprawdzamy komunikat z **ostatniego** uruchomienia, a nie tylko to, czy plik istnieje.

`json.dump(...)` zapisuje caly slownik. Plik moze wygladac troche inaczej, np. miec wiecej wciec, ale ma te same pola i wartosci. Funkcja zwraca sciezke pliku i liczbe zdarzen.

Zastap pierwsza wersje `main.py` ponizsza:

```python
from pathlib import Path

from pipeline import run_pipeline


def main() -> None:
    data_dir = Path(__file__).resolve().parent / "data"
    output_path, events_count = run_pipeline(data_dir)
    print(f"NASA EONET: zapisano zdarzen: {events_count}")
    print(output_path)


if __name__ == "__main__":
    main()
```

`main.py` ustala, gdzie jest katalog `data`, uruchamia `run_pipeline()` i wypisuje wynik. Pobieranie zostaje w `ingestion.py`, a zapis w `pipeline.py`.

### Sprawdz wynik

1. Uruchom `python main.py`. Zobaczysz liczbe pobranych zdarzen i sciezke zapisanego pliku.
2. Otworz `data/raw/eonet_events.json`. Porownaj liczbe elementow listy `events` z wynikiem w terminalu. Moze ich byc najwyzej piec.
3. Jesli program zglosi blad sieci albo odpowiedzi NASA, nie zapisze nowej wersji pliku. Po przywroceniu polaczenia uruchom go ponownie.

## Co dalej z tymi danymi?

- **[04. ETL do PostgreSQL](04_etl_nasa_postgresql.md):** ten sam pipeline pobiera dane z NASA, przeksztalca i waliduje rekordy w Pythonie, a poprawne dane zapisuje w PostgreSQL. Kopia raw pozwala sprawdzic, co dokladnie przyszlo z NASA.
- **[05. Udostepnianie danych przez API](05_fastapi_postgresql.md):** osobny serwis czyta gotowe dane z PostgreSQL i zwraca je klientowi przez HTTP. Zadanie klienta nie uruchamia ETL ani nie odpytuje NASA.

## Dokumentacja

- [NASA EONET: pola zdarzen i parametry endpointu](https://eonet.gsfc.nasa.gov/docs/v3)
- [Requests: parametry, JSON i bledy HTTP](https://requests.readthedocs.io/en/latest/user/quickstart/)
- [HTTP Semantics (RFC 9110)](https://www.rfc-editor.org/rfc/rfc9110.html) i [metoda PATCH (RFC 5789)](https://www.rfc-editor.org/rfc/rfc5789.html)
