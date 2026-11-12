# 01. Zbuduj pipeline zamowien z CSV

Sklep dostaje plik zamowien z systemu sprzedazy. Zespol finansowy chce osobny plik z danymi gotowymi do uzycia, a zespol wsparcia chce widziec, ktore rekordy poprawic. Jedno uruchomienie ma przygotowac oba pliki.

## Punkt wyjscia

Utworz projekt `datashop_csv`. Wez [orders.csv](../materialy/01_csv/orders.csv) i umiesc go w `data/raw/orders.csv`. Nie poprawiaj pliku wejsciowego recznie.

[Material do lekcji](../teoria/01_csv_pipeline.md).

## Co zbudowac

1. Zacznij od odczytu: pobierz `INPUT_PATH` i `OUTPUT_DIR` z `.env` i sprawdz, czy widzisz wszystkie rekordy. CSV ma separator `;`. Notatka z cudzyslowami i srednikiem ma pozostac jedna komorka.
2. Sprawdz naglowek i liczbe komorek w rekordach. Jesli struktura calego pliku jest niepoprawna, przerwij przebieg z czytelnym bledem. Pusta komorka w istniejacej kolumnie nie jest tym samym co brak komorki.
3. Oczysc ID, imie i nazwisko, status, kraj i notatke. ID ma byc dodatnie, nazwa klienta niepusta, a status ma nalezec do `paid`, `pending` albo `cancelled`. Brak kraju daj jako `UNKNOWN`. Kwote przelicz na `Decimal` bez przechodzenia przez `float`: akceptuj kropke lub przecinek, nie odrzucaj zera, odrzuc wartosci ujemne i nieskonczone. Dla powtorzonego ID zachowaj pierwsze poprawne wystapienie.
4. Zapisz w `data/out` dwa CSV: `clean_orders.csv` z polami zamowien oraz `rejected_orders.csv` z nazwa pliku, pozycja rekordu, surowym ID, typem i powodem bledu. Nie przerywaj calej pracy z powodu blednej kwoty w jednym rekordzie.
5. Polacz kroki w pipeline. `main.py` uruchamia proces i wypisuje liczniki; odczyt, transformacje i eksport maja byc w osobnych modulach.

## Jak sprawdzic wynik

- `python main.py` tworzy oba pliki: 5 poprawnych rekordow i 2 odrzucenia. Rekord `1003` z kwota zero i krajem `UNKNOWN` przechodzi.
- Sprawdz, czy `1001` ma kompletna notatke, a `1004` i `1007` maja zapisane powody odrzucenia.
- Po ponownym uruchomieniu wyniki nie sa dopisywane do plikow drugi raz.
