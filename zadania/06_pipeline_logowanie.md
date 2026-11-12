# 06. Dodaj logowanie do dzialajacego ETL

Po nocnym uruchomieniu wiadomo tylko, ze nie ma swiezych danych. Zespol chce odczytac z logow, czy program w ogole ruszyl i na ktorym kroku sie zatrzymal.

## Punkt wyjscia

Kontynuuj ETL z 04. FastAPI nie jest potrzebne do tej proby. Skrypt nadal pobiera dane z NASA i zapisuje je do PostgreSQL.

[Material do lekcji](../teoria/06_logowanie_pipeline.md).

## Co zbudowac

1. Skonfiguruj `logging` raz przy uruchomieniu. Kazdy wpis ma pokazac czas, poziom, nazwe loggera i komunikat. Nie loguj hasel ani pelnych danych.
2. Dodaj logi startu i konca oraz informacje o etapach pobrania, raw, transformacji i zapisu. Liczniki maja odroznic pobrane, poprawne, odrzucone i zapisane dane. Odrzucenie rekordu oznacz jako ostrzezenie; szczegol ze sciezka raw pokaz w trybie `DEBUG`.
3. Gdy blad przerywa pipeline, zapisz traceback i zakoncz z kodem roznym od zera. Sukces zapisu oglaszaj dopiero po jego zakonczeniu.
4. Porownaj log z plikiem raw oraz zapytaniem kontrolnym w PostgreSQL. Sprawdz liczbe wierszy, unikalne ID i brak pustych tytulow oraz pustych list kategorii.

## Jak sprawdzic wynik

- Udany przebieg daje log sukcesu i kod `0`. W prostej probie ustaw zly port jako tekst w `.env`, sprawdz log bledu i kod `1`, a nastepnie przywroc poprawna konfiguracje.
- W `INFO` nie widac szczegolu ze sciezka raw. Po zmianie na `DEBUG` ten wpis sie pojawia. Zaden z logow nie pokazuje hasla.
