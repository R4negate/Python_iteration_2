# 09. Odporne pobieranie z API od podstaw: timeout, retry i backoff

[Zadanie: rozbuduj pipeline](../zadania/09_pipeline_retry.md)

Pracujemy w `nasa_pipeline` po [lekcji 08](08_uv_poetry_ruff_docker.md). Nie zmieniamy danych, transformacji ani SQL-a. Rozbudujemy `ingestion.py`, zeby chwilowy problem przy pobraniu nie musial od razu przerywac ETL. Testy z [07](07_testy_pipeline_pytest.md) pozwola sprawdzic to bez prowokowania awarii NASA.


## 1. Po co nam odporne pobieranie?

Pipeline moze miec poprawny kod, a mimo to nie pobrac danych. Serwer jest chwilowo przeciazony, polaczenie zostaje przerwane albo odpowiedz nie przychodzi na czas. Jesli po chwili powtorzymy to samo zapytanie, moze juz zadzialac.

Nie kazdy blad jest jednak chwilowy. Zly adres nie stanie sie poprawny tylko dlatego, ze zapytamy o niego trzy razy. Brak hasla nie naprawi sie przez czekanie.

**Odporne pobieranie** oznacza, ze program odroznia sytuacje, w ktorych warto sprobowac ponownie, od tych, w ktorych nalezy zatrzymac prace. Ma tez limit prob i przerwy. Nie oznacza ignorowania bledow ani gwarancji, ze NASA zawsze odpowie.

### Co to jest retry?

**Retry to ponowienie tej samej operacji po jej niepowodzeniu.** U nas ta operacja jest GET z tym samym adresem i parametrami. Retry nie jest harmonogramem codziennego ETL ani pobraniem kolejnej strony danych.

```text
GET -> HTTP 503 -> przerwa -> ten sam GET -> HTTP 200 -> dalszy ETL
```

Przy paginacji z USGS kolejne strony mialy inne offsety. Retry tej strony musi zachowac ten sam offset. Dopiero poprawnie pobrana i obsluzona partia pozwala przejsc do nastepnej. Zwiekszenie offsetu po nieudanym pobraniu mogloby pominac dane.

### Dlaczego nie wystarczy try/except?

`except` pozwala zareagowac na blad. Sam z siebie nie ponawia zadania. Taki kod ukrywalby problem:

```python
try:
    payload = load_payload()
except Exception:
    payload = {"events": []}
```

To **antyprzyklad, nie zmiana do wklejenia do pipeline'u**. Awaria pobrania nie jest tym samym co poprawna odpowiedz z pusta lista. Gdy udajemy puste dane, pipeline moze nadpisac raw i oglosic sukces mimo awarii. Po wykorzystaniu prob przekazemy prawdziwy wyjatek dalej.

## 2. Jakie bledy moga wystapic?

Rozdzielmy trzy sytuacje: brak poprawnej odpowiedzi, odpowiedz z bledem HTTP i poprawny HTTP z blednymi danymi.

| Sytuacja | Jak ja rozpoznajemy w Requests? | Nasza decyzja |
| --- | --- | --- |
| Nie udalo sie polaczyc albo polaczenie sie urwalo | `ConnectionError` | Ponawiamy w limicie, z wyjatkiem problemu SSL. |
| Zbyt dlugie oczekiwanie na operacje sieciowa | `Timeout` | Ponawiamy w limicie. |
| Serwer zwrocil np. HTTP 503 | `raise_for_status()` zglasza `HTTPError` | Ponawiamy tylko statusy z jawnej listy. |
| Problem z bezpiecznym polaczeniem, np. certyfikatem | `SSLError`, ktory jest tez odmiana ConnectionError | Przerywamy. Nie wylaczamy sprawdzania certyfikatu. |
| HTTP jest poprawny, ale body nie jest poprawnym JSON-em | `response.json()` zglasza blad dekodowania | Przerywamy bez retry w tej polityce. |
| JSON da sie odczytac, ale nie ma listy events | Nasza walidacja zglasza `ValueError` | Przerywamy bez retry. |

### HTTPError pojawia sie po sprawdzeniu statusu

Samo `requests.get(...)` zwykle oddaje obiekt odpowiedzi takze dla HTTP 404 lub 503. Dopiero `response.raise_for_status()` zamienia taki status w wyjatek. Dlatego potrzebujemy obu instrukcji. `response.json()` nie potwierdza sukcesu HTTP: serwer moze zwrocic poprawny JSON z opisem bledu.

