# Dokumentacja biblioteki mobilnej P24v4

1. [Informacje wstępne](#info)
2. [Zgłaszanie problemów](#problems)
3. [Opis systemu](#desc)
4. [Konfiguracja projekt](#config)
5. [Dostępne metody](#methods)

<a name="info"></a>

## 1. Informacje wstępne

Do korzystania z biblioteki mobilnej konieczne jest posiadanie **ważnej umowy i aktywnego konta** w systemie Przelewy24 - szczegółowe informacje o procesie rejestracji można znaleźć na stronie <https://www.przelewy24.pl/rejestracja>, dzwoniąc pod numer telefonu +48 61 642 93 45 bądź wysyłając maila na adres [oferty@przelewy24.pl](mailto:oferty@przelewy24.pl). Formularz rejestracyjny znajduje się na stronie <https://panel.przelewy24.pl/rejestracja.php>. Po poprawnym zarejestrowaniu się na adres email podany podczas rejestracji zostanie przesłany identyfikator użytkownika - `Merchant ID` oraz hasło.

W bibliotece mobilnej część metod wymaga `klucza crc` - znajduje się on w zakładce **Moje dane** w sekcji **Dane do API i konfiguracja** i jest dostępny po zalogowaniu się na stronie <https://panel.przelewy24.pl> danymi otrzymanymi na mail'a po rejestracji. 

<a name="problems"></a>

### 2. Zgłaszanie problemów

W momencie wystąpienia błędu w bibliotece mobilnej P24, należy zgłosić w zakładce [**Issues**](https://github.com/przelewy24/p24-mobile-lib-android/issues) według poniższego schematu:

- **Numer wersji** - Wersja biblioteki w której odnotowano problem
- **Nazwa metody** - Nazwa metody, w której wystąpiły błędy
- **Opis** - Istota problemu
- **Step to reduce** - Kroki, dzięki którym można odwzorować zaistniały problem 
                     

<a name="desc"></a>

## 3. Opis systemu

Biblioteka mobilna Przelewy24 jest natywną biblioteką platformy Android, która umożliwia wykonanie płatności w ramach jednej aplikacji bez konieczności korzystania z przeglądarki lub innej aplikacji. Biblioteka udostępnia zróżnicowane metody płatności: BLIK, przelewy bankowe, karty płatnicze, wirtualne portfele (np. SkyCash, PayPal) i inne (ApplePay).

### 3.1. Przebieg transakcji przy użyciu bibliotek mobilnej

Wywołanie płatności w bibliotece powoduje otwarcie WebView z załadowanym serwisem transakcyjnym Przelewy24. Kolejnym krokiem jest wybór metody płatności bądź wypełnienie danych potrzebnych do płatności np. dane kartowe bądź kod BLIK. W przypadku, gdy użytkownik wybierze jako metodę płatności przelew bankowy, konieczne jest zalogowanie się do wskazanego banku. Po zatwierdzeniu płatności WebView zamyka się, a biblioteka zwraca do aplikacji klienta jeden ze statusów - `płatność udana`, gdy transakcja przebiegnie pomyślnie bądź ze statusem `płatność nieudana`, w momencie gdy użytkownik wprowadzi na przykład nieprawidłowy kod BLIK. Gdy proces zostanie przerwany przez użytkownika na którymś z wymienionych wcześniej etapów biblioteka zamyka się ze statusem `płatność anulowana`. Przebieg procesu został zilustrowany na poniższym diagramie - jak widać biblioteka jest tylko jedną z części procesu, a do poprawnego działania wymaga również backendu po stronie sprzedawcy, który jest odpowiedzialny m.in. za weryfikację transakcji:


![Transaction flow](img/transaction.png?raw=true "Transaction flow")


<a name="config"></a>

## 4. Konfiguracja projektu

Do zastosowania biblioteki mobilnej P24 konieczne jest poprawne skonfigurowanie projektu i dodanie odpowiednich zależności. Pierwszym krokiem jest dodanie paczki z biblioteką przez Swift Package Manager - `File > Swift Packages > Add Package Dependency` znajdującej się pod adresem `https://mobile.przelewy24.pl/spm/p24_sdk_public`. Do korzystania z metod biblioteki, konieczne jest dodanie importu dodanej paczki w odpowiednich klasach:

``` swift
import p24_sdk_public
```

> :warning:   **UWAGA** - `Minimum deployment target` dla projektu, konieczny do wykorzystywania biblioteki to **iOS 13.0**.

<a name="methods"></a>

## 5. Dostępne metody

Przed wywołaniem jakiejkolwiek metody konieczne jest ustawienie sprzedawcy w konfiguracji biblioteki:

```swift
P24Config.merchant = DefaultMerchant()
```

`DefaultMerchant` jest implementacją protokołu `P24Merchant`:

```swift
class DefaultMerchant : P24Merchant {
    var id: Int = 64195
    var crc = [
        Environment.PRODUCTION : "b36147eeac447028",
        Environment.SANDBOX : "d27e4cb580e9bbfe"
    ]
}
```

gdzie `id` jest identyfikatorem sprzedawcy uzyskanym w trakcie procesu rejestracji, a `production_crc` i `sandbox_crc` są kluczami CRC z odpowiednich środowisk.

### 5.1. TrnRequest

Metoda ta umożliwia inicjalizację płatności przyjmując jako parametr wejściowy `environment`, czyli środowisko, na którym została zarejestrowana transakcja (`Environment.SANDBOX` lub `Environment.PRODUCTION`) oraz `token` transakcji uzyskany w wyniku rejestracji transakcji przez backend sprzedawcy - szczegółowe informacje jak zarejestrować transakcję w systemie Przelewy24 można znaleźć [tutaj](https://developers.przelewy24.pl/index.php?pl#tag/Obsluga-transakcji-API).

Opcjonalnie, w celu obsłużenia rezultatu transakcji, do metody można przekazać obiekt handlera implementującego protokół `TrnRequestResultHandler`, posiadający metodę `onResult` zwracającą obiekt typu `TrnRequestResult`:

```swift
func onResult(_ result: TrnRequestResult) {
        switch result {
            case .SUCCESS:
                // perform action when success
            case .ERROR(let errorMessage):
                // perform action when error
            case .CANCEL:
                // perform action when cancel
        }
    }
```
> :warning:   **UWAGA** - Należy pamiętać, że metoda `onSuccess()` informuje o tym, że backend P24 przyjął transakcję do realizacji - informacja o tym, czy do faktycznego obciążenia doszło wysyłana jest na adres URL przekazany w parametrze `p24_url_status`

Przykładowe wywołanie metody TrnRequest:
``` swift
TrnRequest.builder()
    .token("4BF0AEBE17-31456C-18F126-99A146BC94")
    .environment(Environment.SANDBOX)
    .resultHandler(self)
    .build()
    .start()
```

> :warning:   **UWAGA** - Transakcja zostanie uznana przez system P24 za transakcje mobilną, gdy przy rejestracji zostaną przekazane parametry `p24_mobile_lib = 1` oraz `p24_sdk_version` którego wartością jest aktualna wersja biblioteki mobilnej - można ją uzyskać wywołując metodę `P24SdkVersion.value()` - **brak wskazanych parametrów uniemożliwi poprawne zamknięcie biblioteki**.

### 5.2. TrnDirect

Metoda ta umożliwia inicjalizację płatności, dzięki podaniu danych bezpośrednio w aplikacji.

Jako parametry wejściowe przyjmuje parametry rejestracji transakcji uwzględnione w dokumentacji technicznej P24, poszerzone o wybór środowiska - `Environment.PRODUCTION` lub `Environment.SANDBOX`. Parametry `sessionId`, `amountIngGr`, `currency`, `description`, `email` i `country` są parametrami obowiązkowymi. Dodatkowo, sprzedawca w ramach metody ma możliwość nadpisania domyślnego merchanta opisanego na początku rozdziału, nowym merchantem.

Opcjonalnie, w celu obsłużenia rezultatu transakcji, do metody można przekazać obiekt handlera implementującego protokół `TrnDirectResultHandler`, posiadający metodę `onResult` zwracającą obiekt typu `TrnDirectResult`:

```swift
func onResult(_ result: TrnDirectResult) {
        switch result {
            case .SUCCESS:
                // perform action when success
            case .ERROR(let errorMessage):
                // perform action when error
            case .CANCEL:
                // perform action when cancel
        }
    }
```

> :warning:   **UWAGA** - Należy pamiętać, że metoda `onSuccess()` informuje o tym, że transakcja została przyjęta do realizacji - żeby ustalić czy transakcja jest opłacona, po wywołaniu metody `onSuccess()` aplikacja sprzedawcy powinna odpytać własny backend o status trasakcji.

Przykładowe wywołanie metody `TrnDirect` z wypełnionymi wszystkimi dostępnymi polami:
``` swift
TrnDirect.builder()
    .sessionId("test session")
    .amountInGr(1)
    .currency("PLN")
    .description("Mobile Test")
    .email("test@test.pl")
    .country("PL")
    .client("Jan Kowalski")
    .address("Piękna 7")
    .zip("11-111")
    .city("Poznań")
    .phone("777666555")
    .language("pl")
    .method(25)
    .urlStatus("url_for_merchant_backend_verification")
    .timeLimit(0)
    .channel(3)
    .shipping(5)
    .transferLabel("Test tramsfer label")
    .environment(Environment.SANDBOX)
    .resultHandler(self)
    .build()
    .start()
```

Metoda `TrnDirect` umożliwia również implementację funkcjonalności **Pasażu w wersji 2.0**. Konieczne jest do tego przekazanie parametru `PassageCart` w ramach `TrnDirectBuilder`. `PassageCart` jest listą obiektów typu `PassageCartItem`:
```swift
PassageCartItem.builder()
    .name("Test item")
    .description("Test item description")
    .number(1)
    .price(10)
    .quantity(1)
    .targetAmount(1)
    .targetPosId(another_merchant_id)
    .build(),
```

> :warning:   **UWAGA** - Parametry `number` i `description` zgodnie z dokumentacją `Pasażu 2.0` są opcjonalne.

### 5.3. Express

Metoda `Express` umożliwia realizację transakcji w systemie Express. Jako parametr wejściowy przyjmuje `url`, który został uzyskany w wyniku rejestracji transakcji mobilnej w tym systemie.

Do obsługi rezultatu transakcji, należy dodatkowo przekazać obiekt handlera implementujący protokół `ExpressResultHandler`, posiadający metodę `onResult` zwracającą obiekt typu `ExpressResult`:

```swift
func onResult(_ result: ExpressResult) {
        switch result {
            case .SUCCESS:
                // perform action when success
            case .ERROR(let errorMessage):
                // perform action when error
            case .CANCEL:
                // perform action when cancel
        }
    }
```

> :warning:   **UWAGA** - Należy pamiętać, że metoda `onSuccess()` informuje o tym, że backend P24 przyjął transakcję do realizacji - informacja o tym, czy do faktycznego obciążenia doszło wysyłana jest na adres URL przekazany w parametrze `p24_url_status`

Przykładowe wywołanie:
```swift
Express.builder()
    .url("https://e.przelewy24.pl/ESGByXP2S1SMxDm")
    .resultHandler(self))
    .build()
    .start()
```

> :warning:   **UWAGA** - Transakcja zostanie uznana przez system P24 za transakcje mobilną, gdy przy rejestracji zostaną przekazane parametry `p24_mobile_lib = 1` oraz `p24_sdk_version` którego wartością jest aktualna wersja biblioteki mobilnej - można ją uzyskać wywołując metodę `P24SdkVersion.value()` - **brak wskazanych parametrów uniemożliwi poprawne zamknięcie biblioteki**.

### 5.4. ApplePay

Proces przepływu danych przy wykorzystaniu metody płatności GooglePay wygląda następująco:

![Apple Pay](img/apay.png?raw=true "Apple Pay")

Po wyborze metody Apple Pay, aplikacja powinna wysłać request o stokenizowane dane płatnika. W momencie otrzymania informacji zwrotnej z Apple'a, aplikacja sprzedawcy przekazuje stokenizowane dane do własnego backendu, który rejestruje transakcję, a przekazane dane dołącza do żądania jako wartość parametru `p24_method_ref_id`. Po zarejestrowaniu transakcji token powinien trafić do aplikacji sprzedawcy, która wywoła metodę biblioteki z odpowiednim środowiskiem:

```swift
 ApplePay.builder()
    .token("received_token")
    .environment("environment_for_which_token_was_registrated")
    .resultHandler(self)
    .build()
    .start()
```

Przy poprawnie zarejestrowanej transakcji, biblioteka mobilna wyśle żądanie obciążenia do backendu Przelewy24 - w momencie otrzymania odpowiedzi, biblioteka zamyka się ze statusem adekwatnym do otrzymanej odpowiedzi.

Jako resultHandler w tym przypadku powinna zostać przekazana implementacja protokołu `ApplePayResultHandler`:

```swift
func onResult(_ result: ApplePayResult) {
        switch result {
            case .SUCCESS:
                // perform action when success
            case .ERROR(let errorMessage):
                // perform action when error
            case .CANCEL:
                // perform action when cancel
        }
    }
```
> :warning:   **UWAGA** - Należy pamiętać, że metoda `onSuccess()` informuje o tym, że backend P24 przyjął transakcję do realizacji - informacja o tym, czy do faktycznego obciążenia doszło wysyłana jest na adres URL przekazany w parametrze `p24_url_status`



