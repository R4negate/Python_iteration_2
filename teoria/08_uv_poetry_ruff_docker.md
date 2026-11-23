# 08. Srodowisko projektu od podstaw: uv, Poetry, Ruff i Docker

[Zadanie: zbuduj pipeline](../zadania/08_pipeline_narzedzia_docker.md)

Do tej pory `nasa_pipeline` dzialal na naszym komputerze. Teraz wyobraz sobie, ze inna osoba dostaje repozytorium. Nie wie, jakiej wersji Pythona uzyc, co zainstalowac ani jak uruchomic baze. W tej lekcji przygotujemy projekt tak, zeby mogla go odtworzyc bez zgadywania.

| Narzedzie | Jaki problem rozwiazuje? |
|---|---|
| `uv` albo Poetry | Ustala, jakie biblioteki i w jakich wersjach zainstalowac dla projektu. |
| Ruff | Wskazuje wybrane problemy w kodzie i porzadkuje jego wyglad. |
| Docker | Uruchamia nasz kod i osobna baze w kontenerach. |

**`uv` i Poetry to dwie alternatywy.** Nie uzywamy obu do zarzadzania ta sama kopia projektu. Glowne cwiczenie i Docker uzywaja `uv`, a Poetry sprawdzimy osobno.

Punktem wyjscia jest `nasa_pipeline` po [06](06_logowanie_pipeline.md) i [07](07_testy_pipeline_pytest.md): z plikiem `storage.py` oraz testami w `tests/`. Nie zmieniamy transformacji, SQL-a ani zrodla danych.


## 1. Po co nam kolejna grupa narzedzi?

Wysylasz drugiej osobie pliki projektu. Uruchamia `python main.py` i dostaje `ModuleNotFoundError: No module named 'requests'`. Instaluje requests, ale potem brakuje sterownika PostgreSQL. Po instalacji bibliotek okazuje sie, ze nie ma uruchomionej bazy. U Ciebie program dzialal, bo te rzeczy byly juz przygotowane.

**Kod z plikow .py to nie caly projekt potrzebny do uruchomienia.** Potrzebujemy tez odpowiedniego Pythona, bibliotek, konfiguracji i uslug, z ktorymi ten kod wspolpracuje. W tej lekcji zapisujemy sposob przygotowania tych rzeczy, zamiast liczyc, ze ktos odtworzy je z pamieci.

Nie piszemy nowej transformacji. Wykorzystujemy ten sam pipeline i testy z 07, zeby sprawdzic, czy po zmianie sposobu instalacji nadal zachowuje sie tak samo.

## 2. Interpreter, biblioteka, zaleznosc i srodowisko

### Interpreter wykonuje kod

Python to m.in. program, ktory czyta nasz kod i go wykonuje. Tym programem jest **interpreter**. Polecenie `python main.py` oznacza: uruchom plik main.py przy pomocy interpretera wybranego w tym terminalu.

Na komputerze moze byc kilka interpreterow, np. Python 3.12 i 3.14 oraz interpretery srodowisk wirtualnych. PyCharm ma wlasne ustawienie interpretera projektu. Terminal i przycisk Run w edytorze nie musza przypadkiem wskazywac tego samego miejsca.

### Biblioteka dostarcza kod, z ktorego korzystamy

`requests` pomaga wykonywac zapytania HTTP, `psycopg` rozmawia z PostgreSQL, a pytest uruchamia testy. Gdy nasz projekt ich potrzebuje, nazywamy je jego **zaleznosciami**.

Zaleznosc bezposrednia to taka, ktora dodajemy sami, np. requests. Requests potrzebuje tez innych bibliotek, np. urllib3. To dla naszego projektu zaleznosci **posrednie**. Nie musimy ich osobno wpisywac jako bezposrednich tylko dlatego, ze requests ich uzywa, ale narzedzie musi je rowniez zainstalowac.

`logging` i `pathlib` sa w bibliotece standardowej Pythona: nie dodajemy ich przez `uv add`. Nasze `transformations.py` tez nie jest pakietem do pobrania: to czesc naszego kodu.

### Srodowisko wirtualne oddziela biblioteki projektow

Jeden projekt moze potrzebowac innego zestawu wersji niz drugi. Srodowisko `.venv` pozwala trzymac instalacje osobno. Biblioteka dodana w jednym srodowisku nie pojawia sie automatycznie w drugim.

`.venv` nie jest jednak opisem projektu do wyslania innym. To lokalny wynik instalacji. Zapiszemy opis i konkretne wersje, a druga osoba utworzy z nich swoje srodowisko. Nie kopiujemy `.venv` miedzy Windows i macOS ani do kontenera Linux.

## 3. Za co odpowiada kazde narzedzie?

| Pytanie | Narzedzie | Czego ono nie potwierdza? |
| --- | --- | --- |
| Jakiego Pythona i jakie biblioteki przygotowac? | uv albo Poetry | Czy transformacja poprawnie liczy wynik. |
| Czy kod ma wybrane problemy i jednolity format? | Ruff | Czy udalo sie polaczyc z NASA albo zapisac dane. |
| Czy kod zachowuje sie tak, jak ustalilismy? | pytest z lekcji 07 | Czy caly rzeczywisty ETL dziala na innym serwerze, jesli testowalismy tylko jednostki. |
| Jak uruchomic ETL oraz baze w opisanych srodowiskach? | Docker i Compose | Czy reguly biznesowe sa poprawne i czy zewnetrzne API jest dostepne. |

**uv i Poetry wybieramy jako alternatywy, nie kolejne warstwy tego samego projektu.** Oba pomagaja w zaleznosciach i srodowiskach. Dwa narzedzia zmieniajace te sama konfiguracje i dwa rozne locki utrudnilyby odpowiedz na pytanie: ktory zestaw wersji jest obowiazujacy?

