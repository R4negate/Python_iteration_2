# 03. Zbuduj pipeline pobierania danych z NASA

Zespol monitorujacy zdarzenia naturalne chce zachowac odpowiedz NASA do pozniejszego sprawdzenia. Na tym etapie nie czyscimy danych i niczego nie zapisujemy do bazy.

## Punkt wyjscia

Utworz projekt `nasa_pipeline`. Uzyj endpointu EONET `https://eonet.gsfc.nasa.gov/api/v3/events` z lekcji. Zaczynamy od `status=all`, okresu `2024-08-01` do `2024-08-31` i `limit=5`. Nie przygotowuj zastepczego pliku wejsciowego.

[Material do lekcji](../teoria/03_api_nasa.md).

## Co zbudowac

1. Najpierw wykonaj `GET` i sprawdz kod odpowiedzi oraz liczbe zdarzen. Ustaw timeout. Parametry przekaz osobno od adresu.
2. Wydziel pobieranie do `ingestion.py`. Nie czytaj komunikatu HTTP o bledzie jak danych. Po wczytaniu JSON sprawdz, czy glowna wartosc ma liste `events`. Rozroznij pusta liste od brakujacego pola.
3. W `pipeline.py` zapisz **cala** odpowiedz jako `data/raw/eonet_events.json` dopiero po udanym odczycie. Nie zmieniaj zawartosci zdarzen.
4. W `main.py` uruchom pipeline i wypisz sciezke oraz liczbe pobranych zdarzen. Nie buduj jeszcze zapisu SQL ani serwera HTTP.

## Jak sprawdzic wynik

- Zgodny z odpowiedzia plik raw zawiera liste `events`, a liczba w terminalu zgadza sie z jej dlugoscia. `limit=5` nie obiecuje pieciu wierszy.
- Przy bledzie pobrania nie ma komunikatu sukcesu ani nadpisania poprzedniego pliku nowym pustym wynikiem.
