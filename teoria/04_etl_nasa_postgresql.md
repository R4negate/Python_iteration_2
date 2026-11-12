# 04. ETL: z API NASA do PostgreSQL

[Zadanie: zbuduj pipeline](../zadania/04_pipeline_etl_postgresql.md)

W lekcji 03 pobralismy dane z NASA i zapisalismy je w pliku. Teraz chcemy, zeby te zdarzenia dalo sie znalezc zwyklym `SELECT` w DBeaverze. Najpierw sprawdzimy je w Pythonie.

```text
NASA EONET -> pobranie i kopia raw -> transformacja w Pythonie -> PostgreSQL
                                          |
                                          +-> odrzucone rekordy do sprawdzenia
```

**ETL** to skrot od trzech krokow, ktore zrobimy po kolei:

1. **Extract**, czyli pobierz. Python pyta NASA o zdarzenia i zachowuje kopie odpowiedzi.
2. **Transform**, czyli przygotuj dane. Np. usun spacje z tytulu i zamien tekst z data na date w Pythonie. Jesli tytulu w ogole nie ma, odrzuc ten rekord.
3. **Load**, czyli zapisz wynik. Poprawne zdarzenia trafiaja do tabeli w PostgreSQL.

Pracuj w projekcie `nasa_pipeline` z lekcji 03. Zostaw `ingestion.py` bez zmian. Potrzebujemy Pythona 3.11 lub nowszego i uruchomionego PostgreSQL. Przez cala lekcje pobieramy dane bezposrednio z NASA.

## 1. Co ma oznaczac jeden wiersz?

Czy zdarzenie z dwiema kategoriami ma zajac dwa wiersze? U nas nie. Chcemy widziec **jeden wiersz na jedno zdarzenie**. To wlasnie nazywamy `grainem` tabeli: mowimy, co oznacza jeden wiersz. Rozpoznajemy zdarzenie po `event_id`. Gdy ma dwie kategorie, trzymamy obie w tym samym wierszu, w kolumnie `text[]` (lista tekstow).

| Pole z API | Pole w bazie | Regula |
|---|---|---|
| `id` | `event_id` | Wymagany niepusty tekst; klucz glowny. |
| `title` | `title` | Wymagany niepusty tekst; usuwamy spacje z brzegow. |
| `categories[].title` | `categories` | Co najmniej jedna poprawna nazwa; wynik to lista tekstow. |
| `closed` | `closed_at` | `null` oznacza otwarte zdarzenie; inaczej wymagana data z informacja o strefie lub przesunieciu. |

## 2. Transformacja jednego zdarzenia

Dodaj `transformations.py`:

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass
class Event:
    event_id: str
    title: str
    categories: list[str]
    closed_at: datetime | None


def required_text(value: object, field: str) -> str:
    if not isinstance(value, str) or not value.strip():
        raise ValueError(f"{field}: expected non-empty text")
    return value.strip()


def normalize_event(raw: object) -> Event:
    if not isinstance(raw, dict):
        raise ValueError("Expected an event object")

    event_id = required_text(raw.get("id"), "id")
    title = required_text(raw.get("title"), "title")

    raw_categories = raw.get("categories")
    if not isinstance(raw_categories, list) or not raw_categories:
        raise ValueError("categories: expected a non-empty list")

    categories = []
    for category in raw_categories:
        if not isinstance(category, dict):
            raise ValueError("category: expected an object")
        categories.append(required_text(category.get("title"), "category.title"))

    if "closed" not in raw:
        raise ValueError("Missing closed")
    closed = raw["closed"]
    closed_at = None
    if closed is not None:
        text = required_text(closed, "closed")
        try:
            closed_at = datetime.fromisoformat(text)
        except ValueError as error:
            raise ValueError("closed: expected an ISO date and time") from error
        if closed_at.utcoffset() is None:
            raise ValueError("closed: expected a time zone offset")

    return Event(event_id, title, categories, closed_at)
