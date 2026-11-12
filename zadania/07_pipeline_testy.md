# 07. Zbuduj zestaw testow pipeline'u

Zespol zmienia reguly czyszczenia i nie chce recznie uruchamiac calego ETL po kazdej zmianie. Potrzebuje szybkiej informacji, czy kod nadal dziala tak, jak ustalono.

## Punkt wyjscia

Kontynuuj projekt z 06. Dodaj `tests/`. Wymyslone dane pojawiaja sie tylko wewnatrz testow, nie jako tryb pracy pipeline'u.

[Material do lekcji](../teoria/07_testy_pipeline_pytest.md).

## Co zbudowac

1. Zacznij od testu transformacji poprawnego zdarzenia. Sprawdz oczyszczony tytul, kategorie, ID i brak daty zamkniecia. Nastepnie sprawdz blad brakujacego tytulu i co najmniej dwa niepoprawne warianty kategorii. Wspolne dane przygotuj przez fixture.
2. Przetestuj partie zawierajaca poprawny rekord, niepoprawny rekord i powtorzone ID. Sprawdz dokladna liste zaakceptowanych ID i pozycje odrzucen.
3. Przetestuj zapis raw do katalogu tymczasowego. Po odczycie JSON powinien zawierac te same dane co wejscie. Test nie dotyka prawdziwego `data/raw`.
4. Przetestuj klienta HTTP: poprawna odpowiedz i blad HTTP. W obu testach zastap prawdziwe zapytanie. Dla bledu sprawdz, czy program nie probuje czytac odpowiedzi jak poprawnego JSON-a.
5. Dodaj test przebiegu, ktory uzywa prawdziwej transformacji i zapisu raw, ale podmienia siec i funkcje ladujaca. Sprawdz, ze raw zawiera wszystkie dane, a do zapisu trafiaja tylko poprawne.
6. Uruchom raport pokrycia. Wybierz jedna niepokryta regule danych i dopisz dla niej test. Opcjonalnie sprawdz UPSERT w osobnej bazie `nasa_events_test`.

## Jak sprawdzic wynik

- `python -m pytest -q tests` uruchamia podstawowe testy bez internetu i PostgreSQL. Nie zmienia tabeli szkoleniowej ani pliku raw.
- Pokaz jeden test, ktory wykrywa celowo zmienione oczekiwanie, i przywroc poprawna wersje. Powiedz, czego ten zestaw nie dowodzi o rzeczywistej bazie.