`{"events": []}` z HTTP 200 jest poprawnym pustym wynikiem. `{}` nie spelnia naszego kontraktu. To, czy blad danych moze byc przejsciowy w innym API, zalezy od jego umowy; **w tej lekcji nie ponawiamy go automatycznie**.

## 3. Timeout i limit prob to dwie rozne rzeczy

```python
response = requests.get(NASA_URL, params=PARAMS, timeout=10)
```

`timeout=10` ogranicza czekanie podczas operacji sieciowych. W Requests jedna liczba jest stosowana do laczenia oraz oczekiwania na odczyt. Nie oznacza „cale pobranie i wszystkie ponowienia skoncza sie w 10 sekund”. Odczyt moze trwac dluzej, jezeli serwer dostarcza kolejne dane w dopuszczalnych odstepach.

**Limit prob** mowi natomiast, ile razy wolno sprobowac pobrac dane:

```python
MAX_ATTEMPTS = 3
```

To **trzy proby lacznie**: pierwsza oraz najwyzej dwie ponowione. Nie trzy dodatkowe proby po pierwszej. Gdy za pierwszym razem jest sukces, nie wykonujemy pozostalych.

Suma czasu pracy zalezy od czasu zapytan i przerw. Nie wyliczamy gwarantowanego limitu czasu calego ETL przez samo `3 * 10`. W bardziej rozbudowanym systemie mozna osobno ustalic budzet czasu calej operacji; tutaj uczymy sie limitowania prob i pojedynczych przerw.

## 4. Backoff: dlaczego robimy przerwy?

Gdy serwer ma problem, natychmiastowe wysylanie dziesiatek kolejnych zapytan moze go dodatkowo obciazyc. **Backoff** to odczekanie przed ponowieniem. Mozemy czekac zawsze tyle samo albo wydluzac przerwy. Wybieramy **exponential backoff**, czyli tutaj podwajanie czasu po kolejnych niepowodzeniach.

```python
wait = 2 ** (attempt - 1)
```

`**` oznacza potegowanie, nie mnozenie. Numer proby zaczynamy od 1:

| Nieudana proba | Obliczenie | Co robimy przy limicie 3 prob? |
| --- | --- | --- |
| 1 | `2 ** 0 = 1` | Czekamy 1 sekunde i probujemy drugi raz. |
| 2 | `2 ** 1 = 2` | Czekamy 2 sekundy i probujemy trzeci raz. |
| 3 | Dalej byloby 4 sekundy | To ostatnia proba: zglaszamy blad, bez czekania. |

```text
proba 1 -> blad -> czekaj 1 s -> proba 2 -> blad -> czekaj 2 s -> proba 3
                                                                       |
                                                              sukces albo blad
```

Funkcja `time.sleep(wait)` wstrzymuje wykonanie tego watku na wskazany czas. Potrzebuje `import time` z biblioteki standardowej. Nie instalujemy go przez uv.

Ustalamy tez limit pojedynczej przerwy, `MAX_WAIT = 30`. Zapis `min(2 ** (attempt - 1), MAX_WAIT)` wybiera mniejsza wartosc: czas rosnie, ale nie przekroczy 30 sekund, gdyby pozniej zwiekszono liczbe prob. To limit naszego backoffu, nie pozwolenie na ignorowanie dluzszego Retry-After od serwera.

## 5. Ustalamy jawna polityke ponowien

```python
RETRYABLE_STATUSES = {429, 500, 502, 503, 504}
```

To **zbior dozwolonych statusow**, nie lista wszystkich bledow. Kazdy inny blad HTTP przerywa pobieranie. Nie zakladamy, ze nowy, nieznany kod ma byc automatycznie ponawiany.

| Status | Proste znaczenie | Co robimy? |
| --- | --- | --- |
| 429 | Serwer ogranicza liczbe zapytan. | Mozliwe ponowienie po przerwie, z uwzglednieniem Retry-After. |
| 500 | Serwer nie wykonal poprawnie zadania. | Dopuszczamy ograniczone ponowienie; nie kazdy 500 zniknie sam. |
| 502, 503, 504 | Problem po stronie serwera lub posrednika. | Dopuszczamy ograniczone ponowienie. |
| 400 | Niepoprawne zapytanie. | Przerywamy, sprawdzamy parametry. |
| 401, 403 | Brak prawidlowego uwierzytelnienia lub dostepu. | Przerywamy; czekanie nie zmienia konfiguracji. |
| 404 | Nie znaleziono zasobu pod adresem. | Przerywamy. |
| Inny, np. 501 | Nie ma go na naszej liscie. | Przerywamy, zamiast zgadywac. |

