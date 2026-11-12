# 10. Zbuduj pipeline CSV, ktory nie laduje wszystkich rekordow naraz

Dzial sprzedazy przysyla duzy plik. Wynik biznesowy ma zostac taki sam jak w 01, ale skrypt ma czytac po rekordzie i zapisywac male partie.

## Punkt wyjscia

Wroc do projektu `datashop_csv` z 01. Zacznij od [orders.csv](../materialy/01_csv/orders.csv) i zachowaj dotychczasowe reguly danych oraz dwa wyjscia CSV. [Material do lekcji](../teoria/10_duzy_csv_generatory_partie.md).

## Co zbudowac

1. Zastap liste z odczytu generatorem. Sprawdz naglowek i ksztalt wiersza, a nastepnie oddaj numer i rekord przez `yield`. Sprawdz `next()` na dwoch wierszach.
2. Przetwarzaj jeden rekord naraz. Zachowaj pozycje odrzucen. Duplikat rozpoznaj tez wtedy, gdy pierwsze wystapienie bylo w wczesniejszej partii.
3. Zapisuj poprawne i odrzucone rekordy partiami. Naglowki zapisz raz. Zwracaj liczniki, a nie listy wszystkich rekordow. Nie publikuj czesciowego wyniku, jesli odczyt calego pliku nie dobiegl konca.
4. Wygeneruj wiekszy plik na podstawie malego z lekcji. Powtarzaj dane z nowymi ID; nie dodawaj wygenerowanego pliku do Git.

## Jak sprawdzic wynik

- Maly plik z partia 3: 7 odczytanych, 5 poprawnych, 2 odrzucone i 3 partie. Pliki odpowiadaja wynikowi z 01.
- Dla 20 000 powtorzen i partii 1 000: 140 000 odczytanych, 100 000 poprawnych, 40 000 odrzuconych i 140 partii.
- Na kopii malego pliku dodaj duplikat w innej partii. Ma byc odrzucony. Przy blednej liczbie komorek w ostatnim wierszu sprawdz, czy poprzednie wyniki zostaly nienaruszone.
