# Lab: IDOR / BOLA (Insecure Direct Object Reference)

## Środowisko i konfiguracja
- **Cel:** Praktyczne wykorzystanie braku weryfikacji uprawnień do obiektów w API.
- **Aplikacja:** OWASP Juice Shop (lokalny kontener Docker).
- **Narzędzia:** Przeglądarka internetowa, Burp Suite.

1. Uruchom aplikację w terminalu:
   `docker run --rm -p 3000:3000 bkimminich/juice-shop`
2. Wejdź na http://localhost:3000 i zamknij komunikaty powitalne.
3. Przejdź do Account -> Login. Utwórz dowolne konto (np. test@test.pl / admin123) i zaloguj się. Wymagane do wygenerowania koszyka przypisanego do sesji.
4. Uruchom Burp Suite i włącz proxy w przeglądarce.
   
**Uwaga!**

W niektórych przeglądarkach zapytania wysyłane do localhost nie są przechwytuwane przez Burp Suite. Najlepiej użyć wbudowanej w Burpa przeglądarki, która przekieruje cały ruch przez proxy. Aby to zrobić: 

1. W Burp Suite przejdź do zakładki Proxy > Intercept.
2. Upewnij się, że opcja Intercept is on jest włączona.
3. Kliknij przycisk Open Browser.
4. Wpisz w pasku adresu docelowy URL http://localhost:3000.

## Zadanie 1: Nieautoryzowany odczyt (IDOR w żądaniu GET)

1. Kliknij "Your Basket" w prawym górnym rogu strony.
2. W Burp Suite przejdź do Proxy -> HTTP history. Znajdź żądanie `GET /rest/basket/3` (cyfra to ID twojego koszyka).
3. Przekaż żądanie do Repeatera (Ctrl+R).
4. Zmień ID koszyka w ścieżce na 1 (koszyk administratora): `GET /rest/basket/1`.
5. Wyślij żądanie (Send).

**Wynik:** Serwer zwraca kod 200 OK i całą zawartość koszyka administratora. Brak autoryzacji na poziomie konkretnego obiektu pozwala na odczyt cudzych danych przy użyciu ważnej, ale obcej sesji.

<details>
<summary><b>Podpowiedź: Gdzie szukać parametrów?</b></summary>
Testuj parametry numeryczne w adresach URL, nagłówkach HTTP (np. X-User-Id) oraz bezpośrednio w ciałach zapytań (JSON).
</details>

## Zadanie 2: Nieautoryzowana modyfikacja (BOLA w żądaniu POST)

1. Na stronie głównej kliknij "Add to basket" przy dowolnym produkcie.
2. W Burp Suite odszukaj żądanie `POST /api/BasketItems/`.
3. Przekaż żądanie do Repeatera. Na dole zapytania znajduje się struktura JSON, np.: `{"ProductId": 7, "BasketId": 3, "quantity": 1}`.
4. Zmień parametr `BasketId` z twojego ID na 1.
5. Wyślij żądanie (Send).

**Wynik:** Baza danych zapisuje zmianę i produkt zostaje dodany do koszyka ofiary. Podatność umożliwia bezpośrednią modyfikację zasobów przypisanych do innych kont z pominięciem interfejsu graficznego.

<details>
<summary><b>Podpowiedź: IDOR vs BOLA</b></summary>
IDOR to starsza, ogólna nazwa. BOLA (Broken Object Level Authorization) pochodzi z klasyfikacji OWASP API Top 10 i opisuje dokładnie ten sam błąd, kładąc nacisk na brak weryfikacji na poziomie obiektu w architekturze API.
</details>

## Zadanie 3: Mitygacja w kodzie backendu

Serwer nie może polegać wyłącznie na identyfikatorach dostarczonych z zewnątrz.

**Błędna logika (podatna na atak):**
Klient żąda `GET /basket/1`.
Serwer wykonuje: `SELECT * FROM baskets WHERE id = 1`

**Poprawna logika (weryfikacja właściciela):**
1. Serwer odbiera żądanie odczytu obiektu nr 1.
2. Serwer pobiera identyfikator użytkownika z bezpiecznego tokena sesji (np. JWT).
3. Serwer weryfikuje powiązanie w bazie danych: `if (ID_uzytkownika != ID_wlasciciela_koszyka) { rzuć błąd 403 Forbidden }`.

Nowoczesne aplikacje powinny używać endpointów typu indirect, np. `GET /my-basket`, gdzie backend całkowicie ignoruje dane z frontendu i samodzielnie mapuje token sesji na właściwy obiekt w bazie.

<details>
<summary><b>Podpowiedź: Ograniczenia skanerów</b></summary>
Skanery DAST (automatyczne) nie znają logiki biznesowej aplikacji. Nie potrafią ocenić, czy dostęp użytkownika A do zasobów użytkownika B jest błędem, czy prawidłową funkcją systemu. Z tego powodu testy weryfikujące IDOR trzeba wykonywać ręcznie.
</details>

## Zadanie 4: Nieautoryzowane usuwanie danych (BOLA w żądaniu DELETE)

1. Wejdź w swój profil i dodaj nowy adres dostawy (Saved Addresses).
2. Kliknij ikonę kosza, aby usunąć ten adres i złap to w Burpie (Proxy -> HTTP history).
3. Znajdź żądanie `DELETE /api/Addresss/3` (gdzie 3 to ID twojego adresu).
4. Przekaż żądanie do Repeatera (Ctrl+R).
5. Zmień ID w adresie URL na mniejsze, np. na 1 (adres przypisany do konta administratora): `DELETE /api/Addresss/1`.
6. Wyślij żądanie (Send).

**Wynik:** Serwer zwraca kod 200 OK lub 204 No Content. Adres innego użytkownika został trwale skasowany z bazy. Pokazuje to, że brak weryfikacji uprawnień dotyczy również operacji niszczących (kasowania zasobów).

<details>
<summary><b>Podpowiedź: Metody HTTP</b></summary>
Podatności BOLA bardzo często kryją się w metodach takich jak DELETE, PUT czy PATCH. Programiści nierzadko pamiętają o weryfikacji właściciela przy odczycie (GET), ale zapominają o niej przy aktualizacji lub usuwaniu rekordów.
</details>

## Zadanie 5: IDOR na wygenerowanych plikach (wyciek dokumentów)

1. Przejdź przez proces zakupu (Checkout) i sfinalizuj dowolne zamówienie. Wejdź w historię zamówień (Order History).
2. Kliknij ikonę drukowania/pobierania potwierdzenia zamówienia w formie pliku (Track Order / Receipt).
3. W Burpie odszukaj żądanie pobierania pliku. Będzie ono wyglądać podobnie do: `GET /ftp/receipts/order-1045.pdf` (albo poprzez ścieżkę z samym ID zamówienia).
4. W Repeaterze podmień numer zamówienia w URL na starszy, np. zmień `1045` na `1001`.
5. Wyślij żądanie (Send).

**Wynik:** Pobierasz dokument PDF należący do innego klienta. Wiele systemów poprawnie weryfikuje dostęp do danych w samej bazie (np. API chronione przez tokeny), ale zostawia serwowanie gotowych plików statycznych (np. faktur generowanych na dysk serwera) całkowicie bez autoryzacji.

<details>
<summary><b>Podpowiedź: Enumeracja ID</b></summary>
W realnym teście penetracyjnym takie żądanie wysyła się z Repeatera do modułu Intruder. Ustawia się listę numerów od 1 do 1000, a Burp automatycznie pobiera wszystkie cudze faktury, jedną po drugiej, bez ręcznego wpisywania ID.
</details>