W glownym projekcie wybieramy uv. Poetry poznajemy na osobnej kopii, zeby rozumiec projekt zespolu, ktory juz go uzywa. Ruff i pytest pracuja obok wybranego menedzera. Docker dojdzie dopiero wtedy, gdy lokalne srodowisko przejdzie testy.

## 4. Co zbudujemy i co bedziemy sprawdzac?

```text
kod pipeline'u
    -> opis zaleznosci i lock
    -> lokalne srodowisko
    -> Ruff i testy
    -> obraz ETL
    -> kontener ETL + kontener PostgreSQL
    -> logi, plik raw i wynik SELECT
```

Kazdy krok ma osobne potwierdzenie. `uv sync` potwierdza instalacje. Zielony pytest potwierdza sprawdzane zachowania. `docker build` potwierdza zbudowanie obrazu. Dopiero uruchomienie ETL i sprawdzenie danych potwierdza dzialajacy przeplyw w tym srodowisku.

Przyklady uzywaja **ukladu plikow z lekcji 04, 06 i 07**, czyli modulow obok `main.py`. Jesli masz juz pakiet `nasa_pipeline/` z kodem wewnatrz, zostaw go. W Dockerfile zamiast kopiowania osobnych modulow skopiujesz `main.py` oraz caly katalog pakietu; pokazemy to przy budowaniu obrazu. Testy nadal uruchamiasz z katalogu glownego. Nie naprawiamy importow przez przypadkowe zmiany `sys.path`.

Polecenia wykonujemy w terminalu, nie w pliku Python. Bloki PowerShell mozna w wiekszosci wykonac tez na macOS/Linux; roznice dotyczace sciezek i utworzenia katalogu sa zaznaczone. **Budowanie obrazu i prawdziwy ETL wymagaja internetu**, a podstawowe testy po zainstalowaniu bibliotek nie korzystaja z NASA.

## 5. `pyproject.toml`, lock i `.venv`: co jest czym?

`requests` jest nam potrzebny do pobrania zdarzen z NASA. Ale sam napis "zainstaluj requests" nie mowi, ktora wersje dostaniemy ani jakie dodatkowe biblioteki zainstaluja sie razem z nim. Dlatego rozdzielamy opis wymagan, zapis wybranych wersji i lokalna instalacje:

- **`pyproject.toml`:** czego potrzebujemy, np. "Python 3.12 i biblioteka `requests`". Mozemy wpisac zakres dozwolonych wersji.
- **Plik lock (`uv.lock` albo `poetry.lock`):** jakie konkretne wersje narzedzie dobralo, takze dla zaleznosci posrednich. Zapisujemy go w Git obok `pyproject.toml`, zeby kolejna osoba uzyla tej samej decyzji.
- **`.venv`:** lokalne srodowisko, czyli miejsce na interpreter srodowiska i zainstalowane biblioteki. Kazdy odtwarza je u siebie; nie wrzucamy go do repozytorium.

Przyklad: w opisie mozemy dopuscic kilka wersji requests, ale lock wskaze wybrane konkretne wydanie oraz wymagane wersje jego zaleznosci. Druga osoba odtwarza instalacje z locka, zamiast niezaleznie dobierac nowsze wydania. Plik TOML czytamy i edytujemy; lock aktualizuje narzedzie; .venv tworzymy lokalnie.

`requests` i sterownik PostgreSQL sa potrzebne podczas pracy ETL. `pytest` i Ruff sa potrzebne do pracy nad kodem, ale nie do uruchomienia pipeline'u. Dlatego zapisujemy je w osobnej grupie `dev`.

## 6. Glowna sciezka: uporzadkuj projekt z `uv`

