# 02. Zbuduj pipeline zagniezdzonych zamowien JSON

Sklep zmienil sposob eksportu: dane klienta sa teraz w obiekcie `customer`, a tagi w tablicy. Zespol chce nadal dostac jeden oczyszczony rekord na zamowienie, bez powielania kwot przez liczbe tagow.

## Punkt wyjscia

Utworz projekt `datashop_json`. Wez [orders.json](../materialy/02_json/orders.json) i umiesc go w `data/raw/orders.json`. Sciezki wejsciowa i wyjsciowa maja byc w `.env`.

[Material do lekcji](../teoria/02_json_pipeline.md).

## Co zbudowac

1. Wczytaj caly dokument. Sprawdz, czy glowna wartosc jest obiektem z lista `orders` i waluta `PLN`. Blednego dokumentu albo innej waluty nie przetwarzaj jak pustej listy zamowien.
2. Z kazdego zamowienia utworz jeden plaski rekord: `order_id`, `customer_id`, `customer_name`, `status`, `amount`, `country`, `is_gift` i `tags`. ID maja byc dodatnimi liczbami calkowitymi, nie tekstem ani `bool`.
3. Uzyj regul nazw, statusow i kwot z 01. Kraj brakujacy, pusty lub `null` zamien na `UNKNOWN`. Brak `is_gift` daje `False`, a wartosc obecna musi byc `bool`. Brakujace lub `null` tagi daja pusta liste. Pozostale tagi oczysc, zmien na male litery, usun puste i duplikaty, a wynik posortuj. Przy powtorzonym ID zachowaj pierwszy poprawny rekord.
4. Zapisz poprawne i odrzucone rekordy osobno w `data/out/clean_orders.json` i `data/out/rejected_orders.json`. Odrzucenie ma wskazac plik, pozycje, surowe ID, typ i powod bledu. Przy eksporcie zamien `Decimal` na tekst bez zmiany obiektu trzymanego w pamieci. Tagi pozostaja tablica.
5. Uruchamiaj wszystko z `main.py`, ktory wywoluje pipeline i wyswietla liczniki. Osobno trzymaj odczyt, transformacje i eksport.

## Jak sprawdzic wynik

- Z pliku powstaje 5 poprawnych rekordow i 2 odrzucenia, po jednym poprawnym rekordzie na ID.
- Sprawdz `1001`: `tags` to `["promo", "sql"]` i `amount` to `"120.50"`. `1003` zachowuje kwote zero, `is_gift: true` i pusta liste tagow.
- Po drugim uruchomieniu pliki maja tyle samo rekordow.
