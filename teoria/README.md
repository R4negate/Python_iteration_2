# Teoria - Python iteration 2

W tej iteracji budujemy pipeline'y do odczytu, walidacji i zapisu danych z plikow oraz z API.

[Zadania do wszystkich lekcji](../zadania/README.md).

## Lekcje

1. [Pipeline CSV](01_csv_pipeline.md): odczyt, konfiguracja z `.env`, walidacja i zapis poprawnych oraz odrzuconych zamowien.
2. [Pipeline JSON](02_json_pipeline.md): odczyt zagniezdzonych zamowien, walidacja i zapis poprawnych oraz odrzuconych rekordow.
3. [Pobieranie danych z API NASA](03_api_nasa.md): metody i statusy HTTP, podstawy paginacji, odczyt i zapis surowej odpowiedzi.

4. [ETL do PostgreSQL](04_etl_nasa_postgresql.md): NASA EONET -> transformacja w Pythonie -> PostgreSQL.
5. [Wlasne API do odczytu](05_fastapi_postgresql.md): PostgreSQL -> FastAPI -> klient HTTP. API czyta dane zapisane przez ETL i nie uruchamia go na zadanie.
6. [Logowanie pipeline'u](06_logowanie_pipeline.md): poziomy logow, diagnoza awarii i kontrola wyniku.
7. [Testy pipeline'u z pytest](07_testy_pipeline_pytest.md): transformacje, pliki, klient HTTP i test integracyjny.
8. [uv, Poetry, Ruff i Docker](08_uv_poetry_ruff_docker.md): odtwarzanie srodowiska, kontrola kodu i uruchomienie ETL z PostgreSQL w Compose.

Lekcje 01 i 02 korzystaja z [`orders.csv`](../materialy/01_csv/orders.csv) i [`orders.json`](../materialy/02_json/orders.json). Lekcje 03-08 rozwijaja ten sam projekt NASA EONET. Podstawowe testy w 07 nie lacza sie z NASA ani baza.

