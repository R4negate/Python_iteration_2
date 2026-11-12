# 05. Wlasne API do odczytu z PostgreSQL

W lekcji 03 nasz Python pytal NASA o dane. Teraz odwrocimy role: to **nasz program odpowie** na prosbe innego programu. Np. aplikacja z mapa zapyta: "Jakie zdarzenia macie?". Nasze API odczyta wiersze z tabeli `eonet.events` i odesle je jako JSON.

```text
Zapis:   NASA -> ETL w Pythonie -> PostgreSQL
Odczyt:  PostgreSQL -> FastAPI -> klient HTTP
```

Zauwaz roznice: `python main.py` pobiera dane z NASA i **zapisuje** je w bazie. Nowy program z `api.py` tylko **czyta** to, co juz tam jest. Gdy klient zapyta o zdarzenie, nie musi czekac na NASA. Jesli nikt nie uruchomi ponownie `main.py`, API nadal pokazuje ostatni zapisany wynik.

Kontynuuj w `nasa_pipeline`. Otworz baze `nasa_events_lab` i sprawdz, czy tabela `eonet.events` ma wiersze z [lekcji 04](04_etl_nasa_postgresql.md). Jesli jeszcze nie uruchamiales ETL, zrob to przez `python main.py`. W tej lekcji `main.py` zostaje bez zmian.

## 1. Uruchom najprostszy endpoint

W terminalu projektu:

```powershell
python -m pip install fastapi uvicorn
```

Dodaj `api.py`:

```python
from fastapi import FastAPI


app = FastAPI(title="NASA EONET Events")


@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

`FastAPI()` tworzy aplikacje. Zapis `@app.get("/health")` to instrukcja dla aplikacji: "Gdy przyjdzie `GET /health`, uruchom funkcje `health()`". Funkcja oddaje slownik, a FastAPI wysyla go w odpowiedzi jako JSON. Taka dostepna operacje (metoda i sciezka) nazywamy **endpointem**.

`/health` sluzy tu tylko do pierwszej proby: jesli zwraca `{"status": "ok"}`, nasz serwer odpowiada. Jeszcze niczego nie sprawdza w bazie.

Uruchom serwer w terminalu projektu:

```powershell
python -m uvicorn api:app --host 127.0.0.1 --port 8000
```

FastAPI wie, **jak** odpowiedziec. Uvicorn uruchamia serwer, ktory czeka na zapytania i przekazuje je do FastAPI. `api:app` znaczy: wez obiekt `app` z pliku `api.py`. Adres `127.0.0.1` to ten komputer, a `8000` to numer portu, pod ktorym czeka serwer.

Otworz [Swagger UI](http://127.0.0.1:8000/docs). To strona, ktora pokazuje dostepne operacje i pozwala wyslac zapytanie bez pisania kodu. Rozwin `GET /health`, kliknij **Try it out**, a potem **Execute**. Szukaj statusu `200` i tresci `{"status":"ok"}`. Gdy port 8000 jest zajety, wybierz inny i podmien go tez w adresach. Zatrzymaj serwer przez `Ctrl+C` przed zmiana kodu.

## 2. Daj API dostep tylko do odczytu

Do tej pory Python logowal sie do bazy kontem, ktore moze zapisywac zdarzenia. API ma tylko pokazywac zdarzenia. Damy mu osobny login, ktory moze je odczytac, ale nie moze ich dodawac, zmieniac ani usuwac.

Otworz w DBeaverze edytor SQL dla `nasa_events_lab`. Wykonaj polecenia kontem, ktore ma prawo tworzyc uzytkownikow i nadawac dostep do tabeli, np. kontem administratora:

```sql
CREATE ROLE nasa_events_reader LOGIN PASSWORD 'tu_wpisz_unikalne_haslo';
GRANT CONNECT ON DATABASE nasa_events_lab TO nasa_events_reader;
GRANT USAGE ON SCHEMA eonet TO nasa_events_reader;
GRANT SELECT ON eonet.events TO nasa_events_reader;
```

Wstaw swoje haslo zamiast przykladowego i nie zapisuj go w repozytorium. `CREATE ROLE ... LOGIN` tworzy konto, ktorym API bedzie sie logowac. `GRANT` nadaje uprawnienia: polaczenie z baza (`CONNECT`), korzystanie ze schematu `eonet` (`USAGE`) i odczyt tabeli `events` (`SELECT`). Samo `USAGE` nie pozwala czytac wszystkich tabel w schemacie. Jezeli konto juz istnieje, nie tworz go drugi raz. Uzyj go i wykonaj trzy polecenia `GRANT` w bazie szkoleniowej.

Utworz osobny plik `.env.api` obok `.env`. `.env` pozostaje dla ETL; dopisz `.env.api` do `.gitignore`.

```dotenv
DB_HOST=127.0.0.1
DB_PORT=5432
DB_NAME=nasa_events_lab
DB_USER=nasa_events_reader
DB_PASSWORD="tu wpisz haslo roli odczytowej"
```

Dopisz do `config.py` z lekcji 04:

```python
def load_api_config() -> DatabaseConfig:
    settings = dotenv_values(Path(__file__).with_name(".env.api"), interpolate=False)
    if not settings.get("DB_USER"):
        raise ValueError("Set DB_USER in .env.api")
    return database_config_from(settings)