To polityka **tego klienta**, nie uniwersalna lista dla wszystkich API. Np. status HTTP 408 nie jest w tym cwiczeniu tym samym co wyjatek Requests Timeout: decyzje podejmujemy osobno.

### Dlaczego ponawiamy GET, a nie caly pipeline?

GET pobiera dane bez zlecania ich zmiany. Gdy ponawiamy nasze pobranie, nie zlecamy drugiego utworzenia rekordu na serwerze. To nie oznacza, ze odpowiedz musi byc zawsze identyczna: katalog moze sie zmieniac.

Przy operacji tworzacej cos po drugiej stronie, np. POST, utrata odpowiedzi nie dowodzi, ze serwer nie wykonal zapisu. Ponowienie mogloby utworzyc duplikat. Dlatego nie przenosimy tej petli automatycznie na wszystkie metody HTTP ani na caly ETL. UPSERT w naszej bazie tez nie jest powodem, zeby powtarzac w ciemno wszystkie pozostale kroki.

## 6. Pierwszy krok: retry z backoffem, bez Retry-After

Najpierw robimy wariant, ktory potrafi ograniczyc proby i wydluzyc przerwe. Zastap `ingestion.py` ponizszym kodem. To **pierwsza wersja**, ktora w kroku 8 zostanie rozszerzona.

```python
import logging
import time

import requests
from requests.exceptions import ConnectionError, HTTPError, SSLError, Timeout


NASA_URL = "https://eonet.gsfc.nasa.gov/api/v3/events"
PARAMS = {"status": "all", "start": "2024-08-01", "end": "2024-08-31", "limit": 5}
RETRYABLE_STATUSES = {429, 500, 502, 503, 504}
MAX_ATTEMPTS = 3
MAX_WAIT = 30

logger = logging.getLogger(__name__)


def fetch_response() -> requests.Response:
    for attempt in range(1, MAX_ATTEMPTS + 1):
        wait = min(2 ** (attempt - 1), MAX_WAIT)

        try:
            response = requests.get(NASA_URL, params=PARAMS, timeout=10)
            response.raise_for_status()
            return response
        except HTTPError as error:
            response = error.response
            if response is None or response.status_code not in RETRYABLE_STATUSES:
                raise
            if attempt == MAX_ATTEMPTS:
                raise

            reason = f"HTTP {response.status_code}"
        except (Timeout, ConnectionError) as error:
            if isinstance(error, SSLError) or attempt == MAX_ATTEMPTS:
                raise
            reason = type(error).__name__

        logger.warning(
            "NASA: attempt %s failed (%s); retry in %s s", attempt, reason, wait
        )
        time.sleep(wait)

    raise RuntimeError("Retry loop ended unexpectedly")


def load_payload() -> dict:
    response = fetch_response()
    payload = response.json()
    if not isinstance(payload, dict) or not isinstance(payload.get("events"), list):
        raise ValueError("Expected a JSON object with an events list")
    return payload
```

### Jak podzielilismy odpowiedzialnosci?

- `fetch_response()` odpowiada za zapytanie HTTP, status, limit prob i przerwy. Oddaje obiekt Response albo zglasza blad.
- `load_payload()` korzysta z pobranej odpowiedzi, parsuje JSON i sprawdza liste events. Jej wynik nadal pasuje do pipeline.py: dokument ze zdarzeniami albo wyjatek.
- Transformacja i zapis SQL nie trafily do zadnej z tych funkcji. Ich nie ponawiamy.

### Jak czytac petle po kolei?

