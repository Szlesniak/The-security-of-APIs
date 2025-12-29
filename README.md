# Kompletny Przewodnik Hacking API: crAPI (Completely Ridiculous API)

Ten tutorial przeprowadzi Cię przez proces instalacji środowiska testowego crAPI, konfigurację narzędzi hakerskich (Burp Suite) oraz wykonanie 5 kluczowych ataków na API (OWASP API Top 10).

---

## Część 1: Przygotowanie Środowiska (Windows)

Zanim zaczniesz hakować, musisz postawić "ofiarę" i przygotować "broń".

### 1. Instalacja crAPI
crAPI działa na Dockerze. Upewnij się, że masz zainstalowanego Dockera i Docker Compose.

1.  **Pobierz repozytorium:**
    ```bash
    curl.exe -o crapi.zip -L https://github.com/OWASP/crAPI/archive/refs/heads/main.zip

    tar -xf .\crapi.zip

    cd crAPI-main\deploy\docker

    docker-compose pull

    docker-compose -f docker-compose.yml --compatibility up -d
    ```
    *Czekaj około 2-3 minut, aż wszystkie serwisy (Identity, Workshop, MailHog) wstaną. Java potrzebuje czasu!*

3.  **Rozwiązywanie problemów:**
    Jeśli aplikacja nie ładuje się lub sypie błędami 500:
    ```bash
    docker-compose restart
    ```

Dostęp do aplikacji: `http://localhost:8888`
Dostęp do maili (MailHog): `http://localhost:8025`

### 2. Konfiguracja Burp Suite (Broń)
Burp to proxy, które pozwala przechwytywać i modyfikować ruch między przeglądarką a serwerem.

1.  **Pobierz:** Burp Suite Community Edition.

---

## Część 2: Ataki na API

### 1. BOLA (Broken Object Level Authorization) - Challenge 5
**Cel:** Zmiana nazwy wideo innego użytkownika.
**Wyjaśnienie:** Serwer nie sprawdza, czy ID obiektu w URL należy do Ciebie.

1.  Zaloguj się
2.  
6.  **Sukces:** Jeśli otrzymasz status `200 OK`, zmieniłeś nazwę zasobu, do którego nie powinieneś mieć dostępu.

### 2. BFLA (Broken Function Level Authorization) - Challenge 7
**Cel:** Usunięcie wideo innego użytkownika (funkcja administracyjna).
**Wyjaśnienie:** Serwer nie sprawdza uprawnień (roli) dla ukrytych endpointów administracyjnych.

1.  Znajdź zapytanie edycji wideo (`PUT /identity/api/v2/user/videos/{ID}`).
2.  Wyślij je do **Repeatera**.
3.  Zmień metodę z `PUT` na `DELETE`.
4.  Zmodyfikuj URL, zgadując endpoint admina (zamień `user` na `admin`):
    * Stary: `/identity/api/v2/user/videos/6`
    * Nowy: `/identity/api/v2/admin/videos/0`
    *(Celujemy w ID 0, żeby usunąć wideo systemowe/admina).*
5.  Wyślij zapytanie.
6.  **Sukces:** Status `200 OK` lub `204 No Content` oznacza, że zwykły użytkownik wykonał akcję admina.

### 3. Rate Limiting (Broken User Authentication) - Challenge 3
**Cel:** Zresetowanie hasła innego użytkownika poprzez odgadnięcie OTP.
**Wyjaśnienie:** API nie blokuje konta po wielu nieudanych próbach wpisania kodu.

1.  Na ekranie logowania wybierz "Forgot Password".
2.  Podaj email ofiary (można go znaleźć w wyciekach danych w dashboardzie, np. `adam007@example.com`).
3.  Gdy pojawi się prośba o OTP, wpisz losowy kod (np. `0000`) i przechwyć zapytanie w Burpie.
4.  Wyślij do **Intrudera** (`Ctrl + I`).
5.  W zakładce **Positions** zaznacz kod `0000` i kliknij `Add §`.
6.  W zakładce **Payloads**:
    * Payload type: **Numbers**.
    * From: `0000`, To: `9999`.
7.  Kliknij **Start Attack**.
8.  **Sukces:** Obserwuj kolumnę "Length" lub "Status". Jeden request będzie inny (zwróci `200 OK` zamiast `500`). To jest poprawny kod OTP.

### 4. SSRF (Server Side Request Forgery) - Challenge 8
**Cel:** Zmuszenie serwera crAPI do połączenia się z wewnętrzną infrastrukturą (MailHog).
**Wyjaśnienie:** Serwer ufa adresom URL podanym przez użytkownika i łączy się z nimi.

1.  Zaloguj się i przejdź do "Contact Mechanic" (Warsztat).
2.  Wypełnij formularz i przechwyć request.
    ```http
    POST /workshop/api/merchant/contact_mechanic HTTP/1.1
    ...
    {"mechanic_api": "[http://mechanic.crapi.io/](http://mechanic.crapi.io/)..."}
    ```
3.  Zmień wartość `mechanic_api` na: `http://mailhog:8025/api/v2/messages`.
    *(MailHog to wewnętrzny serwis pocztowy działający w Dockerze, niedostępny z zewnątrz).*
4.  Wyślij zapytanie (Repeater).
5.  **Sukces:** W odpowiedzi (Response) otrzymasz JSON zawierający treść wszystkich e-maili w systemie (w tym kody OTP innych użytkowników).

### 5. JWT Forging (Masowe generowanie tokenów)
**Cel:** Uzyskanie dostępu admina poprzez sfałszowanie tokena.
**Wyjaśnienie:** crAPI w jednej z wersji używa słabego sekretu do podpisywania tokenów lub pozwala na algorytm "None".

1.  Skopiuj swój token JWT (z nagłówka `Authorization: Bearer ...`).
2.  Wejdź na stronę `jwt.io`.
3.  Wklej token. W sekcji "Payload" zmień:
    * `"role": "user"` na `"role": "admin"`
    * (Opcjonalnie) `"sub": "admin@example.com"`
4.  W sekcji "Verify Signature" wpisz sekret: **`crAPI`** (jest to domyślny, słaby sekret w tym API).
5.  Skopiuj nowy, wygenerowany po lewej stronie token.
6.  Użyj go w Burpie, podmieniając oryginalny token w zapytaniu do endpointu administracyjnego.

---

## Jak się bronić? (Wnioski dla programistów)

1.  **BOLA:** Zawsze sprawdzaj po stronie serwera, czy `user_id` z sesji (tokena) ma prawo dostępu do `resource_id` z URL.
2.  **BFLA:** Nie polegaj na ukrywaniu URLi. Każdy endpoint (zwłaszcza `/admin`) musi sprawdzać rolę użytkownika.
3.  **Rate Limiting:** Wprowadź limity (np. 5 prób na minutę) i blokadę konta po nieudanych logowaniach.
4.  **SSRF:** Waliduj adresy URL podawane przez użytkownika (allow-lista domen) i blokuj ruch do sieci wewnętrznej (localhost, 127.0.0.1, 10.x.x.x).
5.  **JWT:** Używaj silnych, długich kluczy kryptograficznych i nigdy nie udostępniaj ich w kodzie źródłowym.

---
*Tutorial stworzony w celach edukacyjnych. Testuj tylko na własnym środowisku lokalnym!*
