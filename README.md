# Kompletny Przewodnik Hacking API: crAPI (Completely Ridiculous API)
Ten tutorial przeprowadzi Cię przez proces instalacji środowiska testowego crAPI, konfigurację narzędzia Burp Suite oraz wykonanie 5 kluczowych ataków na API (z listy OWASP API Top 10).
---
[gitHub crAPI](https://github.com/OWASP/crAPI/tree/develop)
## Część 1: Przygotowanie Środowiska (Windows)

Zanim zaczniesz testy, musisz uruchomić aplikację i skonfigurować proxy.

### 1. Instalacja crAPI
crAPI działa na Dockerze. Upewnij się, że masz zainstalowanego Dockera Desktop i działa on w tle.

1.  **Pobierz i uruchom aplikację:**
    Otwórz terminal (PowerShell lub CMD) i wykonaj poniższe komendy:
    ```bash
    curl.exe -o crapi.zip -L https://github.com/OWASP/crAPI/archive/refs/heads/main.zip

    tar -xf .\crapi.zip

    cd crAPI-main\deploy\docker

    docker-compose pull

    docker-compose -f docker-compose.yml --compatibility up -d
    ```
    *Wskazówka: Uruchomienie wszystkich serwisów (Identity, Workshop, MailHog) może potrwać około 2-3 minut.*

2.  **Rozwiązywanie problemów:**
    Jeśli aplikacja nie ładuje się lub zwraca błędy 500:
    ```bash
    docker-compose restart
    ```

* **Dostęp do aplikacji:** `http://localhost:8888`
* **Dostęp do maili (MailHog):** `http://localhost:8025`

### 2. Konfiguracja Burp Suite
Burp to proxy, które pozwala przechwytywać i modyfikować ruch między przeglądarką a serwerem.

1.  **Pobierz:** Burp Suite Community Edition.
2.  **Konfiguracja:** Zaleca się używanie wbudowanej przeglądarki Burpa (zakładka *Proxy -> Open Browser*), która posiada już skonfigurowane certyfikaty CA.

---

## Część 2: Ataki na API

### 1. BOLA (Broken Object Level Authorization)
**Cel:** Uzyskanie precyzyjnej lokalizacji pojazdu innego użytkownika.
**Wyjaśnienie:** Serwer nie weryfikuje, czy ID obiektu w URL należy do zalogowanego użytkownika.

1.  Zarejestruj konto i zaloguj się.
2.  W dashboardzie kliknij "Refresh Location" (lub dodaj pojazd, jeśli go nie masz).
3.  Przejdź do zakładki **Community**. Znajdź posty innych użytkowników i spisz `vehicle_id` (ID pojazdu) ofiary (widoczne w odpowiedziach JSON w Burpie lub w treści strony).
4.  Wróć do Dashboardu. Włącz w Burpie **Intercept is On**.
5.  Kliknij ponownie "Refresh Location".
6.  Przechwycone zapytanie wyślij do **Repeatera** (`Ctrl + R`).
7.  W URL zapytania (`GET /identity/api/v2/vehicle/{ID}/location`) podmień swoje ID na ID ofiary.
8.  **Wynik:** Otrzymanie statusu `200 OK` wraz ze współrzędnymi GPS oznacza udany atak.

### 2. BFLA (Broken Function Level Authorization)
**Cel:** Usunięcie wideo innego użytkownika (nieautoryzowany dostęp do funkcji administracyjnej).
**Wyjaśnienie:** Serwer nie sprawdza uprawnień (roli) dla ukrytych endpointów administracyjnych.

1.  Wgraj dowolne wideo w zakładce "My Profile".
2.  Kliknij Manage Video -> Change Name i przechwyć to zapytanie w Burpie (`PUT /identity/api/v2/user/videos/{ID}`).
3.  Wyślij je do **Repeatera**.
4.  Zmień metodę HTTP z `PUT` na `DELETE`.
5.  Zmodyfikuj URL, zmieniając segment `user` na `admin`:
    * Stary: `/identity/api/v2/user/videos/6`
    * Nowy: `/identity/api/v2/admin/videos/5`
    *(Celujemy w ID inne niż nasze. W crAPI ID są inkrementowane, więc warto sprawdzić sąsiednie numery).*
6.  Wyślij zapytanie.
7.  **Wynik:** Status `200 OK` lub `204 No Content` oznacza, że zwykły użytkownik usunął zasób, do którego usuwania uprawniony powinien być tylko administrator.

### 3. Omijanie Rate Limitingu (Broken User Authentication)
**Cel:** Zresetowanie hasła innego użytkownika poprzez odgadnięcie kodu OTP (Brute Force).
**Wyjaśnienie:** Endpoint w wersji `v3` nie posiada limitu prób, który został wprowadzony w wersji `v2`.

1.  Na ekranie logowania wybierz "Forgot Password".
2.  Podaj email ofiary (np. `robot001@example.com` - domyślny bot w systemie).
3.  **Setup testowy:** Wejdź na `http://localhost:8025` (MailHog), sprawdź prawdziwy kod OTP (np. `5050`).
4.  Wróć do crAPI, wpisz błędny kod, ale o 20 mniejszy (np. `5030`) i przechwyć zapytanie w Burpie. (Robimy tak, ponieważ w wersji community Intruder działa powoli)
5.  Wyślij do **Intrudera**.
6.  W zakładce **Positions** zaznacz kod OTP (`5030`) i kliknij `Add §`.
7.  W zakładce **Payloads**:
    * Type: **Numbers**
    * From: `5030`, To: `5060`, Step: `1`.
    * Min integer digits: `4`.
8.  Kliknij **Start Attack**.
9.  **Kluczowa modyfikacja:** W pierwszej linii zapytania zmień wersję API z `v2` na `v3`:
    * Zmiana z: `POST /identity/api/auth/v2/check-otp`
    * Na: `POST /identity/api/auth/v3/check-otp`
10. **Wynik:** Większość zapytań zwróci błąd `500` (nieprawidłowy kod), natomiast jedno zwróci `200 OK`. Jest to poprawny kod OTP.

### 4. SSRF (Server Side Request Forgery)
**Cel:** Zmuszenie serwera crAPI do połączenia się z wewnętrzną infrastrukturą (MailHog).
**Wyjaśnienie:** Serwer wykonuje połączenia do adresów URL podanych przez użytkownika bez odpowiedniej walidacji.

1.  Zaloguj się i przejdź do sekcji "Contact Mechanic".
2.  Wypełnij formularz i przechwyć request w Burpie.
    ```json
    {
      ...
      "mechanic_api": "http://mechanic.crapi.io/..."
    }
    ```
3.  Wyślij do Repeatera. Zmień wartość `mechanic_api` na: `http://mailhog:8025/api/v2/messages`.
    *(MailHog to serwis wewnętrzny. Przeglądarka nie ma do niego dostępu po nazwie kontenera, ale backend crAPI tak).*
4.  Wyślij zapytanie.
5.  **Wynik:** W odpowiedzi (Response) otrzymasz obiekt JSON zawierający treść wszystkich e-maili w systemie (w tym kody OTP innych użytkowników).

### 5. JWT Forging
**Cel:** Przejęcie konta innego użytkownika poprzez podmianę adresu e-mail w tokenie.
**Wyjaśnienie:** Serwer identyfikuje użytkownika na podstawie pola `email` w tokenie JWT. Znając słaby sekret podpisujący tokeny, możemy wygenerować własny token dla dowolnego adresu e-mail (np. ofiary znalezionej wcześniej w systemie).

1.  Zaloguj się na swoje konto i przechwyć dowolne zapytanie w Burpie, aby skopiować swój token JWT (z nagłówka `Authorization`).
2.  Wejdź na stronę [jwt.io](https://jwt.io).
3.  Wklej swój token w lewe pole w JWT Decoder.
4.  **Edycja Payloadu - przejdź do sekcji JWT Encoder:**
    * Znajdź w sekcji **PAYLOAD** pole `email` (lub `sub`).
    * Zmień swój adres e-mail na adres ofiary (np. `robot001@example.com` lub mail innego użytkownika zdobyty w poprzednich zadaniach).
5.  Skopiuj **nowy token** z lewej strony (Encoded).
6.  Wróć do Burpa i wyślij zapytanie o dane użytkownika (np. `GET /identity/api/v2/user/dashboard`) podmieniając token w nagłówku na ten sfałszowany.
7.  **Wynik:** Serwer zwróci dane profilowe, historię zamówień lub lokalizację należącą do ofiary, której adres e-mail wpisałeś w tokenie.

---

## Jak się bronić?

1.  **BOLA:** Nie ufaj ID przesyłanemu przez klienta. Zawsze weryfikuj po stronie serwera, czy zalogowany użytkownik jest właścicielem zasobu.
2.  **BFLA:** Stosuj ścisłą kontrolę ról (RBAC) na każdym endpoincie. Ukrywanie URLi (`/admin`) nie jest zabezpieczeniem.
3.  **Rate Limiting:** Wprowadź limity zapytań (np. 5 prób na minutę) dla wszystkich wersji API i blokuj IP po przekroczeniu progu.
4.  **SSRF:** Waliduj adresy URL podawane przez użytkownika (stosuj białą listę dozwolonych domen). Blokuj ruch do sieci wewnętrznej (localhost, 10.x.x.x, nazwy kontenerów).
5.  **JWT:** Używaj silnych, losowych kluczy kryptograficznych (min. 256 bitów) i przechowuj je w bezpiecznym miejscu (Secrets Management), nigdy w kodzie źródłowym.

---
*Tutorial stworzony w celach edukacyjnych. Testuj wyłącznie na własnym środowisku lokalnym.*
