---
name: chomikuj
description: Używaj gdy użytkownik chce napisać skrypt/program do logowania na chomikuj.pl, wyszukiwania, uploadu, downloadu i usuwania plików, pobierania listy folderów i plików, tworzenia i usuwania folderów, a także w każdym innym przypadku gdy użytkownik chce skomunikować się z chomikuj.pl w oparciu o SOAP API lub WEB HTTP scraping.
author: Maciej Bimek
---

# Chomikuj.pl / ChomikBox SOAP API WIKI

Niniejsze wiki stanowi szczegółowy, algorytmiczny przewodnik po nieoficjalnym interfejsie SOAP API serwisu chomikuj.pl (bazującym na komunikacji oficjalnej aplikacji desktopowej **ChomikBox**). Umożliwia ono zaimplementowanie klienta w dowolnym języku programowania (np. Python, C#, Java, Go, PHP, JavaScript).

## Spis Treści
* [1. Informacje Ogólne i Protokół SOAP](#1-informacje-ogólne-i-protokół-soap)
  * [Wykaz Wszystkich Metod API (SOAP Actions)](#wykaz-wszystkich-metod-api-soap-actions)
* [2. Logowanie i Autoryzacja (Auth)](#2-logowanie-i-autoryzacja-auth)
* [3. Listowanie Folderów, Podfolderów i Plików](#3-listowanie-folderów-podfolderów-i-plików)
  * [A. Listowanie Struktury Folderów (Folders)](#a-listowanie-struktury-folderów-folders)
  * [B. Listowanie Plików w Folderze (Download)](#b-listowanie-plików-w-folderze-download)
  * [C. Tworzenie Folderu (AddFolder)](#c-tworzenie-folderu-addfolder)
  * [D. Usuwanie Folderu (RemoveFolder)](#d-usuwanie-folderu-removefolder)
  * [E. Usuwanie Pliku (DeleteFileAction - Web Action)](#e-usuwanie-pliku-deletefileaction---web-action)
* [4. Generowanie URL do Pobrania Pliku](#4-generowanie-url-do-pobrania-pliku)
* [5. Wyszukiwanie Pliku](#5-wyszukiwanie-pliku)
  * [Metoda A: Wyszukiwanie lokalne (na koncie użytkownika)](#metoda-a-wyszukiwanie-lokalne-na-koncie-użytkownika)
  * [Metoda B: Wyszukiwanie Web/HTTP (Scraping i formularze wyszukiwania)](#metoda-b-wyszukiwanie-webhttp-scraping-i-formularze-wyszukiwania)
* [6. Przesyłanie Pliku (Upload)](#6-przesyłanie-pliku-upload)
  * [A. Pobieranie Tokenu Uploadu (UploadToken)](#a-pobieranie-tokenu-uploadu-uploadtoken)
  * [B. Przesyłanie Danych Pliku (HTTP POST Multipart/Mixed)](#b-przesyłanie-danych-pliku-http-post-multipartmixed)
  * [C. Wznawianie Przerwanego Uploadu (Resume)](#c-wznawianie-przerwanego-uploadu-resume)
* [7. Monitorowanie Stanu Aplikacji (CheckEvents)](#7-monitorowanie-stanu-aplikacji-checkevents)

---

## 1. Informacje Ogólne i Protokół SOAP

Komunikacja z API Chomikuj.pl opiera się na protokole SOAP 1.1 przesyłanym za pomocą zapytań HTTP POST.

* **Adres Endpointu:** `http://box.chomikuj.pl/services/ChomikBoxService.svc`
* **Port:** `80`
* **Nagłówek Content-Type:** `text/xml;charset=utf-8`
* **Przestrzeń nazw SOAP (Method Namespace):** `http://chomikuj.pl/`
* **Wspólne nagłówki HTTP dla każdego zapytania SOAP:**
  ```http
  POST /services/ChomikBoxService.svc HTTP/1.1
  Host: box.chomikuj.pl
  Content-Type: text/xml;charset=utf-8
  SOAPAction: http://chomikuj.pl/IChomikBoxService/[NazwaMetody]
  Content-Length: [Długość_Body]
  Connection: Keep-Alive
  ```
* **Struktura koperty SOAP (Wysyłanej):**
  Każde zapytanie must być opakowane w standardową kopertę SOAP:
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
    <s:Body>
      <[NazwaMetody] xmlns="http://chomikuj.pl/">
        <!-- Parametry metody -->
      </[NazwaMetody]>
    </s:Body>
  </s:Envelope>
  ```

---

## Wykaz Wszystkich Metod API (SOAP Actions)

Wszystkie operacje SOAP klienta ChomikBox obsługiwane są przez kontrakt usługi `IChomikBoxService`. Poniżej znajduje się kompletny wykaz i krótkie podsumowanie przeznaczenia wszystkich dostępnych metod w tym interfejsie API:

1. **`Auth`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/Auth`)
   - **Opis:** Uwierzytelnianie użytkownika na podstawie nazwy konta i skrótu hasła (MD5). Zwraca identyfikator sesji (`token`) oraz identyfikator konta (`hamsterId`).
2. **`CheckEvents`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/CheckEvents`)
   - **Opis:** Odpytywanie serwera w celu przekazania bieżących statusów transferów (isUploading, isDownloading) oraz podtrzymania ważności sesji. Bez wywołania tej metody serwer odmawia poprawnego przejścia przez synchronizację webową (log_www).
3. **`Folders`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/Folders`)
   - **Opis:** Pobieranie struktury drzewa katalogów użytkownika (począwszy od folderu o ID `0`).
4. **`Download`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/Download`)
   - **Opis:** Wielofunkcyjna metoda służąca do:
     - Pobrania listy plików znajdujących się w danym folderze (gdy identyfikator to `[chomikbox_url_id]/[folder_id]`, a disposition to `download`).
     - Wygenerowania bezpośredniego biletu/URL do pobrania pliku z konta własnego lub innego użytkownika (gdy identyfikator to `[file_id]`).
5. **`UploadToken`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/UploadToken`)
   - **Opis:** Rejestracja żądania wysłania nowego pliku. Zwraca unikalny klucz uploadu (`key`), znacznik czasu (`stamp`) oraz adres serwera fizycznego, na który należy bezpośrednio wysłać dane binarne.
6. **`AddFolder`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/AddFolder`)
   - **Opis:** Tworzenie nowego podfolderu w wybranym katalogu nadrzędnym.
7. **`RemoveFolder`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/RemoveFolder`)
   - **Opis:** Usuwanie wybranego folderu wraz z jego zawartością z konta użytkownika.

---

## 2. Logowanie i Autoryzacja (Auth)

Logowanie odbywa się dwuetapowo: najpierw następuje autoryzacja przez interfejs SOAP, która zwraca identyfikator sesji (token), a następnie wykonywana jest synchronizacja z sesją webową (opcjonalna, wymagana do generowania list plików i niektórych uprawnień).

> [!NOTE]
> **Ważność i Podtrzymywanie Sesji (Session Keep-Alive):** 
> Ponowne wywoływanie metody `Auth` w celu odświeżenia sesji jest niewskazane, ponieważ generuje nowy token sesyjny, co unieważnia dotychczasową synchronizację z sesją webową (log_www). Zamiast tego, w celu ewentualnego podtrzymania sesji w tle, należy stosować okresowe wywołanie metody `CheckEvents`, która działa jako mechanizm "sliding expiration" (przesuwania czasu wygaśnięcia sesji) bez zmiany tokenu.

### Algorytm:
1. Pobierz hasło użytkownika w postaci czystego tekstu.
2. Oblicz skrót hasła przy użyciu algorytmu **MD5** i przekonwertuj go do postaci heksadecymalnej (zapisanej małymi literami).
3. Przygotuj strukturę XML dla metody `Auth` z nagłówkiem `SOAPAction: http://chomikuj.pl/IChomikBoxService/Auth`.
4. Wyślij zapytanie POST na endpoint usługi.
5. Odczytaj odpowiedź i sprawdź pole `<status>` (lub `<a:status>`). Jeśli ma wartość `Ok`, wyodrębnij token sesji (`<a:token>`) oraz identyfikator chomika (`<a:hamsterId>`).

### Struktura zapytania XML (Auth):
```xml
<Auth xmlns="http://chomikuj.pl/">
  <name>[NazwaUżytkownika]</name>
  <passHash>[MD5_Hash_Hasła]</passHash>
  <ver>4</ver><!-- Wersja protokołu klienta -->
  <client>
    <name>chomikbox</name>
    <version>2.0.8.2</version><!-- Wersja aplikacji ChomikBox -->
  </client>
</Auth>
```

### Struktura odpowiedzi XML (AuthResponse):
```xml
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <AuthResponse xmlns="http://chomikuj.pl/">
      <AuthResult xmlns:a="http://chomikuj.pl" xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
        <a:status>Ok</a:status>
        <a:errorMessage i:nil="true"/>
        <a:hamsterId>[Id_Chomika]</a:hamsterId>
        <a:publisherId i:nil="true"/>
        <a:name>[NazwaUżytkownika]</a:name>
        <a:token>[Token_Sesji_UUID]</a:token>
      </AuthResult>
    </AuthResponse>
  </s:Body>
</s:Envelope>
```

### Synchronizacja z Sesją Webową (log_www) - Wymagana do listowania plików:
Aby wylistować pliki w folderach za pomocą metody SOAP `Download`, konieczne jest uzyskanie identyfikatora `chomikbox_url_id`. Uzyskuje się go poprzez synchronizację sesji SOAP z sesją przeglądarki (HTTP Cookie Jar):

> [!IMPORTANT]
> **Zależność od CheckEvents (CheckEvents Dependency):**
> Przed wykonaniem synchronizacji z sesją webową (krok 1 poniżej), należy bezwzględnie wywołać SOAP-ową metodę `CheckEvents` (patrz sekcja 7) co najmniej raz. Wywołanie to rejestruje klienta SOAP jako aktywny proces ChomikBoxa. Jeśli pominiesz ten krok, zapytanie do `/action/chomikbox/DownloadFolderChomikBox` (krok 4) zwróci okno modalne HTML informujące, że aplikacja ChomikBox nie jest uruchomiona: `"Aby pobrać folder jednym kliknięciem, musisz mieć uruchomiony program ChomikBox"`, zamiast właściwego linku `chomik://`.

1. Wyślij zapytanie GET na: `http://chomikuj.pl/chomik/chomikbox/LoginFromBox?t=[Token_Sesji]&returnUrl=/ChomikBox` i zapisz otrzymane ciasteczka.
2. Z treści odpowiedzi wyodrębnij token weryfikacyjny CSRF: `<input name="__RequestVerificationToken" ... value="([Verification_Token])" />`.
3. Wyślij zapytanie POST na: `http://chomikuj.pl/action/Login/TopBarLogin` z ciałem zakodowanym jako `application/x-www-form-urlencoded`:
   `ReturnUrl=&Login=[NazwaUżytkownika]&rememberLogin=true&Password=[Hasło]&__RequestVerificationToken=[Verification_Token]`
4. Wyślij zapytanie POST na: `http://chomikuj.pl/action/chomikbox/DownloadFolderChomikBox` z parametrami:
   `chomikName=[NazwaUżytkownika]&folderId=0&__RequestVerificationToken=[Verification_Token]`
5. Z odpowiedzi wyodrębnij identyfikator URL za pomocą wyrażenia regularnego: `chomik://files/(?::)?(\d+)/?` (w zależności od wersji serwisu link może mieć postać `chomik://files/:[id]/` lub `chomik://files/[id]/0`). Otrzymaną liczbę oznaczamy jako `chomikbox_url_id` (używaną później przy listowaniu plików).

---

## 3. Listowanie Folderów, Podfolderów i Plików

### A. Listowanie Struktury Folderów (Folders)
Pobranie pełnego drzewa katalogów użytkownika odbywa się za pomocą metody `Folders`.

#### Algorytm:
1. Przygotuj zapytanie dla metody `Folders` z nagłówkiem `SOAPAction: http://chomikuj.pl/IChomikBoxService/Folders`.
2. Parametr `folderId` ustaw na `0`, aby pobrać strukturę od katalogu głównego.
3. Wyślij zapytanie POST.
4. Odpowiedź zawiera strukturę zagnieżdżonych elementów `<FolderInfo>`. Każdy folder posiada:
   - `<id>`: numeryczny identyfikator folderu.
   - `<name>`: nazwę katalogu.
   - `<folders>`: listę podfolderów (zagnieżdżone obiekty `<FolderInfo>`).

#### Struktura zapytania XML (Folders):
```xml
<Folders xmlns="http://chomikuj.pl/">
  <token>[Token_Sesji]</token>
  <hamsterId>[Id_Chomika]</hamsterId>
  <folderId>0</folderId><!-- 0 oznacza katalog główny -->
  <depth>0</depth><!-- Głębokość listowania (0 oznacza pobranie całej struktury) -->
</Folders>
```

---

### B. Listowanie Plików w Folderze (Download)
Listowanie plików odbywa się za pomocą metody `Download`, w której parametr `disposition` ma wartość `download`.

#### Algorytm:
1. Przygotuj zapytanie dla metody `Download` z nagłówkiem `SOAPAction: http://chomikuj.pl/IChomikBoxService/Download`.
2. Wygeneruj identyfikator zapytania `id` w postaci: `[chomikbox_url_id]/[folder_id]`, gdzie `chomikbox_url_id` został pobrany podczas synchronizacji sesji webowej (log_www), a `folder_id` to identyfikator interesującego nas folderu.
3. Parametr `<disposition>` ustaw na `download`.
4. Ustaw parametr `<agreementInfo>` -> `<AgreementInfo>` -> `<name>` na `own`.
5. Wyślij zapytanie POST.
6. Odbierz odpowiedź XML. W sekcji `<files>` wyodrębnij wszystkie elementy `<FileEntry>`. Każdy plik ma pola:
   - `<name>`: nazwa pliku.
   - `<url>`: bezpośredni identyfikator/url pobierania.

> [!NOTE]
> **Obsługa Pustych Folderów (Empty Folder Behavior):**
> Jeżeli zapytywany folder jest pusty (nie zawiera żadnych plików), serwer SOAP w odpowiedzi na metodę `Download` zwróci błąd z polem `<a:status>Error</a:status>` oraz komunikatem `<a:errorMessage>failed : requested file(s) not available</a:errorMessage>`. Aplikacja kliencka powinna obsłużyć tę sytuację i traktować ją jako pomyślne pobranie pustej listy plików (np. zwracając pustą listę), a nie jako błąd krytyczny komunikacji.

#### Struktura zapytania XML (Download do listowania plików):
```xml
<Download xmlns="http://chomikuj.pl/">
  <token>[Token_Sesji]</token>
  <sequence>
    <stamp>0</stamp><!-- Sekwencja/znacznik czasu żądania -->
    <part>0</part><!-- Numer części żądania -->
    <count>1</count><!-- Liczba elementów na liście żądań -->
  </sequence>
  <disposition>download</disposition>
  <list>
    <DownloadReqEntry>
      <id>[chomikbox_url_id]/[folder_id]</id>
      <agreementInfo>
        <AgreementInfo>
          <name>own</name>
        </AgreementInfo>
      </agreementInfo>
    </DownloadReqEntry>
  </list>
</Download>
```

---

### C. Tworzenie Folderu (AddFolder)
Tworzenie nowych katalogów (folderów) na koncie użytkownika odbywa się za pomocą metody `AddFolder`.

#### Algorytm:
1. Przygotuj zapytanie SOAP dla metody `AddFolder` (`SOAPAction: http://chomikuj.pl/IChomikBoxService/AddFolder`).
2. Podaj `token` (aktywny identyfikator sesji).
3. Podaj `newFolderId` jako identyfikator katalogu nadrzędnego (np. `0` dla katalogu głównego).
4. Podaj `name` jako nazwę nowo tworzonego katalogu.
5. Wyślij zapytanie POST.
6. Odczytaj odpowiedź XML:
   - Jeśli operacja się powiodła, element `<status>` zwróci wartość `Ok`, a element `<a:folderId>` (przestrzeń nazw `http://schemas.datacontract.org/2004/07/TeamSolutions.Chomik.DAO.ChomikBox`) będzie zawierał numeryczny identyfikator nowo utworzonego folderu.
   - Jeśli folder o tej samej nazwie już istnieje w wybranej lokalizacji, element `<status>` zwróci wartość `Error`, a w `<errorMessage>` otrzymasz komunikat błędu `NameExistsAtDestination`.

#### Struktura zapytania XML (AddFolder):
```xml
<AddFolder xmlns="http://chomikuj.pl/">
  <token>[Token_Sesji]</token>
  <newFolderId>[Id_Katalogu_Nadrzędnego]</newFolderId>
  <name>[Nazwa_Nowego_Folderu]</name>
</AddFolder>
```

#### Struktura odpowiedzi XML (AddFolderResponse):
```xml
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <AddFolderResponse xmlns="http://chomikuj.pl/">
      <AddFolderResult xmlns:a="http://schemas.datacontract.org/2004/07/TeamSolutions.Chomik.DAO.ChomikBox" xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
        <status xmlns="http://chomikuj.pl">Ok</status>
        <errorMessage i:nil="true" xmlns="http://chomikuj.pl"/>
        <a:folderId>[Id_Utworzonego_Folderu]</a:folderId>
      </AddFolderResult>
    </AddFolderResponse>
  </s:Body>
</s:Envelope>
```

---

### D. Usuwanie Folderu (RemoveFolder)
Usuwanie istniejących katalogów z konta użytkownika odbywa się za pomocą metody `RemoveFolder`.

#### Algorytm:
1. Przygotuj zapytanie SOAP dla metody `RemoveFolder` (`SOAPAction: http://chomikuj.pl/IChomikBoxService/RemoveFolder`).
2. Podaj `folderId` usuwanego folderu.
3. Parametr `force` ustaw na `1` (aby wymusić usunięcie folderu wraz z całą zawartością).
4. Wyślij zapytanie POST.
5. Odczytaj odpowiedź i sprawdź pole `<status>` (lub `<a:status>`). Pomyślne usunięcie zwraca status `Ok`.

#### Struktura zapytania XML (RemoveFolder):
```xml
<RemoveFolder xmlns="http://chomikuj.pl/">
  <token>[Token_Sesji]</token>
  <folderId>[Id_Usuwanego_Folderu]</folderId>
  <force>1</force><!-- Wymuszenie usunięcia wraz z zawartością (1 = tak, 0 = nie) -->
</RemoveFolder>
```

---

### E. Usuwanie Pliku (DeleteFileAction - Web Action)
Ponieważ kontrakt SOAP usługi `IChomikBoxService` **nie zawiera** dedykowanej metody usuwania pojedynczych plików (np. metoda SOAP `RemoveFile` nie istnieje), usuwanie plików w oficjalnej aplikacji ChomikBox realizowane jest za pomocą standardowego żądania HTTP POST do endpointu webowego.

#### Wymagania:
- Skuteczna autoryzacja sesji webowej (ciasteczka uzyskane za pomocą metody `LoginFromBox` opisanej w sekcji 2).

#### Adres Endpointu:
`http://chomikuj.pl/action/FileDetails/DeleteFileAction` (obsługiwany również przez HTTPS)

#### Metoda HTTP:
`POST`

#### Nagłówki HTTP:
```http
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
```

#### Parametry POST (Form Data):
- `ChomikName`: Nazwa konta użytkownika (właściciela pliku).
- `FolderId`: Numeryczny identyfikator katalogu nadrzędnego (np. `0` dla katalogu głównego).
- `FileId`: Numeryczny identyfikator pliku, który ma zostać usunięty.
- `FolderTo`: Zawsze wartość `0` (reprezentuje kosz / akcję kasowania).

#### Przykładowy format ciała żądania:
`ChomikName=[Id_Chomika]&FolderId=0&FileId=1234567890&FolderTo=0`

#### Odpowiedź JSON (Sukces):
Pomyślna operacja zwraca odpowiedź typu JSON (Status `200 OK`):
```json
{
  "Type": "Growl",
  "Title": "",
  "Content": "Plik został usunięty",
  "refreshTopBar": false,
  "IsSuccess": true,
  "Data": {
    "Status": "OK"
  },
  "ContainsCaptcha": false,
  "trackingCodeJS": null
}
```

#### Odpowiedź (Błąd):
Jeśli podany `FileId` nie istnieje w podanej lokalizacji, serwer zwraca błąd:
`HTTP Error 500: Internal Server Error`

---

## 4. Generowanie URL do Pobrania Pliku

Aby pobrać plik, należy wygenerować jednorazowy token/URL pobierania za pomocą metody `Download` o innej konfiguracji parametrów.

### Algorytm:
1. Przygotuj zapytanie SOAP dla metody `Download` (`SOAPAction: http://chomikuj.pl/IChomikBoxService/Download`).
2. Parametr `stamp` w sekcji `sequence` ustaw na wartość np. `2` (lub inną liczbę całkowitą oznaczającą sekwencję).
3. Podaj identyfikator pliku `file_id` (pobrany z listowania plików lub bezpośrednio z witryny) jako element `<id>`.
4. Jeżeli pobierasz plik z własnego konta, ustaw `<name>` w `agreementInfo` na `own`. Jeżeli plik pochodzi z cudzego konta, ustaw wartość na `access`.
5. Wyślij zapytanie POST.
6. Odczytaj odpowiedź XML. W polu `<url>` (lub zagnieżdżonym w strukturze odpowiedzi) otrzymasz bezpośredni link HTTP, pod którym plik jest dostępny do pobrania.

### Struktura zapytania XML (Generowanie URL):
```xml
<Download xmlns="http://chomikuj.pl/">
  <token>[Token_Sesji]</token>
  <sequence>
    <stamp>2</stamp><!-- Numer sekwencji żądania -->
    <part>0</part><!-- Numer części żądania -->
    <count>1</count><!-- Liczba elementów na liście żądań -->
  </sequence>
  <disposition>download</disposition>
  <list>
    <DownloadReqEntry>
      <id>[Id_Pliku]</id>
      <agreementInfo>
        <AgreementInfo>
          <name>own</name>
        </AgreementInfo>
      </agreementInfo>
    </DownloadReqEntry>
  </list>
</Download>
```
> [!NOTE]
> **Stabilność Strumienia Pobierania (Download Stream Stability):**
> Bezpośredni link HTTP pobierania uzyskany z pola `<url>` jest całkowicie niezależny od stanu sesji SOAP po nawiązaniu połączenia. Serwer pobierania (`s*.chomikuj.pl`) utrzymuje stabilne i nieprzerwane połączenie aż do pobrania pełnego strumienia danych.

---

## 5. Wyszukiwanie Pliku

### Ograniczenia protokołu SOAP:
Wielką wadą i specyfiką usługi SOAP ChomikBox jest **brak bezpośredniej metody wyszukiwania plików (zarówno globalnie, jak i lokalnie)**. Wyszukiwanie musi być zaimplementowane na poziomie aplikacji klienckiej przy użyciu jednej z dwóch poniższych strategii:

### Metoda A: Wyszukiwanie lokalne (na koncie użytkownika)
Realizowane przez aplikację kliencką poprzez rekurencyjne przeszukiwanie struktury drzewa katalogów pobranego przez SOAP.

#### Algorytm:
1. Pobierz pełne drzewo katalogów użytkownika wywołując metodę `Folders` (katalog główny `0`).
2. Przejdź rekurencyjnie (np. algorytmem DFS lub BFS) przez wszystkie foldery.
3. Dla każdego folderu pobierz listę plików za pomocą metody `Download` (jak w sekcji 3B).
4. Przeprowadź lokalne dopasowanie nazwy szukanego pliku do nazw plików znajdujących się na listach.

### Metoda B: Wyszukiwanie Web/HTTP (Scraping i formularze wyszukiwania)
Ponieważ SOAP nie oferuje bezpośrednich metod wyszukiwania plików (ani globalnie, ani lokalnie), wyszukiwanie w serwisie realizowane jest za pomocą zapytań HTTP GET/POST na tradycyjne endpointy webowe. Analiza formularzy na zalogowanym profilu użytkownika (`chomikuj.pl/[nazwa_użytkownika]`) ujawnia trzy odrębne mechanizmy wyszukiwania:

#### 1. Globalne wyszukiwanie kont / chomików (HTTP GET)
Służy do wyszukiwania kont użytkowników (chomików) w serwisie.
* **URL:** `http://chomikuj.pl/action/Search`
* **Metoda:** `GET`
* **Parametry (Query String):**
  - `Query`: Szukana fraza (np. `Ziemia`).
* **Uwaga:** Adres `Search.aspx` jest przestarzały i nie zwraca wyników wyszukiwania (zwraca stronę główną portalu zarówno dla użytkowników zalogowanych, jak i niezalogowanych). Adres `/action/Search/SearchFiles` również nie istnieje (404).

#### 2. Globalne wyszukiwanie plików według typu (HTTP POST)
Umożliwia przeszukiwanie serwisu pod kątem konkretnych formatów plików.
* **URL:** `http://chomikuj.pl/action/SearchFiles`
* **Metoda:** `POST`
* **Parametry (Form Data / URL-encoded):**
  - `FileName`: Szukana fraza (np. `Ziemia`).
  - `FileType`: Filtr typu pliku. Dopuszczalne wartości:
    * `all` (wszystkie)
    * `video` (pliki wideo)
    * `image` (obrazy)
    * `music` (pliki muzyczne)
    * `document` (dokumenty)
    * `archive` (archiwa)
    * `application` (programy)

#### 3. Wyszukiwanie lokalne na wybranym koncie (HTTP POST)
Umożliwia zawężenie wyszukiwania plików do konta jednego, określonego użytkownika.
* **URL:** `http://chomikuj.pl/action/SearchFiles`
* **Metoda:** `POST`
* **Parametry (Form Data / URL-encoded):**
  - `FileName`: Szukana fraza.
  - `SearchOnAccount`: Wartość `true` (wymusza wyszukiwanie na wskazanym koncie).
  - `TargetAccountName`: Nazwa konta chomika, które przeszukujemy.

---

## 6. Przesyłanie Pliku (Upload)

Upload plików w Chomikuj.pl jest procesem dwuetapowym: najpierw uzyskuje się token autoryzacyjny uploadu dla danego pliku (poprzez SOAP), a następnie plik przesyłany jest strumieniowo bezpośrednio na dedykowany serwer uploadu (poprzez zapytanie HTTP POST typu Multipart).

### A. Pobieranie Tokenu Uploadu (UploadToken)
Przed wysłaniem pliku należy zapytać serwer główny o to, na który serwer fizyczny należy go wysłać oraz uzyskać klucz sesji uploadu (`key`) i znacznik czasu (`stamp`).

#### Algorytm:
1. Przygotuj zapytanie dla metody `UploadToken` (`SOAPAction: http://chomikuj.pl/IChomikBoxService/UploadToken`).
2. Podaj `folderId` (gdzie zapisać plik) oraz `fileName` (nazwa tworzonego pliku).
3. Wyślij zapytanie POST.
4. Odczytaj odpowiedź i wyodrębnij:
   - `key`: token uwierzytelniający upload.
   - `stamp`: znacznik czasu używany jako element boundary.
   - `server`: adres serwera uploadu (np. `upload15.chomikuj.pl:80`). Wyodrębnij z tego ciąg znaków hosta oraz port.

#### Struktura zapytania XML (UploadToken):
```xml
<UploadToken xmlns="http://chomikuj.pl/">
  <token>[Token_Sesji]</token>
  <folderId>[Id_Folderu_Docelowego]</folderId>
  <fileName>[Nazwa_Pliku]</fileName>
</UploadToken>
```

#### Struktura odpowiedzi XML (UploadTokenResponse):
```xml
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <UploadTokenResponse xmlns="http://chomikuj.pl/">
      <UploadTokenResult xmlns:a="http://chomikuj.pl" xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
        <a:status>Ok</a:status>
        <a:errorMessage i:nil="true"/>
        <a:key>[Klucz_Uploadu]</a:key>
        <a:locale>PL</a:locale>
        <a:server>[Serwer_Uploadu_Host:Port]</a:server>
        <a:stamp>[Stamp_Uploadu]</a:stamp>
      </UploadTokenResult>
    </UploadTokenResponse>
  </s:Body>
</s:Envelope>
```

---

### B. Przesyłanie Danych Pliku (HTTP POST Multipart/Mixed)
Właściwy plik wysyłany jest bezpośrednio na serwer otrzymany w odpowiedzi `UploadToken` (np. `upload15.chomikuj.pl:80`). Zapytanie to jest specyficznym połączeniem POST HTTP z formatem Multipart.

#### Algorytm:
1. Nawiąż bezpośrednie połączenie TCP na wskazany serwer i port.
2. Zdefiniuj unikalny separator boundary w formacie: `--!CHB[stamp]`, gdzie `[stamp]` to znacznik otrzymany w kroku A.
3. Skonstruuj nagłówek zapytania POST:
   ```http
   POST /file/ HTTP/1.0
   Content-Type: multipart/mixed; boundary=!CHB[stamp]
   Host: [serwer_uploadu]
   Content-Length: [całkowita_długość_ciała_multipart]
   ```
4. Ciało zapytania składa się z sekcji Multipart opisujących parametry sesji oraz strumienia pliku binarnego. Kolejne sekcje to:
   - `chomik_id` (Identyfikator użytkownika)
   - `folder_id` (Folder docelowy)
   - `key` (Klucz uploadu z kroku A)
   - `time` (Znacznik stamp z kroku A)
   - `client` (`ChomikBox-2.0.8.2`)
   - `locale` (`PL`)
   - `file` (Sekcja binarna pliku)
5. Wyślij nagłówek, a następnie przesyłaj plik w kawałkach (np. bufor 1024 bajtów) monitorując postęp.
6. Na końcu wyślij sekwencję zamykającą boundary: `\r\n--!CHB[stamp]--\r\n\r\n`.
7. Odbierz odpowiedź serwera. Pomyślny upload zwraca strukturę XML zawierającą: `<resp res="1" fileid="[id_nowego_pliku]"/>`.

#### Przykładowy format ciała Multipart (z danymi binarnymi):
```text
--!CHB[stamp]
name="chomik_id"
Content-Type: text/plain

[Id_Chomika]
--!CHB[stamp]
name="folder_id"
Content-Type: text/plain

[Id_Folderu]
--!CHB[stamp]
name="key"
Content-Type: text/plain

[Klucz_Uploadu]
--!CHB[stamp]
name="time"
Content-Type: text/plain

[Stamp_Uploadu]
--!CHB[stamp]
name="client"
Content-Type: text/plain

ChomikBox-2.0.8.2
--!CHB[stamp]
name="locale"
Content-Type: text/plain

PL
--!CHB[stamp]
name="file"; filename="[Nazwa_Pliku]"

[DANE_BINARNE_PLIKU]
--!CHB[stamp]--
```

---

### C. Wznawianie Przerwanego Uploadu (Resume)
Protokół uploadu Chomikuj.pl pozwala na wznowienie wysyłania pliku, jeśli połączenie zostało przerwane w trakcie.

#### Algorytm:
1. Przed ponownym wysłaniem, wyślij zapytanie sprawdzające stan na serwer uploadu za pomocą metody GET:
```http
   GET /resume/check/?key=[Klucz_Uploadu]& HTTP/1.1
   Host: [serwer_uploadu]
   User-Agent: ChomikBox
```
2. Odbierz odpowiedź serwera. Jeśli plik był już częściowo przesłany, serwer zwróci XML z rozmiarem odebranych danych:
```xml
   <!-- skipThumbnails: 0 = nie pomijaj, res: 1 = sukces statusu -->
   <resp file_size="[rozmiar_w_bajtach_na_serwerze]" skipThumbnails="0" res="1"/>
```
3. Jeśli serwer zwróci rozmiar większy niż zero (np. `filesize_sent`):
   - Otwórz plik lokalny i przesuń wskaźnik czytania (seek) do pozycji `filesize_sent`.
   - Przygotuj zapytanie POST HTTP multipart/mixed do `/file/` identyczne z pierwotnym, ale **dodaj nową sekcję** w Multipart przed sekcją pliku:
     ```text
     --!CHB[stamp]
     name="resume_from"
     Content-Type: text/plain

     [filesize_sent]
     ```
   - Jako rozmiar pliku w nagłówku i multipart wyślij pozostałą część pliku: `(rozmiar_całkowity - filesize_sent)`.
   - Prześlij strumieniowo pozostałe bajty pliku, wyślij boundary końcowe i odbierz odpowiedź końcową.

---

## 7. Monitorowanie Stanu Aplikacji (CheckEvents)

Metoda `CheckEvents` służy jako oficjalny mechanizm podtrzymywania sesji SOAP (Keep-Alive). Pozwala na odświeżenie sesji na serwerze bez konieczności ponownego uwierzytelniania (metodą `Auth`), zapobiegając wygaśnięciu tokenu i zachowując ważność synchronizacji z sesją webową.
Klient ChomikBox okresowo odpytuje serwer główny w celu weryfikacji poprawności działania sesji oraz przekazania informacji statystycznych o bieżących transferach za pomocą metody `CheckEvents`.

### Algorytm:
1. Przygotuj zapytanie dla metody `CheckEvents` (`SOAPAction: http://chomikuj.pl/IChomikBoxService/CheckEvents`).
2. W parametrach `stats` przekaż statusy:
   - `isUploading` (np. `0` lub `1` jeśli trwa wysyłanie).
   - `isDownloading` (np. `0` lub `1` jeśli trwa pobieranie).
   - `panelSelectedTab` (identyfikator otwartej karty panelu).
   - `animation` (parametr animacji interfejsu).
3. Wyślij zapytanie POST.
4. Odpowiedź serwera w polu `<status>` (lub `<status><#text>`) powinna zwrócić `Ok`.

### Struktura zapytania XML (CheckEvents):
```xml
<CheckEvents xmlns="http://chomikuj.pl/">
  <token>[Token_Sesji]</token>
  <stats>
    <isUploading>0</isUploading><!-- Status wysyłania: 0 = brak, 1 = w toku -->
    <isDownloading>0</isDownloading><!-- Status pobierania: 0 = brak, 1 = w toku -->
    <panelSelectedTab>0</panelSelectedTab><!-- Indeks otwartej zakładki w ChomikBox (domyślnie 0) -->
    <animation>2</animation><!-- Typ animacji chomika w aplikacji (domyślnie 2) -->
  </stats>
</CheckEvents>
```