```

Teraz `main.py` nadal uzywa `.env`, a API bierze login i haslo z `.env.api`. Gdy zapomnisz podac `DB_USER` dla API, program pokaze blad zamiast uzyc mocniejszego konta `postgres`. Sprawdz, z jakim kontem program sie laczy:

```powershell
@'
from config import load_api_config
from db import open_connection

with open_connection(load_api_config()) as conn:
    print(conn.execute("""
        SELECT current_user,
               has_table_privilege('eonet.events', 'SELECT'),
               has_table_privilege('eonet.events', 'INSERT')
    """).fetchone())
'@ | python -
```

Oczekuj `('nasa_events_reader', True, False)`. Trzy wartosci odpowiadaja trzem czesciom `SELECT`: nazwa zalogowanego uzytkownika, czy wolno mu czytac tabele (`True`) i czy wolno mu wstawic wiersz (`False`). Nie musisz probowac niczego zapisywac.

## 3. Odczytaj dane w osobnym module

Dodaj `repository.py`. Tutaj trzymamy dwie funkcje: jedna pobiera kilka wierszy, druga szuka po ID. Ten plik zwraca wyniki z SQL. To `api.py` zdecyduje potem, co odeslac klientowi, np. `404`, gdy nie znaleziono ID.

```python
from psycopg.rows import dict_row

from config import DatabaseConfig
from db import open_connection


def select_events(config: DatabaseConfig, limit: int, offset: int) -> list[dict]:
    with open_connection(config) as conn:
        with conn.cursor(row_factory=dict_row) as cursor:
            cursor.execute(
                """
                SELECT event_id, title, categories, closed_at
                FROM eonet.events
                ORDER BY event_id
                LIMIT %s OFFSET %s
                """,
                (limit, offset),
            )
            return cursor.fetchall()


def select_event(config: DatabaseConfig, event_id: str) -> dict | None:
    with open_connection(config) as conn:
        with conn.cursor(row_factory=dict_row) as cursor:
            cursor.execute(
                """
                SELECT event_id, title, categories, closed_at
                FROM eonet.events
                WHERE event_id = %s
                """,
                (event_id,),
            )
            return cursor.fetchone()
