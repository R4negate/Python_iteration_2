# 04. Rozbuduj pobieranie do pelnego ETL

Zespol nie chce recznie przegladac JSON-a. Potrzebuje tabeli ze zdarzeniami i mozliwosci uruchomienia importu ponownie bez duplikatow.

## Punkt wyjscia

Kontynuuj `nasa_pipeline` z 03. Utworz szkoleniowa baze `nasa_events_lab` i przygotuj w DBeaverze tabele `eonet.events`. Polaczenie skonfiguruj przez `.env` bez dodawania hasla do repozytorium.

[Material do lekcji](../teoria/04_etl_nasa_postgresql.md).

## Co zbudowac

1. Zachowaj pobieranie z NASA i zapis pelnej odpowiedzi raw. Z jednego zdarzenia wyciagnij `event_id`, tytul, liste nazw kategorii i date zamkniecia. Najpierw sprawdz transformacje jednego elementu.
2. Przetworz cala liste. Zapis do bazy dostaje tylko poprawne rekordy. Nie zgaduj brakujacego tytulu ani kategorii. `closed: null` jest poprawne; jesli jest data, zamien ja na `datetime` z informacja o strefie. Drugie poprawne zdarzenie o tym samym ID w jednej partii odrzuc z powodem.
3. Zaprojektuj jeden wiersz na zdarzenie: `event_id` to klucz glowny, `title` to tekst, `categories` to lista tekstow, `closed_at` to data z czasem, ktora moze byc pusta. Polacz sie z baza z Pythona.
4. Zapisuj dane parametryzowanym UPSERT-em w jednej transakcji. Zmienione zdarzenie ma zaktualizowac wiersz, a nie utworzyc nowy. W terminalu pokaz liczbe zapisanych i odrzuconych rekordow oraz powody odrzucenia.
5. Zostaw `main.py` jako punkt uruchomienia. Odczyt, transformacje i zapis do bazy maja byc osobnymi krokami.

## Jak sprawdzic wynik

- `python main.py` robi pelny przeplyw NASA -> raw -> Python -> PostgreSQL. Pokaz w DBeaverze wybrane zdarzenie i znajdz to samo ID w raw.
- Uruchom drugi raz. Te same ID nie moga powielic sie w tabeli. Nie wymagamy odrzucen, gdy prawdziwa odpowiedz zawiera same poprawne zdarzenia.