Zainstaluj `uv` zgodnie z [oficjalna instrukcja](https://docs.astral.sh/uv/getting-started/installation/). Jesli uzywasz Windows z WinGet, mozesz wykonac:

```powershell
winget install --id=astral-sh.uv -e
```

`--id` wskazuje konkretny pakiet, a `-e` wymaga dokladnego dopasowania. To instalacja polecenia uv na komputerze, nie dodanie requests do naszego projektu. Na macOS z Homebrew odpowiednikiem jest `brew install uv`; inne warianty sa w podlinkowanej instrukcji.

Po instalacji otworz nowy terminal i sprawdz `uv --version`. Nie powtarzaj instalacji, jesli uv juz dziala. Otworz katalog `nasa_pipeline`, w ktorym jest `main.py`.

Jezeli masz aktywne stare `.venv`, otworz terminal bez jego aktywacji. Najbezpieczniej pracowac na kopii kodu bez `.venv`, bo synchronizacja moze usunac biblioteki, ktorych nie wpisalismy do projektu. Nie kopiuj hasel do nowego repozytorium. Jezeli jest juz `pyproject.toml`, nie uruchamiaj `uv init` ponownie; sprawdz jego zawartosc.


### Krok A. Wybieramy Pythona i tworzymy opis projektu

Wykonaj po kolei:

```powershell
uv python install 3.12
uv init --bare --python 3.12
uv python pin 3.12
```

- `uv python install 3.12` przygotowuje interpreter tej wersji zarzadzany przez uv. Nie musisz usuwac innych Pythonow.
- `uv init` tworzy opis projektu. `--bare` ogranicza generowane pliki; nie chcemy nowego przykladowego main.py ani zastapienia naszego kodu. W istniejacym projekcie z pyproject.toml ten krok pomijamy.
- `uv python pin` zapisuje wybor do `.python-version`. To wskazowka dla uv, nie instalacja bibliotek.

Otworz utworzony `pyproject.toml`. Znajdz `requires-python` i w tym cwiczeniu ustaw:

```toml
requires-python = ">=3.12,<3.13"
```

To **jedna linia wewnatrz istniejacej sekcji `[project]`**, nie caly plik do wklejenia zamiast reszty. Zakres oznacza Python 3.12.x: od 3.12 wlacznie, ale bez 3.13 i nowszych. Pin wybiera lokalny interpreter, a requires-python opisuje dopuszczalny zakres projektu. Trzymamy sie 3.12, zeby cwiczenie, testy i obraz Dockera mialy zgodny punkt wyjscia.

Sprawdzenie: masz pyproject.toml i .python-version, a main.py nadal zawiera Twoj pipeline. Na razie nie zakladamy, ze sa juz zainstalowane jego biblioteki.

### Krok B. Dodajemy zaleznosci do uruchomienia

```powershell
uv add requests python-dotenv "psycopg[binary]" fastapi uvicorn pydantic
```

Ta komenda nie tylko instaluje. Dopisuje wymagania do pyproject.toml, ustala zgodne wersje, zapisuje uv.lock i synchronizuje .venv. FastAPI, uvicorn i pydantic zostawiamy, bo ten sam projekt zawiera API z lekcji 05. ETL sam nie uruchamia tego API.

Cudzyslow przy `"psycopg[binary]"` pomaga przekazac caly zapis jako jeden argument powloki. Czesc `[binary]` jest nazwa dodatku biblioteki: wybieramy wariant z gotowymi skladnikami binarnymi sterownika.

Sprawdzenie: otworz pyproject.toml i znajdz liste dependencies. Potem otworz uv.lock: jest dluzszy, bo uwzglednia takze zaleznosci posrednie. Nie przepisujemy locka recznie.

### Krok C. Dodajemy narzedzia do pracy nad kodem

```powershell
uv add --dev pytest pytest-cov pytest-mock ruff
```

`--dev` oznacza grupe narzedzi developerskich. W zwyklej lokalnej pracy sa instalowane, ale przy budowaniu obrazu samego ETL mozemy je pominac. Pytest nie musi byc w obrazie tylko dlatego, ze uzywalismy go podczas pisania kodu.

Sprawdzenie: w pyproject.toml pojawia sie grupa dev. Sa w niej narzedzia, a nie np. requests potrzebny przez ETL.

### Krok D. Sprawdzamy srodowisko i testy

```powershell
uv sync --locked
uv run --locked python --version
uv run --locked python -c "import sys; print(sys.executable)"
uv run --locked python -m pytest -q tests
```

`sync` doprowadza srodowisko do stanu opisanego przez projekt. Domyslnie moze tez usunac nadmiarowe pakiety: dlatego nie robimy tej proby na starej, przypadkowej instalacji pelnej recznie dodawanych bibliotek. `--locked` zabrania cichej aktualizacji locka. Gdy pyproject.toml i lock nie pasuja, dostajesz blad do wyjasnienia, nie nowy zestaw wersji bez decyzji.

`uv run` uruchamia wskazane polecenie w srodowisku projektu. Nie musisz recznie aktywowac .venv. `sys.executable` pokazuje konkretna sciezke interpretera; sama informacja „3.12” nie pozwalalaby odroznic dwoch roznych srodowisk.

Oczekiwany wynik to interpreter 3.12 z projektu i przechodzace podstawowe testy z 07. Test PostgreSQL pozostaje pominiety, jezeli nie wlaczylismy go jawnie. Instalacja narzedzi nie powinna uruchomic prawdziwego ETL.

### Co robi druga osoba po pobraniu repozytorium?

Nie wykonuje ponownie init i nie dodaje od nowa bibliotek. Ma juz zapisany opis i lock:

```powershell
uv sync --locked
uv run --locked python -m pytest -q tests
```

Zwykle nie potrzebuje tez ponownie wykonywac pin, bo .python-version jest w repozytorium. Moze byc potrzebny internet do pobrania interpretera i pakietow. W PyCharmie wskazuje .venv jako interpreter: na Windows to `.venv/Scripts/python.exe`, a na macOS/Linux `.venv/bin/python`.

### Jak dodac lub zaktualizowac biblioteke pozniej?

Nowa bezposrednia zaleznosc trafia do projektu przez `uv add nazwa`. Nie wystarcza instalacja przez pip w przypadkowym terminalu, bo repozytorium nie dowie sie wtedy o nowym wymaganiu. Zamierzona aktualizacje np. requests wykonujemy przez `uv lock --upgrade-package requests`, potem `uv sync --locked` i testy. Zmiany opisu oraz locka przegladamy razem w Git.

Lock nie „zamraza projektu na zawsze” ani nie gwarantuje bezpieczenstwa bibliotek. Pozwala kontrolowac, **kiedy** zmieniamy wersje. Aktualizacje i poprawki bezpieczenstwa nadal sa potrzebne.


Podstawowe testy z 07 powinny przejsc bez NASA i bazy. Uruchamiamy `python -m pytest` z katalogu projektu, zeby Python widzial moduly lezace obok `main.py`. Jesli dodales opcjonalne testy, niektore moga miec wynik `skipped`. Do repozytorium dodaj `pyproject.toml`, `uv.lock` i `.python-version`. W `.gitignore` uwzglednij `.venv/`, `.env`, `.env.api`, `.env.docker`, `data/raw/`, `logs/`, `__pycache__/`, `.pytest_cache/`, `.ruff_cache/`, `.coverage` i `htmlcov/`. Gitignore nie usuwa plikow, ktore juz trafily do historii Git. Hasla, ktore zostaly ujawnione, trzeba zmienic.

### Co trafia do repozytorium?

| Zapisujemy w Git | Odtwarzamy lokalnie albo ustawiamy osobno |
| --- | --- |
| Kod, tests/, pyproject.toml, uv.lock, .python-version | .venv i cache narzedzi |
| Dockerfile, compose.yaml, sql/init.sql, .dockerignore | Kontenery i wolumen bazy |
| README z nazwami zmiennych i instrukcja | Pliki .env z wartosciami hasel, surowe dane i logi |

## 7. Poetry: ten sam problem, inne polecenia

Poetry tez umie zapisac wymagania, dobrac wersje i uruchomic polecenie we wlasciwym srodowisku. Przecwiczymy to w osobnym katalogu `nasa_pipeline_poetry`, obok glownego projektu. Skopiuj tam pliki `.py`, ewentualny podfolder pakietu z kodem oraz katalog `tests/`, a jezeli masz `pytest.ini`, skopiuj rowniez ten plik. Nie kopiuj `.venv`, `uv.lock`, `pyproject.toml` ani plikow z haslami.

Zainstaluj Poetry wedlug [oficjalnej instrukcji](https://python-poetry.org/docs/#installation) i sprawdz `poetry --version`. Alternatywnie `uv tool install poetry` instaluje samo polecenie Poetry w osobnym srodowisku narzedzia; nie oznacza to, ze `uv` zaczyna zarzadzac projektem Poetry.

Utworz w kopii nowy `pyproject.toml`:

```toml
[project]
name = "nasa-pipeline-poetry"
version = "0.1.0"
requires-python = ">=3.12,<3.13"

[tool.poetry]
package-mode = false
```

`package-mode = false` mowi: instalujemy zaleznosci, ale nie probujemy budowac i instalowac naszego ETL jako paczki. Mamy zwykly projekt uruchamiany przez `main.py`. Otworz katalog kopii w terminalu bez aktywnego srodowiska z projektu `uv`.


### Krok A. Wskazujemy interpreter dla kopii

```powershell
poetry env use 3.12
poetry env info
```

Pierwsza komenda wybiera Pythona. Druga pokazuje informacje o srodowisku. Poetry moze trzymac je poza folderem projektu, wiec brak katalogu .venv w kopii nie oznacza automatycznie bledu.

Jesli Poetry nie znajdzie interpretera 3.12, podaj pelna sciezke do niego w poetry env use. Jesli zainstalowales go przez uv, polecenie `uv python find --managed-python 3.12` wyswietli sciezke. Uzywamy uv tylko do znalezienia interpretera, a nie do synchronizowania projektu Poetry.

### Krok B. Zapisujemy biblioteki i osobny lock

```powershell
poetry add requests python-dotenv "psycopg[binary]" fastapi uvicorn pydantic
poetry add --group dev pytest pytest-cov pytest-mock ruff
```

`poetry add` dodaje wymagania i dobiera wersje. `--group dev` oddziela narzedzia od bibliotek potrzebnych w czasie pracy. Powstaje `poetry.lock`, ktory zapisujemy razem z pyproject.toml **tej kopii**. Nie musi zawierac identycznych wersji jak uv.lock: niezaleznie tworzymy nowy projekt, a jego wymagania i wybrane narzedzie ustalaja rozwiazanie.

### Krok C. Instalujemy i uruchamiamy te same testy

```powershell
poetry install
poetry run python -c "import sys; print(sys.executable)"
poetry run python -m pytest -q tests
```

`poetry install` korzysta z istniejacego locka. Nie jest poleceniem do zamierzonej aktualizacji wszystkich bibliotek. `poetry run` uruchamia polecenie we wlasciwym srodowisku, tak jak uv run w glownym projekcie. Zmienilismy sposob przygotowania srodowiska, nie reguly transformacji, wiec te same testy powinny przejsc.

Na drugim komputerze z gotowym pyproject.toml i poetry.lock wykonujesz install i run. Nie tworzysz od nowa projektu ani nie instalujesz kazdej biblioteki osobno.


| Co chcesz zrobic? | Projekt z `uv` | Projekt z Poetry |
|---|---|---|
| Dodac biblioteke | `uv add requests` | `poetry add requests` |
| Dodac narzedzie do pracy nad kodem | `uv add --dev pytest` | `poetry add --group dev pytest` |
| Zainstalowac projekt z zapisanego locka | `uv sync --locked` | `poetry install`, gdy `poetry.lock` jest juz w repozytorium |
| Uruchomic testy | `uv run --locked python -m pytest` | `poetry run python -m pytest` |

Nie przenosimy `poetry.lock` do glownego projektu. Po cwiczeniu wroc do katalogu `nasa_pipeline` z `uv.lock`.

## 8. Ruff: znajdz problem, a potem sprawdz format

Ruff ma dwa rozne polecenia. **Linter** (`ruff check`) szuka m.in. importow, z ktorych kod nie korzysta. **Formatter** (`ruff format`) ustala m.in. spacje, wciecia i podzial dlugich linii. Zaden z nich nie sprawdzi, czy dane naprawde zapisaly sie do PostgreSQL. Do tego mamy testy i kontrole z 06-07.

Na koncu `pyproject.toml` glownego projektu dopisz (nie dubluj istniejacych sekcji):

```toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["E4", "E7", "E9", "F"]
```


### Jak czytac konfiguracje?

- `[tool.ruff]` to sekcja ustawien Ruffa w tym samym pliku, ktory opisuje zaleznosci. TOML grupuje ustawienia w sekcje oznaczone nawiasami kwadratowymi.
- `target-version = "py312"` ustala docelowy poziom skladni Pythona. Nie instaluje Pythona i nie przelacza interpretera PyCharma.
- `line-length = 88` jest docelowa szerokoscia linii dla formatowania. Nie kazda linie da sie podzielic, np. dlugiego adresu, i nie jest to gwarancja twardego limitu.
- `select` wybiera grupy kontroli. `F` obejmuje m.in. nieuzywane importy i niezdefiniowane nazwy; `E4`, `E7` i `E9` obejmuja wybrane problemy skladni i organizacji kodu. Nie wlaczamy od razu wszystkich mozliwych regul.

### Krok A. Tylko sprawdzamy, niczego nie zmieniamy

```powershell
uv run --locked ruff check .
uv run --locked ruff format --check .
```

Kropka oznacza biezacy katalog. Ruff szuka w nim plikow zgodnie ze swoimi regulami odkrywania i wykluczeniami. `ruff check` domyslnie raportuje problemy, ale nie poprawia plikow. `format --check` tez tylko sprawdza: kod moze miec poprawna logike, ale nie byc jeszcze sformatowany.

### Krok B. Wywolujemy jeden rozpoznawalny problem

W storage.py dodaj na chwile `import csv`, chociaz go nie uzywasz. Wykonaj:

```powershell
uv run --locked ruff check storage.py
```

Oczekujemy `F401` i wskazania nieuzywanego importu. To kod reguly, nie kod HTTP. Przeczytaj nazwe pliku, numer linii i opis, a potem usun dodany import. Nie dopisuj `# noqa` tylko po to, zeby ukryc ten problem.

Gdy masz pakiet z modulami, wskazujesz odpowiednio `nasa_pipeline/storage.py`. Nie zmieniasz przez to reguly.

### Krok C. Formatujemy i sprawdzamy wynik

```powershell
uv run --locked ruff format .
uv run --locked ruff check .
uv run --locked ruff format --check .
uv run --locked python -m pytest -q tests
```

Tym razem format bez --check **zapisuje zmiany w plikach**. Obejrzyj diff w Git przed zatwierdzeniem. Inne rozmieszczenie nawiasow czy podzial linii nie powinny zmienic wyniku pipeline'u, dlatego uruchamiamy znane testy.

`ruff check --fix` to osobne polecenie, ktore moze automatycznie poprawic czesc problemow lintowania. Na pierwszym przejsciu naprawiamy znaleziony import recznie. Nie wlaczamy `--unsafe-fixes` bez zrozumienia zmian. W kopii Poetry uruchamiasz Ruffa przez `poetry run ruff ...`.

Zielony Ruff i zielony pytest to rozne sygnaly. Kod moze byc idealnie sformatowany, a mimo to blednie odrzucac `closed=None`. Moze tez poprawnie czyscic dane, a nadal zawierac niepotrzebne importy.


## 9. Docker: obraz, kontener i dane

`.venv` rozwiazuje problem bibliotek Pythona. Nie uruchamia PostgreSQL i nie pakuje calego srodowiska z systemem, w ktorym startuje kod. W Dockerze budujemy **obraz** z Pythonem, bibliotekami i plikami projektu. **Kontener** jest uruchomieniem tego obrazu. **Docker Compose** pozwala opisac kilka kontenerow, ktore maja razem pracowac.

W tym cwiczeniu uruchomimy jednorazowy ETL i osobny kontener PostgreSQL. Nie potrzebujemy FastAPI ani dostepu do PostgreSQL z komputera. Nie zmieniamy lokalnej bazy z lekcji 04. Zainstaluj i uruchom Docker Desktop z kontenerami Linux. W terminalu sprawdz `docker version` i `docker compose version`.


### Obraz to nie to samo co kontener

Dockerfile jest **przepisem**, obraz jest **przygotowanym wynikiem budowania**, a kontener jest **uruchomieniem obrazu**. Z jednego obrazu mozemy uruchomic kilka kontenerow. Samo zbudowanie obrazu nie pobiera jeszcze zdarzen i nie zapisuje ich do bazy.

```text
Dockerfile + kod + lock -> docker build -> obraz nasa-etl
obraz nasa-etl          -> docker run   -> proces Python w kontenerze
```

Obraz ma swoj system plikow i zainstalowane biblioteki. Kontener ma dodatkowo zapisywalna warstwe na czas swojego zycia. Usuniecie kontenera usuwa te warstwe, dlatego wyniki wymagajace zachowania umieszczamy w podlaczonych katalogach lub wolumenach.

Kontener nie jest po prostu kopia calego komputera. Kontenery Linux korzystaja z jadra Linux; na Windows i macOS Docker Desktop zapewnia odpowiednie srodowisko do ich uruchomienia. Nie musimy tworzyc odrebnej pelnej maszyny wirtualnej dla kazdego procesu ETL.

### Co to jest rejestr i tag?

`python:3.12-slim` to nazwa obrazu wraz z tagiem. Tag wybiera wariant: tutaj Python 3.12 i mniejszy obraz bazowy. Obrazy pobieramy z rejestrow, np. Docker Hub lub ghcr.io. Korzystamy z oficjalnego obrazu Pythona, obrazu uv jego autorow i oficjalnego obrazu PostgreSQL.

W tym cwiczeniu nie uzywamy latest. Zostawiamy konkretny tag uv i glowna wersje PostgreSQL z dotychczasowego przykladu. Tag wersji nadal moze z czasem wskazywac odswiezony obraz, wiec sam lock Pythona nie oznacza identycznych wszystkich skladnikow systemu. W dokladniej kontrolowanym wdrozeniu obrazy mozna przypiac po digest; na teraz wystarcza rozumienie, skad pochodzi kazdy obraz.

### Dlaczego uruchomimy dwa kontenery?

ETL wykonuje zadanie i konczy prace. PostgreSQL ma dzialac dalej i przechowywac dane. Oddzielne kontenery pozwalaja ponownie uruchomic ETL bez ponownego budowania calej bazy. Compose opisuje je razem oraz laczy siecia.

Budowanie potrzebuje dzialajacego silnika Docker, nie tylko samego polecenia docker w terminalu. Jesli docker version pokazuje Client, ale blad przy Server, najpierw uruchom Docker Desktop i poczekaj na gotowosc. To nie blad kodu Pythona.

### Krok A. Zapisujemy przepis na obraz

Dodaj w glownym katalogu plik `Dockerfile`:

```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:0.11.29 /uv /bin/uv

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev

COPY main.py pipeline.py config.py ingestion.py loading.py db.py storage.py transformations.py ./
ENTRYPOINT ["/app/.venv/bin/python", "main.py"]
```

Przejdzmy instrukcja po instrukcji:

| Instrukcja | Co robi i dlaczego jest w tym miejscu? |
| --- | --- |
| `FROM python:3.12-slim` | Ustala obraz bazowy z Pythonem. Nie korzysta z Pythona z Twojej .venv. |
| `COPY --from=... /uv /bin/uv` | Kopiuje gotowy program uv z obrazu jego autorow do budowanego obrazu ETL. |
| `WORKDIR /app` | Ustawia katalog dla kolejnych polecen. To katalog w obrazie, nie na Twoim komputerze. |
| `COPY pyproject.toml uv.lock ./` | Kopiuje opis i lock przed kodem, zeby instalacja mogla korzystac z cache. |
| `RUN uv sync --locked --no-dev` | Podczas budowania instaluje zaleznosci uruchomieniowe w /app/.venv. Brak albo nieaktualny lock przerywa budowanie. |
| `COPY` z lista plikow .py | Dodaje nasz kod. Zmiana samego kodu nie musi wymuszac ponownego pobrania bibliotek. |
| `ENTRYPOINT [...]` | Ustala, co uruchomi sie przy starcie kontenera: interpreter z obrazu i main.py. Nie wykonuje ETL podczas budowania. |

Gdy uzywasz pakietu z modulami w podfolderze, **zastap tylko ostatnie COPY z lista plikow .py** powyzszego Dockerfile nastepujacymi dwiema instrukcjami:

```dockerfile
COPY main.py ./
COPY nasa_pipeline/ ./nasa_pipeline/
```

Pozostale instrukcje zostaja. Upewnij sie, ze main.py importuje pakiet, a config.py wyznacza katalog projektu jako rodzica katalogu pakietu. Wtedy .env ma byc w /app/.env, a dane w /app/data. Nie kopiuj plikow pakietu luzem do /app, bo zmieniloby to strukture importow.

To obraz szkoleniowy, nie komplet zabezpieczen produkcyjnych. Wdrozenie wymaga tez m.in. konta aplikacji z minimalnymi uprawnieniami, kontroli obrazow i sposobu przechowywania sekretow. Tu nie dodajemy hasel do obrazu.

### Krok B. Wykluczamy rzeczy, ktorych nie chcemy w obrazie

Dodaj `.dockerignore`:

```text
.env
.env.*
.venv/
.git/
data/
__pycache__/
.pytest_cache/
logs/
.ruff_cache/
.coverage
htmlcov/
```

Kropka w `docker build ... .` wskazuje **kontekst budowania**, czyli katalog, z ktorego Docker moze pobierac pliki przez COPY. .dockerignore ogranicza jego zawartosc. To inny plik niz .gitignore: wykluczenie czegos z Git nie wyklucza automatycznie z budowania i odwrotnie.

Nie chcemy wysylac hasel ani lokalnych danych do procesu budowania obrazu. Nie kopiujemy tez `.venv` z Windows: kontener tworzy wlasne srodowisko pod Linuxem.

### Krok C. Budujemy i sprawdzamy obraz, jeszcze bez ETL

Zbuduj obraz i sprawdz go bez laczenia z baza:

```powershell
docker build -t nasa-etl .
docker run --rm --entrypoint /app/.venv/bin/python nasa-etl --version
```

W pierwszej komendzie `-t nasa-etl` nadaje nazwe obrazowi, a koncowa kropka wskazuje biezacy katalog jako kontekst. Obraz musi zbudowac sie poprawnie przed wykonaniem drugiej komendy.

Drugie polecenie na chwile zastepuje domyslne uruchomienie i pokazuje wersje Pythona z obrazu. `--rm` usuwa ten jednorazowy kontener po wykonaniu polecenia, ale nie usuwa obrazu.

## 10. Uruchom ETL i osobna baze przez Compose

### Krok A. Przygotowujemy schemat i konfiguracje

Utworz `sql/init.sql` z **`CREATE SCHEMA` i `CREATE TABLE`** z [lekcji 04](04_etl_nasa_postgresql.md). Nie kopiuj `CREATE DATABASE`: kontener utworzy baze z konfiguracji.

Obok `Dockerfile` utworz `.env.docker`. Przykladowe haslo zastap swoim, uzywanym tylko w tym cwiczeniu. Tego pliku nie dodawaj do Git.

```dotenv
DB_HOST=db
DB_PORT=5432
DB_NAME=nasa_events_lab
DB_USER=nasa
DB_PASSWORD=change_me_locally
```

### Krok B. Opisujemy dwie uslugi i ich polaczenie

W Compose `db` jest nazwa uslugi z PostgreSQL. Z perspektywy kontenera ETL `127.0.0.1` oznaczaloby sam kontener ETL, nie baze. Zapisz `compose.yaml` obok `Dockerfile`:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_DB: nasa_events_lab
      POSTGRES_USER: nasa
      POSTGRES_PASSWORD: ${DB_PASSWORD:?Set DB_PASSWORD in .env.docker}
    healthcheck:
      test: ["CMD", "pg_isready", "-h", "127.0.0.1", "-U", "nasa", "-d", "nasa_events_lab"]
      interval: 2s
      timeout: 3s
      retries: 15
    volumes:
      - pgdata:/var/lib/postgresql
      - type: bind
        source: ./sql/init.sql
        target: /docker-entrypoint-initdb.d/init.sql
        read_only: true
        bind:
          create_host_path: false
  etl:
    build: .
    image: nasa-etl
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - type: bind
        source: ./.env.docker
        target: /app/.env
        read_only: true
        bind:
          create_host_path: false
      - ./data:/app/data

volumes:
  pgdata:
```

- Compose odczytuje `DB_PASSWORD` z `--env-file .env.docker` i przekazuje je do PostgreSQL. Ten sam plik podlaczamy tylko do odczytu (`read_only: true`) jako `/app/.env`, bo takiego pliku szuka nasz `config.py`.
- `healthcheck` sprawdza, czy PostgreSQL przyjmuje polaczenia. `init.sql` wykona sie przy pierwszym uruchomieniu z pustym wolumenem. Na istniejacej bazie zmiana tego pliku nie uruchomi go ponownie.
- `pgdata` zachowuje dane bazy poza kontenerem. `./data:/app/data` oznacza, ze surowy JSON pojawi sie tez w katalogu projektu. Nie publikujemy portu PostgreSQL na komputerze.

### Jak czytac YAML i podlaczenia?

Wciecia w YAML okreslaja, do ktorego elementu naleza ustawienia. `services` grupuje uslugi, `db` i `etl` sa ich nazwami. `image` wskazuje gotowy obraz, a `build: .` sposob zbudowania naszego. Na koncu `volumes: pgdata:` deklaruje nazwany wolumen, ktory zostanie utworzony przy pierwszym uzyciu.

Podlaczenie `type: bind` udostepnia istniejacy plik z komputera wewnatrz kontenera. `source` to plik projektu, `target` to miejsce w kontenerze, a `read_only: true` zabrania kontenerowi zmieniania go przez to podlaczenie. `create_host_path: false` nie pozwala tworzyc katalogu, gdy pomylilismy sciezke do wymaganego pliku. Przed startem **musza istniec pliki** sql/init.sql i .env.docker. W ten sposob brak pliku nie zamieni sie w zagadkowy blad „to jest katalog”.

Nazwany wolumen i bind mount rozwiazuja rozne potrzeby. Dane PostgreSQL oddajemy Dockerowi w pgdata. Plik raw chcemy obejrzec w projekcie, dlatego podlaczamy katalog data z komputera. Dla obrazu PostgreSQL **18** montujemy /var/lib/postgresql; nie zamieniamy tej sciezki bez sprawdzenia dokumentacji wersji obrazu.

`depends_on` z warunkiem `service_healthy` czeka na pomyslny healthcheck przed startem zaleznosci. Sam start procesu bazy nie znaczy jeszcze, ze baza przyjmuje polaczenia. To sprawdzenie nie gwarantuje, ze baza nigdy nie przestanie dzialac podczas ETL; taki blad nadal musi obsluzyc nasz program.

Ustawienia POSTGRES_* inicjalizuja nowa baze na pustym wolumenie. Zmiana hasla w pliku konfiguracyjnym nie zmieni automatycznie hasla uzytkownika juz istniejacej bazy. Konto tworzone przez POSTGRES_USER ma szerokie uprawnienia: nadaje sie do tego odizolowanego cwiczenia, nie jako gotowe konto produkcyjnego ETL.

### Dwa odczyty jednego pliku .env.docker

1. Compose czyta go przez `--env-file`, zeby zastapic `${DB_PASSWORD:?...}` w YAML. `:?` oznacza: brak lub pusta wartosc ma zatrzymac konfiguracje, zamiast uruchomic baze bez ustalonego hasla.
2. Python czyta ten plik podlaczony jako `/app/.env`, przez dotychczasowa funkcje w config.py.

`--env-file` samo w sobie nie oznacza, ze wszystkie zmienne automatycznie pojawia sie w kazdym kontenerze. W naszym przykladzie jawnie przekazujemy POSTGRES_PASSWORD do bazy, a ETL czyta podlaczony plik. Ustawiona juz w powloce zmienna DB_PASSWORD ma pierwszenstwo przy podstawianiu Compose, wiec nie ustawiaj innej wartosci tej samej zmiennej w terminalu tego cwiczenia. Nie pokazuj rozwinietej konfiguracji z haslem na ekranie ani w logach.


### Krok C. Sprawdzamy konfiguracje i uruchamiamy sama baze

W PowerShellu utworz katalog na dane, a potem sprawdz opis Compose:

```powershell
New-Item -ItemType Directory -Path data -Force | Out-Null
docker compose --env-file .env.docker config --quiet
docker compose --env-file .env.docker up -d --wait db
docker compose --env-file .env.docker ps
```

Na macOS/Linux pierwsze polecenie zastap `mkdir -p data`. `config --quiet` sprawdza skladnie i podstawienie zmiennych bez wypisywania rozwinietego YAML z haslem. Brak pliku lub pusty DB_PASSWORD ma byc bledem, ktory poprawiamy przed startem.

`up` tworzy i uruchamia usluge. `-d` zostawia ja dzialajaca w tle, `--wait` czeka na gotowosc, a koncowe `db` wybiera tylko baze. `ps` pokazuje stan uslug. Oczekujemy dzialajacej, zdrowej bazy. Nie pobralismy jeszcze zadnych zdarzen z NASA.

### Krok D. Jednorazowo uruchamiamy ETL

```powershell
docker compose --env-file .env.docker run --rm --build etl
```

`run` uruchamia jednorazowe zadanie dla uslugi etl. `--build` uwzglednia aktualny kod przy budowaniu obrazu. `--rm` usuwa kontener tego zadania po jego zakonczeniu, nie baze ani wolumen.

Tutaj po raz pierwszy w tej czesci nastapi prawdziwe pobieranie z NASA i zapis do bazy. Powinny pojawic sie znane logi. Jesli NASA jest niedostepna, to nie dowodzi bledu budowania obrazu: szukamy etapu awarii w logach. Nie dorabiamy cichego „sukcesu bez danych”.

### Krok E. Sprawdzamy faktyczny wynik

```powershell
docker compose --env-file .env.docker exec db psql -U nasa -d nasa_events_lab -c "SELECT COUNT(*) FROM eonet.events;"
```

`exec db` wykonuje polecenie w **juz dzialajacym** kontenerze bazy. `psql` jest klientem PostgreSQL, `-U` wskazuje uzytkownika, `-d` baze, a `-c` jedno polecenie SQL. Nie musimy instalowac psql na komputerze ani publikowac portu bazy.

Oprocz liczby wierszy obejrzyj plik data/raw/eonet_events.json na komputerze i porownaj ID z baza. Liczba wszystkich wierszy w tabeli nie musi byc rowna liczbie rekordow z ostatniego pobrania, jesli wczesniej zapisywalismy inne ID. W pliku raw mamy ostatnia odpowiedz, a w bazie wynik kolejnych UPSERT-ow.

### Krok F. Zatrzymujemy uslugi, zostawiamy dane

```powershell
docker compose --env-file .env.docker down
```


Po udanym przebiegu zobaczysz logi z 06, plik `data/raw/eonet_events.json` i liczbe wierszy z `psql`. `down` zatrzymuje projekt i usuwa kontenery oraz jego siec. W tej komendzie celowo nie ma `-v`, zeby nie usuwac danych z wolumenu.

### Samodzielnie

Przed `down` uruchom ETL drugi raz i ponow zapytanie `COUNT(*)`. Porownaj ID z raw przed i po uruchomieniu. Jesli NASA zwrocila te same ID, liczba wierszy nie powinna wzrosnac. W `README.md` projektu zapisz komendy do synchronizacji, lintowania, testow i uruchomienia Compose. Podaj nazwy potrzebnych zmiennych, nie ich wartosci.


## 11. Co sprawdzic przy problemie?

| Objaw | Pierwsze sprawdzenie |
| --- | --- |
| Biblioteka jest zainstalowana, a import nie dziala | Porownaj sys.executable z interpreterem PyCharma; uruchom przez uv run. |
| uv odrzuca nieaktualny lock | Sprawdz zmiane pyproject.toml. Aktualizuj lock swiadomie i uruchom testy. Nie usuwaj go, zeby uciszyc blad. |
| Poetry uzywa innego srodowiska | Sprawdz poetry env info i czy terminal nie ma aktywnej .venv innego projektu. |
| Ruff check przechodzi, format --check nie | Lint i format to inne kontrole. Uruchom format, obejrzyj diff i testy. |
| docker pokazuje blad polaczenia z silnikiem | Uruchom Docker Desktop i poczekaj na Server w docker version. |
| Kontener ETL nie laczy sie z baza | Wewnatrz Compose hostem jest db, nie localhost; sprawdz stan db i jego logi. |
| Haslo zmienione w .env.docker nie pasuje | Wolumen moze zawierac baze z dawnym haslem. Nie kasuj go automatycznie; popraw konfiguracje lub swiadomie zmien haslo. |
| Nie ma tabeli po zmianie init.sql | Skrypty inicjalizacji dzialaja na pustym wolumenie. Na istniejacej bazie wykonaj swiadoma zmiane SQL, nie kasowanie danych. |
| ETL zakonczyl sie poprawnie, a kontenera nie widac | Dla run --rm to oczekiwane. Sprawdz plik raw, SQL i logi z zakonczonego uruchomienia. |

Do diagnozy bazy uzyj `docker compose --env-file .env.docker logs db`. Nie zaczynaj od kasowania .venv, locka ani wolumenu. Zwlaszcza `down -v` moze usunac dane bazy i **nie jest zwyklym restartem**.

## 12. Podsumowanie: co umie nowa osoba z repozytorium?

Opis projektu i lock pozwalaja jej odtworzyc biblioteki. Polecenia Ruffa i pytest pozwalaja sprawdzic kod. Dockerfile opisuje obraz ETL, a Compose uruchamia baze i ETL ze wspolna konfiguracja. Sekrety ustawia osobno, bo nie powinny trafic do repozytorium.

Instalacja zakonczona bez bledu, testy, zbudowany obraz i dzialajacy import danych to **cztery rozne poziomy potwierdzenia**. W README zapisujemy, ktore komendy sprawdzaja kazdy z nich.

## Dokumentacja

- [uv: praca z projektem](https://docs.astral.sh/uv/guides/projects/) i [uv w Dockerze](https://docs.astral.sh/uv/guides/integration/docker/)
- [Poetry: podstawy](https://python-poetry.org/docs/basic-usage/) i [grupy zaleznosci](https://python-poetry.org/docs/managing-dependencies/)
- [Ruff: konfiguracja](https://docs.astral.sh/ruff/configuration/) i [formatowanie](https://docs.astral.sh/ruff/formatter/)
- [Docker Compose: kolejnosc startu](https://docs.docker.com/compose/how-tos/startup-order/) i [oficjalny obraz PostgreSQL](https://hub.docker.com/_/postgres)
