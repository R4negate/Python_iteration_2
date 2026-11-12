# 09. Odporne pobieranie z API: timeout, retry i Retry-After

Pipeline dziala codziennie. Dzisiaj pierwsza proba trafia na chwilowy problem: serwer zwraca `503`. Gdybys uruchomil skrypt jeszcze raz po chwili, moglby zadzialac. **Retry** oznacza, ze program sam wykona kolejna probe. Nie probuje jednak w nieskonczonosc i nie powtarza kazdego bledu.

Kontynuuj `nasa_pipeline` po [lekcji 08](08_uv_poetry_ruff_docker.md). Zmienimy tylko klienta pobierajacego w `ingestion.py` i dodamy testy. Pipeline nadal pobiera prawdziwe dane z NASA. Nie dodajemy pliku zastepczego ani trybu offline.

[Zadanie: rozbuduj pipeline](../zadania/09_pipeline_retry.md)

## 1. Kiedy czekac, a kiedy sie zatrzymac?

| Co sie wydarzylo? | Co zrobi nasz program? |
|---|---|
| Nie dostalismy odpowiedzi na czas albo polaczenie zostalo przerwane. | Ponowi probe w ustalonym limicie. |
| Dostajemy `429`: wyslalismy za duzo zapytan. | Sprawdzi, czy serwer podal `Retry-After`, czyli ile odczekac. |
| Dostajemy wybrany chwilowy blad serwera, np. `503`. | Odczeka i sprobuje jeszcze raz. |
| Dostajemy `401`, `403` lub `404`. | Zglosi blad od razu. Czekanie nie poprawi loginu, uprawnien ani adresu. |
| Dostajemy `200`, ale JSON jest uszkodzony albo brakuje listy `events`. | Zglosi blad danych, bez kolejnej proby. |

**Timeout** dotyczy czekania w pojedynczym zadaniu. **Limit prob** mowi, ile razy wolno je wyslac. Tu mamy najwyzej trzy proby **lacznie**: pierwsza i maksymalnie dwie kolejne. Nie oznacza to, ze caly przebieg potrwa najwyzej 10 sekund.

Nie powtarzamy automatycznie dowolnych operacji zapisu. W tym cwiczeniu ponawiamy tylko `GET` do NASA, nie `INSERT`, zapis pliku ani calego ETL.

## 2. Najpierw ustal czas oczekiwania

Jesli serwer zwroci `Retry-After: 5`, prosi o odczekanie pieciu sekund. Moze tez podac date w czasie UTC: "sprobuj dopiero o tej godzinie". Zamienimy oba zapisy na liczbe sekund.

Gdy nie ma poprawnej wskazowki, odczekamy 1 sekunde po pierwszym bledzie i 2 sekundy po drugim. Rosnaca przerwa to **backoff**. W tej lekcji nie chcemy blokowac pojedynczej proby ponowienia na dluzej niz 30 sekund. Jesli serwer prosi o 60 sekund, przerywamy przebieg; nie probujemy po 30 wbrew wskazowce.

## 3. Zastap `ingestion.py`

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
        wait = 2 ** (attempt - 1)

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

Czytaj kod od `load_payload()`. Ta funkcja nadal robi to, czego oczekuje `pipeline.py`: zwraca dokument z `events` lub zglasza blad. W srodku `fetch_response()` odpowiada tylko za pobranie poprawnej odpowiedzi HTTP. JSON parsujemy dopiero **poza petla retry**.

W petli `return response` konczy prace po sukcesie. Jesli odpowiedz ma status z listy do ponowienia albo wystapi chwilowy blad polaczenia, log pokazuje numer nieudanej proby, powod i planowana przerwe. Przy ostatniej probie `raise` przekazuje oryginalny wyjatek dalej, bez kolejnego `sleep()`. `SSLError` tez jest bledem polaczenia, ale nie ponawiamy go w tej lekcji: moze oznaczac problem z certyfikatem.