1. `range(1, MAX_ATTEMPTS + 1)` daje numery 1, 2, 3. Koniec range jest wylaczony, dlatego dodajemy 1.
2. Na poczatku iteracji wyliczamy wait. Nie oznacza to jeszcze, ze bedziemy czekac.
3. W try wykonujemy GET i sprawdzamy status. `return response` przy sukcesie konczy **cala funkcje**, nie tylko aktualny obieg.
4. Przy HTTPError pobieramy odpowiedz przypisana do wyjatku. Jezeli jej nie ma albo statusu nie ma na liscie, `raise` od razu przekazuje oryginalny wyjatek dalej.
5. Przy Timeout lub ConnectionError mozemy sprobowac ponownie. SSLError jest szczegolnym bledem polaczenia; rozpoznajemy go przez isinstance i przerywamy.
6. Gdy to ostatnia proba, `raise` wykonuje sie przed logiem ponowienia i przed sleep. Nie zapowiadamy kolejnej proby, ktorej juz nie bedzie.
7. Jezeli blad kwalifikuje sie do retry i zostaly proby, logujemy powod, czekamy i przechodzimy do nastepnej iteracji.

Nie lapalismy `Exception`, bo retry ma dotyczyc okreslonych problemow, a nie np. literowki w kodzie czy blednej konfiguracji URL. Wyjatki spoza wymienionych klas wychodza dalej.

### Dwie wazne pulapki

**Sprawdzenie odpowiedzi:** `response is None` pyta, czy nie ma obiektu. Nie zastepuj go przez `if not response`: w Requests odpowiedz z bledem HTTP moze byc logicznie falszywa, chociaz istnieje i zawiera status oraz naglowki.

**Miejsce parsowania:** `response.json()` jest poza petla retry, w load_payload. Uszkodzony JSON nie uruchomi ponownie GET. Bledy i strukture danych sprawdzamy dopiero po uzyskaniu odpowiedzi HTTP bez bledu.

Koncowy `RuntimeError` jest zabezpieczeniem przed nieoczekiwanym wyjsciem z petli, np. po nieprawidlowej zmianie stalej na zero prob. Przy obecnych ustawieniach funkcja wczesniej oddaje odpowiedz albo przekazuje wyjatek.


## 7. Pierwszy test: najpierw 503, potem sukces

Nie potrzebujemy prawdziwego bledu NASA. Utworz `tests/test_retry.py`:

```python
from unittest.mock import Mock, patch

from requests import Response
from requests.exceptions import HTTPError

from ingestion import load_payload


def http_response(status: int, payload: object, headers: dict | None = None):
    response = Mock(spec=Response)
    response.status_code = status
    response.headers = headers if headers is not None else {}
    response.json.return_value = payload
    if 400 <= status < 600:
        response.raise_for_status.side_effect = HTTPError(
            f"HTTP {status}", response=response
        )
    return response


def test_503_then_success():
    # Given: pierwsza odpowiedz bledna, druga poprawna
    failure = http_response(503, {})
    success = http_response(200, {"events": []})

    # When: wykonujemy prawdziwy klient, podmieniamy tylko zaleznosci
    with patch("ingestion.requests.get", side_effect=[failure, success]) as get:
        with patch("ingestion.time.sleep") as sleep:
            result = load_payload()

    # Then: dwie proby, jedna przerwa i wynik drugiej odpowiedzi
    assert result == {"events": []}
    assert get.call_count == 2
    sleep.assert_called_once_with(1)
    failure.json.assert_not_called()
```

Tu podmieniamy GET, ale nie load_payload. Pierwszy GET oddaje zastepcza odpowiedz 503, a drugi odpowiedz 200. Wyjatek pojawia sie w `raise_for_status()`, tak jak w prawdziwym kliencie.

- `Mock(spec=Response)` ogranicza dostepne nazwy metod do interfejsu odpowiedzi Requests; ustawiamy potrzebne atrybuty jawnie. Nie jest to prawdziwe polaczenie.
- `response.json.return_value` ustala wynik metody json. `raise_for_status.side_effect` wskazuje wyjatek z odpowiedzia przypisana do pola response.
- `side_effect=[failure, success]` dostarcza kolejne wyniki kolejnych wywolan GET. To nie sa dwie rownolegle operacje.
- Podmienione sleep tylko zapamietuje argument. Test sprawdza, ze chcielismy czekac sekunde, ale sam nie czeka.
- `failure.json.assert_not_called()` pilnuje, ze nie parsowalismy blednej odpowiedzi jak poprawnego dokumentu.

Uruchom tylko ten nowy test:

```powershell
uv run --locked python -m pytest tests/test_retry.py -v
```

Oczekujemy `1 passed`. Nie uruchamiaj jeszcze prawdziwego ETL do sprawdzania 429: najpierw dodamy obsluge informacji od serwera.