```

`Event` to klasa na gotowe, sprawdzone zdarzenie. Zobaczmy, co robi `normalize_event()`, zanim zwroci taki obiekt:

1. Sprawdza, czy dostala slownik. Potem `required_text()` pilnuje, zeby ID i tytul byly tekstem i nie byly puste po `strip()`. Sam zapis `title: str` tego nie sprawdza.
2. Z listy kategorii bierze same nazwy. Np. z `[{"id": "wildfires", "title": "Wildfires"}]` robi `["Wildfires"]`.
3. Sprawdza `closed`, czyli date zamkniecia. Brak pola to blad. Ale pole o wartosci `null` oznacza, ze zdarzenie nadal jest otwarte. W Pythonie zapisujemy wtedy `None`.
4. Jesli jest data, `datetime.fromisoformat()` zamienia tekst na obiekt daty i godziny. Np. `2024-08-09T00:00:00Z` oznacza 9 sierpnia 2024 o polnocy w UTC. `Z` oznacza UTC; kod odrzuca date, jesli nie wiadomo, do jakiej strefy lub przesuniecia sie odnosi.

Oryginalny slownik zostaje bez zmian. Funkcja tworzy nowy obiekt z danymi, ktore chcemy zapisac.

Sprawdz pojedyncze zdarzenie bez laczenia sie z baza. W terminalu projektu:

```powershell
@'
from ingestion import load_payload
from transformations import normalize_event

events = load_payload()["events"]
if events:
    print(normalize_event(events[0]))
else:
    print("NASA nie zwrocila zdarzen w tym zakresie dat.")
'@ | python -
```

`@' ... '@` w PowerShellu trzyma kilka linii tekstu, a `| python -` przekazuje je do Pythona. `load_payload()` pobiera dane z NASA, a `events[0]` bierze pierwsze zdarzenie, o ile lista nie jest pusta. Wynik to obiekt `Event` z ID, tytulem, lista kategorii i data zamkniecia albo `None`. Porownaj te pola z odpowiedzia NASA z lekcji 03.

## 3. Sprawdz wszystkie zdarzenia z listy

Dopisz do `transformations.py`:

```python
def transform_events(events: list) -> tuple[list[Event], list[dict]]:
    clean = []
    rejected = []
    seen_ids = set()

    for position, raw in enumerate(events, start=1):
        try:
            event = normalize_event(raw)
            if event.event_id in seen_ids:
                raise ValueError("Duplicate id in this batch")
        except ValueError as error:
            rejected.append({"position": position, "reason": str(error)})
            continue

        clean.append(event)
        seen_ids.add(event.event_id)

    return clean, rejected
```

Wczesniej sprawdzilismy jedno zdarzenie. Teraz robimy to samo w petli. **Partia** to wszystkie zdarzenia z jednego pobrania.

- `enumerate(..., start=1)` nadaje zdarzeniom numery 1, 2, 3 itd. To pozycje na liscie, a nie numery linii w pliku.
- Jesli zdarzenie jest poprawne, dopisujemy je do `clean`. Jesli `normalize_event()` zglosi `ValueError`, do `rejected` trafia numer i powod. `continue` przechodzi do nastepnego zdarzenia, wiec jeden blad nie zatrzymuje calej listy.
- `seen_ids` pamieta ID poprawnych zdarzen z tej partii. Gdy takie samo ID pojawi sie drugi raz, zostawiamy pierwsze poprawne zdarzenie.

Nie zakladamy z gory liczby odrzucen. Jesli wszystkie zdarzenia maja wymagane pola, lista `rejected` bedzie pusta. To sa sprawdzenia danych, a nie obsluga awarii internetu czy bazy.

## 4. Przygotuj tabele i sprawdz polaczenie

Utworzymy osobna baze tylko do tych cwiczen. W DBeaverze otworz edytor SQL dla istniejacej bazy `postgres`. Wlacz **Auto-commit**, czyli automatyczne zatwierdzanie polecen, i wykonaj osobno:

```sql
CREATE DATABASE nasa_events_lab;
```

Utworz polaczenie do `nasa_events_lab` i otworz edytor SQL dla **tej** bazy. Sprawdz `SELECT current_database();`, a nastepnie wykonaj:

```sql
CREATE SCHEMA IF NOT EXISTS eonet;