`error.response is None` sprawdza, czy w ogole mamy odpowiedz. Nie uzywamy `if not response`: w Requests odpowiedz z bledem HTTP moze miec wartosc logiczna `False`, chociaz zawiera status i naglowki. Dla daty `now` przekazujemy wprost, zeby w tescie moc podstawic stala godzine.

## 4. Przetestuj retry bez czekania na awarie NASA

Zostaw testy z lekcji 07. Dodaj `tests/test_retry.py`:

```python
from datetime import datetime, timezone
from unittest.mock import Mock, patch

import pytest
from requests.exceptions import HTTPError, Timeout

from ingestion import load_payload, retry_after_seconds


def http_error(status: int, headers: dict | None = None) -> HTTPError:
    response = Mock(status_code=status, headers=headers or {})
    return HTTPError(response=response)


def test_503_then_success():
    response = Mock()
    response.json.return_value = {"events": []}

    with patch("ingestion.requests.get", side_effect=[http_error(503), response]) as get:
        with patch("ingestion.time.sleep") as sleep:
            assert load_payload() == {"events": []}

    assert get.call_count == 2
    sleep.assert_called_once_with(1)


def test_429_respects_retry_after():
    response = Mock()
    response.json.return_value = {"events": []}
    with patch(
        "ingestion.requests.get",
        side_effect=[http_error(429, {"Retry-After": "5"}), response],
    ):
        with patch("ingestion.time.sleep") as sleep:
            load_payload()
    sleep.assert_called_once_with(5)


def test_401_is_not_retried():
    with patch("ingestion.requests.get", side_effect=http_error(401)) as get:
        with patch("ingestion.time.sleep") as sleep:
            with pytest.raises(HTTPError):
                load_payload()

    assert get.call_count == 1
    sleep.assert_not_called()


def test_timeout_stops_after_three_attempts():
    with patch("ingestion.requests.get", side_effect=Timeout) as get:
        with patch("ingestion.time.sleep") as sleep:
            with pytest.raises(Timeout):
                load_payload()

    assert get.call_count == 3
    assert [call.args[0] for call in sleep.call_args_list] == [1, 2]


def test_retry_after_exceeds_allowed_wait():
    error = http_error(429, {"Retry-After": "60"})
    with patch("ingestion.requests.get", side_effect=error) as get:
        with patch("ingestion.time.sleep") as sleep:
            with pytest.raises(HTTPError):
                load_payload()

    assert get.call_count == 1
    sleep.assert_not_called()


def test_invalid_payload_is_not_retried():
    response = Mock()
    response.json.return_value = {}
    with patch("ingestion.requests.get", return_value=response) as get:
        with pytest.raises(ValueError, match="events list"):
            load_payload()

    assert get.call_count == 1


def test_retry_after_http_date():
    now = datetime(2024, 8, 9, 12, 0, 0, tzinfo=timezone.utc)
    assert retry_after_seconds("Fri, 09 Aug 2024 12:00:05 GMT", now) == 5
```

Pierwszy test ustawia dwa wyniki: najpierw blad, potem sukces. `patch("ingestion.time.sleep")` pozwala sprawdzic, ile sekund skrypt **chcial** czekac, ale nie zatrzymuje testu. Po `Timeout` widzimy trzy zadania, lecz tylko dwie przerwy. Nie musimy wylaczac internetu ani zmieniac prawdziwego serwera.

```powershell
uv run --locked python -m pytest -q tests
```

Na koniec uruchom `uv run --locked python main.py` z poprawna konfiguracja ETL. Gdy NASA odpowie od razu, brak wpisow retry jest poprawnym wynikiem. Jesli wszystkie proby pobrania zawioda, wciaz nie nadpisujemy raw ani nie uruchamiamy zapisu SQL. O tym decyduje kolejnosc etapow z 04 i 06.

## Dokumentacja

- [Requests: timeout i wyjatki](https://requests.readthedocs.io/en/latest/user/quickstart/#errors-and-exceptions)
- [HTTP: znaczenie i format Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.3)