```

- `dict_row` sprawia, ze zamiast krotki dostajemy slownik. Np. mozemy czytac `row["title"]` zamiast pamietac, na ktorej pozycji jest tytul.
- `select_events()` bierze strone z bazy. `ORDER BY event_id` uklada rekordy zawsze w tej samej kolejnosci, `OFFSET` pomija poczatek, a `LIMIT` ogranicza liczbe wierszy. `fetchall()` odbiera wszystkie wiersze z **tego wyniku**, a nie cala tabele.
- `select_event()` przekazuje szukane ID jako parametr SQL. W `(event_id,)` przecinek tworzy krotke z jedna wartoscia. `fetchone()` oddaje jeden wiersz albo `None`, jesli wiersza nie ma.

## 4. Zwroc klientowi liste albo jedno zdarzenie

Zastap `api.py` caloscia ponizej:

```python
from datetime import datetime
from typing import Annotated

from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel

from config import load_api_config
from repository import select_event, select_events


app = FastAPI(title="NASA EONET Events")
config = load_api_config()


class EventResponse(BaseModel):
    event_id: str
    title: str
    categories: list[str]
    closed_at: datetime | None


class EventPage(BaseModel):
    items: list[EventResponse]
    next_offset: int | None


@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}


@app.get("/events")
def list_events(
    limit: Annotated[int, Query(ge=1, le=50)] = 2,
    offset: Annotated[int, Query(ge=0)] = 0,
) -> EventPage:
    rows = select_events(config, limit + 1, offset)
    items = [EventResponse(**row) for row in rows[:limit]]
    next_offset = offset + limit if len(rows) > limit else None
    return EventPage(items=items, next_offset=next_offset)


@app.get("/events/{event_id}")
def get_event(event_id: str) -> EventResponse:
    row = select_event(config, event_id)
    if row is None:
        raise HTTPException(status_code=404, detail="Event not found")
    return EventResponse(**row)
```

Zacznijmy od tego, **co dostaje klient**. `EventResponse` opisuje jedno zdarzenie: ID, tytul, kategorie i date zamkniecia. `EventPage` opisuje cala odpowiedz z listy: `items` to zdarzenia na tej stronie, a `next_offset` podpowiada, od ktorego miejsca pobrac nastepna. To klasy z biblioteki Pydantic, ktorych FastAPI uzywa do sprawdzania i przygotowania odpowiedzi. Nie tworza tabeli. `EventResponse(**row)` bierze wartosci ze slownika po nazwach, np. przekazuje `title=row["title"]`. W JSON data pojawia sie jako tekst, a Pythonowe `None` jako `null`.

Mamy dwa adresy:

- `GET /events/{event_id}`: zamiast `{event_id}` wpisz prawdziwe ID z tabeli. `get_event()` szuka tego jednego zdarzenia. Gdy funkcja SQL zwroci `None`, serwer wysyla `404`.
- `GET /events?limit=2&offset=0`: liczby po `?` mowia, ile zdarzen pokazac i ile pominac. `Query(ge=1, le=50)` znaczy "co najmniej 1, najwyzej 50", a `Query(ge=0)` znaczy "co najmniej 0". `Annotated` pozwala zapisac te reguly obok typu `int`. Np. `limit=0` zostanie odrzucone, zanim wywola sie `list_events()`.

**Skad wiadomo, czy jest kolejna strona?** Klient prosi o dwa zdarzenia, a my pytamy baze o maksymalnie trzy (`limit + 1`). Jezeli dostaniemy trzy, pokazemy pierwsze dwa (`rows[:limit]`). Trzeci wiersz potwierdza, ze jest jeszcze cos do pobrania, wiec odsylamy `next_offset=2`. W nastepnym zapytaniu pominiemy pierwsze dwa wiersze. Gdy dostaniemy juz tylko dwa lub mniej, `next_offset=null` oznacza, ze to ostatnia strona. Nie uruchamiaj ETL miedzy tymi probami, zeby lista nie zmienila sie w trakcie cwiczenia.

Na kazde zapytanie o zdarzenia otwieramy polaczenie z baza, odczytujemy wynik i zamykamy polaczenie. Jesli baza nie odpowiada, serwer zwroci blad. Nie zwracamy pustej listy ani `404`, bo bez bazy nie wiemy, czy zdarzenia sa.

## 5. Sprawdz API z perspektywy klienta

Uruchom ponownie Uvicorn ta sama komenda. W DBeaverze sprawdz `SELECT COUNT(*) FROM eonet.events;` i wybierz dowolne ID z tabeli. W [Swagger UI](http://127.0.0.1:8000/docs) wykonaj ponizsze proby. Zamiast `N` wpisz liczbe wierszy z `COUNT(*)`, a zamiast `ID_Z_TABELI` wpisz wybrane ID.

| Zadanie HTTP | Oczekiwany wynik |
|---|---|
| `GET /health` | `200`; serwer HTTP odpowiada. |
| `GET /events?limit=2&offset=0` | `200`; najwyzej dwa pierwsze zdarzenia. Jesli sa kolejne, `next_offset: 2`. |
| `GET /events?limit=2&offset=2` | `200`; najwyzej dwa kolejne zdarzenia po pominieciu pierwszych dwoch. |
| `GET /events?limit=2&offset=N` | `200`; `items: []`, bo prosisz o strone po ostatnim wierszu. |
| `GET /events?limit=0` | `422`; niepoprawny parametr. |
| `GET /events/ID_Z_TABELI` | `200`; jedno zdarzenie o podanym ID. |
| `GET /events/UNKNOWN` | `404`; brak zdarzenia o takim ID. |

Na Windows w drugim terminalu mozesz sprawdzic tez `curl.exe -i -X POST http://127.0.0.1:8000/events`. Wynik `405` oznacza, ze pod tym adresem nie udostepniamy metody POST.

