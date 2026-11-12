# 05. Udostepnij wynik pipeline'u przez API

Aplikacja z mapa potrzebuje gotowych zdarzen, ale nie ma dostepu do SQL ani kluczy do infrastruktury. Ma wysylac zapytania HTTP do naszego serwera.

## Punkt wyjscia

Kontynuuj `nasa_pipeline` z 04. Tabela `eonet.events` ma zawierac rzeczywiste dane pobrane przez ETL. Dane do polaczenia API trzymaj w osobnym pliku `.env.api`.

[Material do lekcji](../teoria/05_fastapi_postgresql.md).

## Co zbudowac

1. Uruchom FastAPI i dodaj `GET /health`. Sprawdz w `/docs`, czy aplikacja odpowiada, zanim polaczysz ja z PostgreSQL.
2. Utworz osobne konto bazy tylko do odczytu tabeli. SQL trzymaj poza plikiem z endpointami. API ma czytac gotowe dane, nie odpytuje NASA i nie uruchamia ETL.
3. Dodaj `GET /events/{event_id}`. Dla znanego ID zwroc zdarzenie, a dla nieznanego `404`.
4. Dodaj `GET /events` z `limit` od 1 do 50 i `offset` od 0. Ustal kolejnosc po ID. Zwroc liste `items` oraz `next_offset` albo `null`, gdy nie ma kolejnej strony. Paginacja ma ograniczac wynik w SQL.
5. Napisz klienta w Pythonie, ktory zaczyna od pierwszej strony i uzywa `next_offset` do pobierania dalszych. Zakoncz po `null`.

## Jak sprawdzic wynik

- Pokaz `200` dla prawdziwego ID, `404` dla nieznanego i `422` dla `limit=0`. Strona po ostatnim wierszu zwraca `200` i pusta liste.
- Przy niezmienianej tabeli klient wypisuje wszystkie ID raz, w tej samej kolejnosci co `SELECT ... ORDER BY event_id`.