## 8. Retry-After: serwer tez moze ustalic przerwe

Serwer moze dolaczyc do odpowiedzi **naglowek**, czyli dodatkowa informacje obok statusu i body:

```text
HTTP 429
Retry-After: 5
```

To prosba „odczekaj piec sekund przed kolejna proba”. Naglowek nie jest polem w JSON-ie. Odczytujemy go przez `response.headers.get("Retry-After")`. Przy braku naglowka get zwroci None.

Drugi dopuszczalny zapis to data HTTP, np. `Fri, 09 Aug 2024 12:00:05 GMT`: „nie wczesniej niz o tej godzinie”. To inny format niz tekst ISO z pola closed. Uzyjemy parsera dat HTTP, nie datetime.fromisoformat.

### Co wybieramy, gdy czasy sie roznia?

| Nasz backoff | Retry-After | Decyzja |
| --- | --- | --- |
| 1 s | Brak albo niepoprawny zapis | Czekamy 1 s. |
| 1 s | 5 s | Czekamy 5 s. |
| 2 s | 0 s | Czekamy 2 s: zostaje nasz backoff. |
| 1 s | Data piec sekund w przyszlosci | Czekamy co najmniej do tej daty i nie krocej niz nasz backoff. |
| 1 s | 60 s | Przerywamy: przekroczony limit oczekiwania w tym cwiczeniu. |

Mamy limit pojedynczej przerwy 30 sekund. Jesli serwer wymaga 60, **nie skracamy tego do 30 i nie wysylamy zapytania za wczesnie**. Przekazujemy blad do programu, ktory uruchomil pipeline. Inny projekt moglby dopuszczac dluzsze oczekiwanie; to swiadoma decyzja, nie blad biblioteki.

### Docelowy ingestion.py

Teraz zastap pierwsza wersje pliku ponizsza. Funkcje i stale z kroku 6 zostaja z rozszerzeniem o pomocnicze parsowanie naglowka.

```python
import logging
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests
from requests.exceptions import ConnectionError, HTTPError, SSLError, Timeout


NASA_URL = "https://eonet.gsfc.nasa.gov/api/v3/events"
PARAMS = {"status": "all", "start": "2024-08-01", "end": "2024-08-31", "limit": 5}
RETRYABLE_STATUSES = {429, 500, 502, 503, 504}
MAX_ATTEMPTS = 3
MAX_WAIT = 30

logger = logging.getLogger(__name__)


def retry_after_seconds(value: str | None, now: datetime) -> int | float | None:
    if value is None:
        return None

    value = value.strip()
    if value.isascii() and value.isdecimal():
        try:
            return int(value)
        except ValueError:
            return MAX_WAIT + 1

    try:
        retry_at = parsedate_to_datetime(value)
    except (ValueError, TypeError, OverflowError):
        return None

    if retry_at.tzinfo is None:
        retry_at = retry_at.replace(tzinfo=timezone.utc)
    return max(0.0, (retry_at - now).total_seconds())


def fetch_response() -> requests.Response:
    for attempt in range(1, MAX_ATTEMPTS + 1):
        wait = min(2 ** (attempt - 1), MAX_WAIT)

        try:
            response = requests.get(NASA_URL, params=PARAMS, timeout=10)
            response.raise_for_status()
            return response
        except HTTPError as error:
            response = error.response
            if response is None or response.status_code not in RETRYABLE_STATUSES:
                raise
            if attempt == MAX_ATTEMPTS:
                raise

            server_wait = retry_after_seconds(
                response.headers.get("Retry-After"), datetime.now(timezone.utc)
            )
            if server_wait is not None:
                wait = max(wait, server_wait)
            if wait > MAX_WAIT:
                logger.warning("NASA: Retry-After exceeds the allowed wait")
                raise
            reason = f"HTTP {response.status_code}"
        except (Timeout, ConnectionError) as error:
            if isinstance(error, SSLError) or attempt == MAX_ATTEMPTS:
                raise
            reason = type(error).__name__

        logger.warning(
            "NASA: attempt %s failed (%s); retry in %s s", attempt, reason, wait
        )
        time.sleep(wait)

    raise RuntimeError("Retry loop ended unexpectedly")


def load_payload() -> dict:
    response = fetch_response()
    payload = response.json()
    if not isinstance(payload, dict) or not isinstance(payload.get("events"), list):
        raise ValueError("Expected a JSON object with an events list")
    return payload
```

