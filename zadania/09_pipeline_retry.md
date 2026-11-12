# 09. Rozbuduj pipeline o odporne pobieranie z NASA

Zespol chce, zeby pojedynczy chwilowy blad NASA nie zatrzymywal od razu ETL. Program nie moze jednak powtarzac zlych zapytan w nieskonczonosc.

## Punkt wyjscia

Kontynuuj `nasa_pipeline` z 08. Pracuj w `ingestion.py` i `tests/`. Zostaw NASA jako jedyne zrodlo w normalnym przebiegu. [Material do lekcji](../teoria/09_odporne_pobieranie_api.md).

## Co zbudowac

1. Dodaj najwyzej trzy proby pobrania. Ponawiaj po timeoutach, wybranych bledach polaczenia, `429` i wybranych `5xx`. Nie ponawiaj `401`, `403`, `404` ani bledu danych po poprawnej odpowiedzi.
2. Dodaj rosnaca przerwe po kolejnych bledach. Obsluz `Retry-After` podany w sekundach i jako date. Gdy serwer kaze czekac dluzej niz dopuszczamy, zakoncz przebieg zamiast sprobowac za wczesnie.
3. Loguj numer proby, powod i czas oczekiwania. Po ostatnim bledzie przekaz go do `main.py`. Nie modyfikuj raw i nie wywoluj SQL po nieudanym pobraniu.
4. Dodaj testy, w ktorych kontrolujesz odpowiedz i sprawdzasz oczekiwanie bez prawdziwego `sleep()`.

## Jak sprawdzic wynik

- `503`, potem sukces: dwa wywolania i jedna przerwa.
- `429` z `Retry-After: 5`: przerwa trwa wedlug kodu piec sekund, ale test nie czeka.
- Trzy timeouty: trzy proby i tylko dwie przerwy. `401` albo brak `events`: jedna proba.
- Dotychczasowe testy pipeline'u nadal przechodza. Prawdziwy przebieg nadal pobiera NASA.