## 6. Pobierz wszystkie strony z Pythona

Utworz `read_pages.py` obok `api.py`. Ten skrypt jest klientem: zna adres API, ale nie zna hasla do bazy. Uzywa `requests.get()`, tak jak w lekcji 03. Tym razem pyta nasz serwer, a nie NASA.

```python
import requests


def main() -> None:
    url = "http://127.0.0.1:8000/events"
    offset = 0

    while True:
        response = requests.get(url, params={"limit": 2, "offset": offset}, timeout=10)
        response.raise_for_status()
        page = response.json()

        for event in page["items"]:
            print(event["event_id"])

        next_offset = page["next_offset"]
        if next_offset is None:
            break
        offset = next_offset


if __name__ == "__main__":
    main()
```

Gdy serwer dziala, w drugim terminalu uruchom `python read_pages.py`. Skrypt wypisze ID w tej samej kolejnosci co `SELECT ... ORDER BY event_id` w DBeaverze.

1. Przy `offset=0` pobiera i wypisuje najwyzej dwa pierwsze ID.
2. Jesli serwer podal liczbe w `next_offset`, skrypt uzywa jej w kolejnym zapytaniu.
3. Gdy serwer zwroci `null`, Python odczyta je jako `None`. `break` konczy petle.

Na koniec zatrzymaj serwer przez `Ctrl+C`.

### Samodzielnie

Dodaj opcjonalny filtr `?category=Wildfires`. Filtruj w SQL przed `ORDER BY`, `LIMIT` i `OFFSET`. Zostaw dotychczasowe dzialanie bez parametru. Porownaj odpowiedz z danymi w tabeli: kazde zwrocone zdarzenie powinno miec `Wildfires` na liscie kategorii.

## Dokumentacja

- [FastAPI: pierwsze kroki](https://fastapi.tiangolo.com/tutorial/first-steps/) i [walidacja parametrow](https://fastapi.tiangolo.com/tutorial/query-params-str-validations/)
- [FastAPI: modele odpowiedzi](https://fastapi.tiangolo.com/tutorial/response-model/)
- [PostgreSQL: LIMIT i OFFSET](https://www.postgresql.org/docs/current/queries-limit.html)