### Co robi retry_after_seconds, linia po linii?

| Fragment | Znaczenie |
| --- | --- |
| Argument value | Tekst naglowka albo None, jesli naglowka brak. |
| Argument now | Biezaca chwila w UTC. Podajemy ja jawnie, zeby test mogl uzyc stalej daty. |
| `if value is None` | Brak wskazowki serwera: oddajemy None, nie zero sekund. |
| `value.strip()` | Usuwamy biale znaki z brzegow. |
| `isascii()` i `isdecimal()` | Dopuszczamy zwykle cyfry 0-9 jako zapis calych sekund. Nie uznajemy np. -5 lub 2.5 za poprawny zapis sekund tego naglowka. |
| `int(value)` | Zamieniamy cyfry na liczbe. |
| `except ValueError: return MAX_WAIT + 1` | Wyjatkowo dlugi zapis liczby moze przekroczyc ograniczenie konwersji Pythona. Zwracamy wartosc przekraczajaca budzet, zeby zatrzymac retry, a nie potraktowac ogromnego zadania jako „naglowka brak”. |
| `parsedate_to_datetime(value)` | Gdy to nie cyfry, probujemy odczytac date HTTP. Funkcja jest z biblioteki standardowej email.utils, mimo ze uzywamy jej do HTTP. |
| Wyjatki parsowania daty | Zly format daje None: wracamy do naszego backoffu. |
| `retry_at.tzinfo is None` | Jezeli parser oddal date bez strefy, przypisujemy UTC jako fallback tego klienta. Nie jest to przeliczanie daty lokalnej. |
| `(retry_at - now).total_seconds()` | Liczymy roznice czasu w sekundach. |
| `max(0.0, ...)` | Data z przeszlosci daje zero, nie ujemne sleep. |

W fetch_response dochodza trzy decyzje: odczyt naglowka, wybor `max(wait, server_wait)` oraz odrzucenie oczekiwania dluzszego niz MAX_WAIT. Naglowek analizujemy tylko dla statusow dopuszczonych do ponowienia i tylko wtedy, gdy zostala kolejna proba. Sam Retry-After przy 401 nie dodaje tego statusu do dozwolonej listy.


## 9. Sprawdzamy kolejne scenariusze, nie prawdziwa awarie

Zastap caly `tests/test_retry.py` ponizsza wersja. Zawiera pierwszy test oraz pozostale przypadki z polityki. Nie dopisuj drugiej definicji tej samej funkcji testowej pod stara.

```python
from datetime import datetime, timezone
from unittest.mock import Mock, patch

import pytest
from requests import Response
from requests.exceptions import HTTPError, Timeout

from ingestion import load_payload, retry_after_seconds


def http_response(status: int, payload: object, headers: dict | None = None):
    response = Mock(spec=Response)
    response.status_code = status
    response.headers = headers if headers is not None else {}
    response.json.return_value = payload
    if 400 <= status < 600:
        response.raise_for_status.side_effect = HTTPError(
            f"HTTP {status}", response=response
        )
    return response


def test_503_then_success():
    # Given
    failure = http_response(503, {})
    success = http_response(200, {"events": []})

    # When
    with patch("ingestion.requests.get", side_effect=[failure, success]) as get:
        with patch("ingestion.time.sleep") as sleep:
            result = load_payload()

    # Then
    assert result == {"events": []}
    assert get.call_count == 2
    sleep.assert_called_once_with(1)
    failure.json.assert_not_called()


def test_429_respects_retry_after():
    failure = http_response(429, {}, {"Retry-After": "5"})
    success = http_response(200, {"events": []})

    with patch("ingestion.requests.get", side_effect=[failure, success]):
        with patch("ingestion.time.sleep") as sleep:
            load_payload()

    sleep.assert_called_once_with(5)


@pytest.mark.parametrize("status", [400, 401, 403, 404, 408, 422, 501])
def test_status_outside_allowlist_is_not_retried(status):
    response = http_response(status, {})

    with patch("ingestion.requests.get", return_value=response) as get:
        with patch("ingestion.time.sleep") as sleep:
            with pytest.raises(HTTPError):
                load_payload()

    assert get.call_count == 1
    sleep.assert_not_called()
    response.json.assert_not_called()


def test_timeout_stops_after_three_attempts():
    error = Timeout("No response")

    with patch("ingestion.requests.get", side_effect=error) as get:
        with patch("ingestion.time.sleep") as sleep:
            with pytest.raises(Timeout) as raised:
                load_payload()

    assert raised.value is error
    assert get.call_count == 3
    assert [call.args[0] for call in sleep.call_args_list] == [1, 2]


def test_retry_after_exceeds_allowed_wait():
    response = http_response(429, {}, {"Retry-After": "60"})

    with patch("ingestion.requests.get", return_value=response) as get:
        with patch("ingestion.time.sleep") as sleep:
            with pytest.raises(HTTPError):
                load_payload()

    assert get.call_count == 1
    sleep.assert_not_called()


def test_invalid_payload_is_not_retried():
    response = http_response(200, {})

    with patch("ingestion.requests.get", return_value=response) as get:
        with patch("ingestion.time.sleep") as sleep:
            with pytest.raises(ValueError, match="events list"):
                load_payload()

    assert get.call_count == 1
    sleep.assert_not_called()


def test_retry_after_http_date():
    now = datetime(2024, 8, 9, 12, 0, 0, tzinfo=timezone.utc)

    seconds = retry_after_seconds("Fri, 09 Aug 2024 12:00:05 GMT", now)

    assert seconds == 5
```

