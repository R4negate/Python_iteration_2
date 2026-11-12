# 08. Przygotuj pipeline do uruchomienia przez inna osobe

Kolejny czlonek zespolu dostaje projekt. Ma uruchomic testy i caly ETL bez recznego zgadywania wersji bibliotek i bez dostepu do twojej lokalnej bazy.

## Punkt wyjscia

Wez `nasa_pipeline` po 07 z dzialajacymi testami. Glowny projekt bedzie uzywal `uv`. Poetry wyprobuj w osobnej kopii kodu, bez kopiowania `.venv` i plikow z haslami.

[Material do lekcji](../teoria/08_uv_poetry_ruff_docker.md).

## Co zbudowac

1. Zapisz zaleznosci w `pyproject.toml` i przypnij wersje Pythona. Oddziel biblioteki potrzebne do uruchomienia od narzedzi `dev`. Wygeneruj `uv.lock` i sprawdz, ze z niego da sie odtworzyc srodowisko.
2. Na osobnej kopii skonfiguruj Poetry, utworz `poetry.lock` i uruchom te same testy. Nie dodawaj obu plikow lock do tej samej kopii projektu.
3. Skonfiguruj Ruff. Sprawdz lint i format osobnymi poleceniami. Na probe dodaj jeden nieuzywany import, zobacz wynik kontroli i go usun. Po formatowaniu uruchom testy.
4. Przygotuj Dockerfile dla ETL i Compose z osobnym PostgreSQL. Zainicjuj tabele, zapewnij gotowosc bazy przed startem ETL i zachowaj dane poza kontenerem ETL. Nie kopiuj sekretow ani lokalnego `.venv` do obrazu.
5. Uruchom pipeline w Compose, sprawdz logi, plik raw i liczbe wierszy w bazie kontenerowej. Opisz w `README.md` komendy potrzebne nowej osobie do synchronizacji, testow i uruchomienia.

## Jak sprawdzic wynik

- Pokaz udane testy osobno w `uv` i Poetry oraz przechodzace kontrole Ruffa.
- Po uruchomieniu w Compose dane z NASA sa w kontenerowym PostgreSQL, a JSON zostaje w katalogu projektu. Powtorzenie przebiegu z tymi samymi ID nie dubluje wierszy. Zatrzymaj Compose bez kasowania wolumenu.