CREATE TABLE IF NOT EXISTS eonet.events (
    event_id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    categories TEXT[] NOT NULL CHECK (cardinality(categories) > 0),
    closed_at TIMESTAMPTZ
);
```

`eonet` to nazwa schematu, a `events` to tabela w tym schemacie. `PRIMARY KEY` nie pozwala zapisac dwoch wierszy z tym samym `event_id`. `CHECK` pilnuje, zeby lista kategorii nie byla pusta. `TIMESTAMPTZ` to typ dla daty i godziny. Ta sama chwila moze wyswietlic sie w DBeaverze w UTC albo w czasie lokalnym.

W terminalu projektu:

```powershell
python -m pip install "psycopg[binary]" python-dotenv
```

`psycopg` to biblioteka, przez ktora Python rozmawia z PostgreSQL. To odpowiednik posrednika: Python przekazuje mu SQL, a biblioteka wysyla polecenie do bazy i oddaje wynik. PostgreSQL musi juz dzialac; ta komenda go nie instaluje.

Utworz `.env` obok `main.py`. Wpisz dane swojego polaczenia z DBeavera. Wartosc hasla ponizej jest przykladem do zastapienia; jesli lokalnie nie jest wymagane, pozostaw `DB_PASSWORD=`.

```dotenv
DB_HOST=127.0.0.1
DB_PORT=5432
DB_NAME=nasa_events_lab
DB_USER=postgres
DB_PASSWORD="tu wpisz swoje haslo"
```

Sprawdz, czy `.gitignore` zawiera wpis `.env`. Nie commituj hasel. Konto `postgres` jest przykladem dla lokalnej bazy na zajeciach, nie kontem zalecanym do pracy aplikacji na produkcji.

Dodaj `config.py` obok `main.py`:

```python
from dataclasses import dataclass
from pathlib import Path

from dotenv import dotenv_values


@dataclass(frozen=True)
class DatabaseConfig:
    host: str
    port: int
    dbname: str
    user: str
    password: str


def database_config_from(settings: dict[str, str | None]) -> DatabaseConfig:
    return DatabaseConfig(
        host=settings.get("DB_HOST") or "127.0.0.1",
        port=int(settings.get("DB_PORT") or "5432"),
        dbname=settings.get("DB_NAME") or "nasa_events_lab",
        user=settings.get("DB_USER") or "postgres",
        password=settings.get("DB_PASSWORD") or "",
    )



def load_config() -> DatabaseConfig:
    settings = dotenv_values(Path(__file__).with_name(".env"), interpolate=False)
    return database_config_from(settings)
```

`config.py` czyta ustawienia z `.env` przy pomocy biblioteki `python-dotenv`. Np. `DB_PORT` jest w pliku tekstem, ale w Pythonie zamieniamy go na liczbe przez `int(...)`. `DatabaseConfig` trzyma dane polaczenia razem, zeby nie przekazywac hosta, portu, nazwy bazy, loginu i hasla osobno do kazdej funkcji.

Dodaj `db.py`:

```python
import psycopg

from config import DatabaseConfig


def open_connection(config: DatabaseConfig):
    return psycopg.connect(
        host=config.host,
        port=config.port,
        dbname=config.dbname,
        user=config.user,
        password=config.password,
        connect_timeout=5,
    )
```

`open_connection()` otwiera polaczenie na podstawie tych ustawien. Jesli baza jest niedostepna, `connect_timeout=5` ogranicza czekanie na otwarcie polaczenia.

Sprawdz polaczenie przed pisaniem kodu zapisu:

```powershell
@'
from config import load_config
from db import open_connection

config = load_config()
with open_connection(config) as conn:
    print(conn.execute("SELECT current_database(), current_user").fetchone())
'@ | python -
```

Powinienes zobaczyc nazwe bazy i konto zgodne z konfiguracja. `execute()` wysyla SQL, a `fetchone()` pobiera jeden wiersz. Jesli ten krok nie dziala, sprawdz serwer, port i logowanie przed uruchomieniem ETL.

## 5. Dodaj nowe zdarzenie albo zaktualizuj istniejace

Dodaj `loading.py`:

```python
from config import DatabaseConfig
from db import open_connection
from transformations import Event


UPSERT_EVENT = """
    INSERT INTO eonet.events (event_id, title, categories, closed_at)
    VALUES (%s, %s, %s, %s)
    ON CONFLICT (event_id) DO UPDATE SET
        title = EXCLUDED.title,
        categories = EXCLUDED.categories,
        closed_at = EXCLUDED.closed_at
"""


def load_events(events: list[Event], config: DatabaseConfig) -> None:
    if not events:
        return

    rows = [
        (event.event_id, event.title, event.categories, event.closed_at)
        for event in events
    ]
    with open_connection(config) as conn:
        with conn.cursor() as cursor:
            cursor.executemany(UPSERT_EVENT, rows)