### Jak rozumiec wyniki?

Dla 503 przed sukcesem widzimy dwa GET i jedna przerwe. Dla trzech timeoutow sa trzy GET, lecz tylko dwie przerwy: sprawdzenie `raised.value is error` potwierdza dodatkowo, ze nie zgubilismy pierwotnego wyjatku przez zamiane na ogolny ValueError.

Parametryzowany test statusow obejmuje tez 501 i 408: nie sa na liscie, wiec nie ponawiamy ich „na wszelki wypadek”. Test z Retry-After 60 ma jeden GET i zadnego sleep. Daty testujemy ze stala chwila now, nie rzeczywistym zegarem komputera.

Ten plik daje **13 przypadkow testowych**: siedem statusow w parametryzacji oraz szesc pozostalych scenariuszy. Liczba przypadkow nie jest tym samym co liczba definicji funkcji.

### Dostosowanie jednego testu z lekcji 07

Zostaw dotychczasowe testy. W `tests/test_ingestion.py` zastap tylko `test_load_payload_propagates_timeout` ponizszym kodem. Nadal sprawdza przekazanie timeoutu, ale teraz musi podmienic tez czekanie:

```python
def test_load_payload_propagates_timeout():
    from requests import Timeout

    with patch("ingestion.requests.get", side_effect=Timeout("Test timeout")):
        with patch("ingestion.time.sleep") as sleep:
            with pytest.raises(Timeout):
                load_payload()

    assert [call.args[0] for call in sleep.call_args_list] == [1, 2]
```

Test z 07, ktory przekazuje HTTPError **bez przypisanej odpowiedzi**, nadal przerywa od razu: nie znamy statusu i nie zgadujemy. To inny przypadek niz rzeczywista odpowiedz 503 z testu powyzej.

### Uruchomienie calego zestawu

```powershell
uv run --locked python -m pytest -q tests
uv run --locked ruff check .
uv run --locked ruff format --check .
```

Jesli Ruff wskaze format nowych plikow, wykonaj `uv run --locked ruff format ingestion.py tests/test_retry.py tests/test_ingestion.py`, obejrzyj diff i uruchom testy jeszcze raz. Nowe mechanizmy korzystaja z requests oraz biblioteki standardowej, wiec nie dodajemy nowych zaleznosci i nie aktualizujemy locka bez potrzeby.

W podstawowym zestawie po 07 mielismy 24 przypadki. Po zastapieniu starego testu timeoutu i dodaniu tego pliku oczekujemy **37 passed** oraz `1 skipped`, jesli dodales niewlaczony test PostgreSQL. Wlasne dodatkowe testy lub dodatki DataFrame moga zmienic te liczby.

### Gdy masz kod w pakiecie albo argumenty w load_payload

Przy pakiecie import testowy ma postac `from nasa_pipeline.ingestion import load_payload, retry_after_seconds`, a podmiany trafiaja do `nasa_pipeline.ingestion.requests.get` i `nasa_pipeline.ingestion.time.sleep`. Uruchamiaj pytest z katalogu projektu, nie przez bezposredni start pliku ingestion.py.

