# Lab: IDOR w praktyce (OWASP Juice Shop)

## Środowisko i narzędzia
- **Aplikacja:** OWASP Juice Shop. Aplikacja jest uruchamiana lokalnie za pomocą środowiska Docker.
- **Narzędzia:** Przeglądarka internetowa oraz narzędzie Burp Suite Community Edition.

## Konfiguracja środowiska (Docker)
1. Uruchom wiersz poleceń (terminal) i zainicjuj kontener poniższym poleceniem:
   `docker run --rm -p 3000:3000 bkimminich/juice-shop`
2. Otwórz przeglądarkę internetową, przejdź pod adres http://localhost:3000 i zamknij okna powitalne.
3. Uruchom aplikację Burp Suite i skonfiguruj proxy w przeglądarce w celu przechwytywania ruchu sieciowego.

**Uwaga!**

W niektórych przeglądarkach zapytania wysyłane do hosta `localhost` mogą nie być przechwytywane przez Burp Suite. Zaleca się użycie przeglądarki wbudowanej w oprogramowanie Burp Suite, która domyślnie przekierowuje cały ruch przez zintegrowane proxy. W tym celu należy:

1. W programie Burp Suite przejść do zakładki **Proxy > Intercept**.
2. Upewnić się, że opcja **Intercept is on** jest aktywna.
3. Kliknąć przycisk **Open Browser**.
4. Wpisać w pasku adresu przeglądarki docelowy URL: `http://localhost:3000`.

## Przygotowanie scenariusza testowego
1. W panelu aplikacji (Account -> Login) utwórz dwa niezależne konta użytkowników:
   - Ofiara: `victim@agh`
   - Atakujący: `attacker@agh`
2. Zaloguj się na konto `victim@agh` i dodaj do koszyka produkt "Berry Juice". Następnie wyloguj się z systemu.
3. Zaloguj się na konto `attacker@agh` i dodaj do koszyka produkt "Basil Juice". 
4. Wszelkie kolejne kroki w ramach poniższych zadań należy wykonywać, pozostając zalogowanym na koncie atakującego (`attacker@agh`).

---

## Zadanie 1: Nieautoryzowany odczyt koszyka użytkownika (GET)
**Cel:** Uzyskanie wglądu w zawartość koszyka przypisanego do konta `victim@agh` z perspektywy konta `attacker@agh`.

1. Przejdź do widoku własnego koszyka (Your Basket).
2. Zlokalizuj w programie Burp Suite żądanie pobierające zawartość koszyka i zmodyfikuj je w taki sposób, aby serwer zwrócił dane przypisane do konta ofiary.

<details>
<summary><b>Rozwiązanie (Kliknij, aby rozwinąć)</b></summary>

1. W zakładce HTTP history w Burp Suite odszukaj żądanie `GET /rest/basket/X` (gdzie X oznacza identyfikator przypisany do Twojego koszyka).
2. Przekaż wybrane żądanie do modułu Repeater (skrót Ctrl+R).
3. Zmodyfikuj identyfikator koszyka w adresie URL, zastępując go identyfikatorem przypisanym do konta ofiary.
4. Wyślij żądanie, klikając przycisk Send. W odpowiedzi serwer powinien zwrócić obiekt JSON zawierający listę produktów ofiary (w tym produkt Berry Juice).
</details>

---

## Zadanie 2: Nieautoryzowana modyfikacja koszyka użytkownika (POST)
**Cel:** Dodanie nowego produktu bezpośrednio do koszyka użytkownika `victim@agh` z poziomu sesji konta `attacker@agh`.

1. Zainicjuj operację dodania dowolnego produktu do własnego koszyka na stronie sklepu.
2. Przechwyć wygenerowane żądanie i poddaj je modyfikacji w celu skierowania produktu na przypisane ID ofiary.

<details>
<summary><b>Rozwiązanie (Kliknij, aby rozwinąć)</b></summary>

1. Odszukaj w historii ruchu żądanie typu `POST` wysłane pod adres `/api/BasketItems/`.
2. Przekaż to żądanie do modułu Repeater.
3. Zlokalizuj strukturę JSON znajdującą się w ciele żądania (np. `{"ProductId": 3, "BasketId": 2, "quantity": 1}`).
4. Dodaj wartość parametru `BasketId`, wprowadzając identyfikator koszyka ofiary pozyskany w ramach Zadania 1 `{"ProductId": 3, "BasketId": 2, "BasketId": 1, "quantity": 1}`.
5. Wyślij żądanie (Send). Zmiany zostaną zapisane w bazie, a produkt zostanie dopisany do rachunku ofiary.
</details>

---

## Zadanie 3: Manipulacja parametrami autorskimi (PUT / POST)
**Cel:** Dodanie recenzji produktu, która zostanie zapisana w systemie i opublikowana jako recenzja innego użytkownika.

1. Przejdź do sekcji wprowadzania opinii (Customer Feedback).
2. Zainicjuj proces wysyłania formularza, przechwyć odpowiednie żądanie i wymuś na serwerze przypisanie modyfikowanego autorstwa.

<details>
<summary><b>Rozwiązanie (Kliknij, aby rozwinąć)</b></summary>

1. Przechwyć żądanie odpowiedzialne za przesłanie recenzji do serwera.
2. W module Repeater przeanalizuj parametry przesyłane w formacie JSON w ciele żądania.
3. Odszukaj pole określające autora wiadomości (np. parametr `author`).
4. Zastąp własny adres e-mail (`attacker@agh`) adresem ofiary (`victim@agh`) lub dowolnym, nieistniejącym ciągiem znaków.
5. Wyślij żądanie. Aplikacja zaakceptuje zmanipulowaną wartość bez jej weryfikacji i opublikuje opinię pod zmienionym aliasem.
</details>

---

## Zadanie 4: Ominięcie weryfikacji hasła (Bypass)
**Cel:** Wykonanie operacji zmiany hasła do konta z pominięciem mechanizmu weryfikacji aktualnego hasła.

1. Przejdź do formularza zmiany hasła w ustawieniach konta.
2. Wypełnij wymagane pola, wyślij żądanie, a następnie poddaj je modyfikacji w celu ominięcia walidacji po stronie backendu.

<details>
<summary><b>Rozwiązanie (Kliknij, aby rozwinąć)</b></summary>

1. Przechwyć żądanie inicjujące proces zmiany hasła.
2. Przekaż żądanie do modułu Repeater.
3. Wśród przesyłanych parametrów zlokalizuj pole przechowujące podane przez użytkownika aktualne hasło (parametr `currentPassword`).
4. Usuń całkowicie wspomniany parametr z ciała zapytania JSON.
5. Wyślij żądanie. Ze względu na błąd implementacji w API Juice Shop, brak zmiennej zawierającej obecne hasło skutkuje pominięciem procedury weryfikacyjnej i natychmiastowym zaktualizowaniem hasła na nowe.
</details>