```

Zalozmy, ze program pierwszy raz widzi ID zdarzenia z NASA. `INSERT` dodaje wiersz. Przy kolejnym pobraniu moze przyjsc to samo ID z nowym tytulem. Wtedy `ON CONFLICT (event_id) DO UPDATE` zmienia istniejacy wiersz zamiast tworzyc drugi. Taki zapis nazywamy **UPSERT**. `EXCLUDED.title` to nowy tytul, ktory przyszedl do zapisu. Jesli NASA nie zwroci jakiegos starego zdarzenia, samo to go z tabeli nie usuwa.

Jak Python przekazuje dane? Jeden `Event` zamieniamy na jedna krotke z czterema wartosciami. W SQL kazde `%s` to miejsce na kolejna wartosc z tej krotki. Przekazujemy SQL i wartosci osobno. Dzieki temu nawet tytul z apostrofem jest po prostu tekstem, a nie czescia polecenia SQL. Sterownik zamienia liste na `text[]`, a `None` na `NULL`.

`executemany()` robi ten sam UPSERT dla wszystkich zdarzen z `rows`. Chcemy, zeby baza zapisala cala partie albo zadnego z jej wierszy. To **transakcja**: jesli blok `with open_connection(...)` skonczy sie bez bledu, baza zatwierdzi zapis. Jesli SQL zglosi wyjatek w bloku, baza wycofa zmiany z calej partii, nawet jesli zdazyla zapisac pierwszy wiersz. To nie cofa zapisu pliku raw.

## 6. Polacz Extract, Transform i Load

Zastap `pipeline.py` z 03:

```python
import json
from pathlib import Path

from config import DatabaseConfig
from ingestion import load_payload
from loading import load_events
from transformations import transform_events


def run_pipeline(data_dir: Path, config: DatabaseConfig) -> tuple[int, list[dict]]:
    payload = load_payload()

    raw_path = data_dir / "raw" / "eonet_events.json"
    raw_path.parent.mkdir(parents=True, exist_ok=True)
    with raw_path.open("w", encoding="utf-8") as file:
        json.dump(payload, file, ensure_ascii=False, indent=2)

    clean, rejected = transform_events(payload["events"])
    load_events(clean, config)
    return len(clean), rejected
```

Zastap `main.py`:

```python
from pathlib import Path

from config import load_config
from pipeline import run_pipeline


def main() -> None:
    config = load_config()
    data_dir = Path(__file__).resolve().parent / "data"
    loaded_count, rejected = run_pipeline(data_dir, config)

    print(f"NASA EONET: zapisane: {loaded_count}; odrzucone: {len(rejected)}")
    for row in rejected:
        print(f"Pozycja {row['position']}: {row['reason']}")


if __name__ == "__main__":
    main()
```

`pipeline.py` wykonuje trzy kroki w kolejnosci: pobiera dokument, sprawdza zdarzenia i przekazuje poprawne do zapisu. `main.py` tylko uruchamia pipeline z ustawieniami i wypisuje podsumowanie.

Komunikat `zapisane: 3` znaczy, ze trzy zdarzenia udalo sie przekazac do zapisu. To moga byc trzy nowe wiersze, trzy aktualizacje albo kombinacja obu. Gdy zapis sie nie uda, program nie wypisze komunikatu o sukcesie.

## 7. Sprawdz wynik w DBeaverze

1. Uruchom `python main.py`. W terminalu zobaczysz liczbe zapisanych i odrzuconych zdarzen. Jesli cos zostalo odrzucone, skrypt wypisze powod.
2. W DBeaverze w bazie `nasa_events_lab` wykonaj ponizszy `SELECT`. Na poczatku tabela byla pusta, wiec po pierwszym uruchomieniu liczba wierszy powinna zgadzac sie z liczba zapisanych zdarzen.
3. Uruchom `python main.py` jeszcze raz i powtorz `SELECT`. Dla tych samych ID nie powstana dodatkowe wiersze. Jesli NASA zmieni zestaw zwracanych ID, w tabeli moga pojawic sie nowe zdarzenia.

```sql
SELECT event_id, title, categories, closed_at
FROM eonet.events
ORDER BY event_id;
```

### Samodzielnie

W DBeaverze napisz zapytanie, ktore pokaze tylko otwarte zdarzenia. Zwroc `event_id`, `title` i liczbe kategorii. Otwarte zdarzenie ma `closed_at IS NULL`. Wybierz jedno ID z wyniku: w lekcji 05 sprawdzisz, czy mozna je pobrac przez API.

W [lekcji 05](05_fastapi_postgresql.md) inna aplikacja odczyta te wiersze przez HTTP. ETL dalej uruchamiamy osobno.

## Dokumentacja

- [Psycopg: polaczenia, kursory i transakcje](https://www.psycopg.org/psycopg3/docs/basic/usage.html)
- [Psycopg: przekazywanie parametrow SQL](https://www.psycopg.org/psycopg3/docs/basic/params.html)
- [PostgreSQL: INSERT i ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html)