Jesli Twoje load_payload przyjmuje url i params, zachowaj te argumenty i przekaz je rowniez do fetch_response oraz do requests.get. W testach wywoluj funkcje z tymi samymi argumentami. Podane kompletne wersje odpowiadaja bezargumentowemu kontraktowi z 03-08; nie zamieniaj wywolania w swoim main.py bez sprawdzenia sygnatury.


## 10. Gdzie konczy sie retry, a zaczyna awaria ETL?

Na koniec mozesz uruchomic zwykly pipeline z prawidlowa lokalna konfiguracja:

```powershell
uv run --locked python main.py
```

To **rzeczywisty ETL**: pobierze NASA, zapisze raw i wykona SQL, jesli poprzednie kroki sie powioda. Nie uruchamiaj go na bazie produkcyjnej. Podstawowe testy nie wymagaly tego uruchomienia.

Gdy NASA odpowie od razu, brak warningow retry jest poprawny. Nie znaczy, ze petla sie nie uruchomila: pierwsze pobranie zakonczylo funkcje przez return.

Gdy wszystkie proby zawioda, load_payload nie oddaje dokumentu. Pipeline z 06 nie dochodzi do zapisu raw, transformacji ani SQL. Blad dociera do main, ktory loguje awarie i konczy z kodem 1. Raw z poprzedniego udanego przebiegu moze nadal istniec: **nie jest wynikiem dzisiejszej nieudanej proby**.

| Wpis | Co faktycznie potwierdza? |
| --- | --- |
| `attempt 1 failed ... retry in 1 s` | Jedna proba zawiodla, planujemy nastepna. Nie oglaszamy jeszcze porazki calego ETL. |
| Log sukcesu pobrania po retry | Udalo sie otrzymac i sprawdzic dane; SQL moze byc dopiero przed nami. |
| Blad po ostatniej probie | Pobranie zostalo przerwane, nie wolno oglaszac sukcesu. |
| Sukces testow retry | Sprawdzone scenariusze dzialaja z kontrolowanymi odpowiedziami. Nie jest to pomiar dostepnosci NASA. |

### Samodzielnie: przypadki, ktorych jeszcze nie ma w pliku

Dopisz testy dla: sukcesu za pierwszym razem bez sleep; ConnectionError zakonczonego sukcesem; SSLError bez retry; Retry-After niepoprawnego lub z data w przeszlosci; uszkodzonego JSON-a bez retry.

Dodaj tez test pipeline'u: jezeli pobranie ostatecznie zawiedzie, funkcje zapisu raw i SQL nie sa wywolane. Gdy przygotujesz w tmp_path wczesniejszy raw, sprawdz, ze jego zawartosc zostala bez zmian. Podmieniaj odpowiedzi i sleep, nie wylaczaj internetu ani prawdziwego serwera.


## 11. Co warto wiedziec przed uzyciem w wiekszym projekcie?

**Jitter** to niewielkie losowe zroznicowanie przerw. Gdy wiele procesow zawiedzie naraz i kazdy czeka dokladnie sekunde, ponowia zapytania jednoczesnie. Losowy dodatek pomaga je rozsunac. Nie dodajemy go do pierwszej wersji, zeby latwo bylo policzyc opoznienia. Przy rozszerzeniu trzeba uwzglednic limit i nie sprobowac wczesniej, niz pozwala Retry-After.

Gotowa obsluge retry oferuje tez urllib3 z Requests HTTPAdapter. Na tej lekcji jawna petla pozwala przejsc przez kazda decyzje i ja przetestowac. W wiekszym projekcie warto rozpatrzyc biblioteke zamiast utrzymywac rozbudowany wlasny mechanizm. Nie nakladaj kilku niezaleznych retry na siebie bez policzenia lacznej liczby prob: petla aplikacji i retry w adapterze moglyby pomnozyc zapytania.

Zostajemy przy waskim zakresie: retry jednego pobrania GET. Nie dokladamy retry SQL, automatycznego resetowania bazy, zapisu pliku zastepczego ani cichego pomijania stron.

## Dokumentacja

- [Requests: timeout i wyjatki](https://requests.readthedocs.io/en/latest/user/quickstart/#errors-and-exceptions)
- [HTTP: znaczenie i format Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.3)
- [urllib3: gotowy mechanizm Retry](https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html#urllib3.util.Retry)
