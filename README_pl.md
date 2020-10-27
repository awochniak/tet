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

Biblioteka mobilna Przelewy24 jest natywną biblioteką platformy Android, która umożliwia wykonanie płatności w ramach jednej aplikacji bez konieczności korzystania z przeglądarki lub innej aplikacji. Biblioteka udostępnia zróżnicowane metody płatności: BLIK, przelewy bankowe, karty płatnicze, wirtualne portfele (np. SkyCash, PayPal) i inne (GooglePay).

### 3.1. Przebieg transakcji przy użyciu bibliotek mobilnej

Wywołanie płatności w bibliotece powoduje otwarcie WebView z załadowanym serwisem transakcyjnym Przelewy24. Kolejnym krokiem jest wybór metody płatności bądź wypełnienie danych potrzebnych do płatności np. dane kartowe bądź kod BLIK. W przypadku, gdy użytkownik wybierze jako metodę płatności przelew bankowy, konieczne jest zalogowanie się do wskazanego banku. Po zatwierdzeniu płatności WebView zamyka się, a biblioteka zwraca do aplikacji klienta jeden ze statusów - `płatność udana`, gdy transakcja przebiegnie pomyślnie bądź ze statusem `płatność nieudana`, w momencie gdy użytkownik wprowadzi na przykład nieprawidłowy kod BLIK. Gdy proces zostanie przerwany przez użytkownika na którymś z wymienionych wcześniej etapów biblioteka zamyka się ze statusem `płatność anulowana`. Przebieg procesu został zilustrowany na poniższym diagramie - jak widać biblioteka jest tylko jedną z części procesu, a do poprawnego działania wymaga również backendu po stronie sprzedawcy, który jest odpowiedzialny m.in. za weryfikację transakcji:


![Transaction flow](img/transaction.png?raw=true "Transaction flow")


<a name="config"></a>

## 4. Konfiguracja projektu

Do zastosowania biblioteki mobilnej P24 konieczne jest poprawne skonfigurowanie projektu i dodanie odpowiednich zależności. Flaga `minSdkVersion` w pliku `gradle` aplikacji i projektu powinna zostać ustawiona na wersję **co najmniej 21**. Dodatkowo w pliku `gradle` aplikacji należy dodać linię: 
```gradle
apply from: 'https://mobile.przelewy24.pl/p24plugin.gradle'
```
a wewnątrz sekcji `dependencies`:
```gradle
implementation "pl.przelewy24.sdk:public:+"
```

Przykładowo, końcowy plik gradle powinien wyglądać następująco:

```gradle
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
apply from: 'https://mobile.przelewy24.pl/p24plugin.gradle'

android {
    compileSdkVersion 29
    buildToolsVersion "29.0.2"
    defaultConfig {
        applicationId "pl.przelewy24.sdk.example"
        minSdkVersion 21
        targetSdkVersion 29
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
    implementation 'androidx.appcompat:appcompat:1.1.0'
    implementation 'androidx.core:core-ktx:1.2.0'
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
    implementation 'androidx.cardview:cardview:1.0.0'
    implementation 'com.google.android.material:material:1.1.0'
    implementation "pl.przelewy24.sdk:public:+"
}
```

<a name="methods"></a>

## 5. Dostępne metody

Przed wywołaniem jakiejkolwiek metody konieczne jest ustawienie sprzedawcy w konfiguracji biblioteki

```kotlin
P24Config.merchant = 
    object : P24Merchant {
		override val id = merchant_id
		override val crc = mapOf(
			Environment.PRODUCTION to "production_crc",
			Environment.SANDBOX to "sandbox_crc"
		)
	}
```

gdzie `merchant_id` jest identyfikatorem sprzedawcy uzyskanym w trakcie procesu rejestracji, a `production_crc` i `sandbox_crc` są kluczami CRC z odpowiednich środowisk.

### 5.1. TrnRequest

Metoda ta umożliwia inicjalizację płatności przyjmując jako parametr wejściowy `environment`, czyli środowisko, na którym została zarejestrowana transakcja (`Environment.SANDBOX` lub `Environment.PRODUCTION`) oraz `token` transakcji uzyskany w wyniku rejestracji transakcji przez backend sprzedawcy - szczegółowe informacje jak zarejestrować transakcję w systemie Przelewy24 można znaleźć [tutaj](https://developers.przelewy24.pl/index.php?pl#tag/Obsluga-transakcji-API).

Opcjonalnie, w celu obsłużenia rezultatu transakcji, do metody można przekazać obiekt handlera implementującego interfejs `TrnRequestResultHandler`, posiadający 3 metody wywoływane w momencie otrzymania przez bibliotekę określonego statusu:

```kotlin
override fun onSuccess() {}
override fun onError(errorCode: String) {}
override fun onCanceled() {}
```
> :warning:   **UWAGA** - Należy pamiętać, że metoda `onSuccess()` informuje o tym, że backend P24 przyjął transakcję do realizacji - informacja o tym, czy do faktycznego obciążenia doszło wysyłana jest na adres URL przekazany w parametrze `p24_url_status`

Przykładowe wywołanie metody TrnRequest:
``` kotlin
TrnRequest.builder()
    .token("00EA6F0B7B-EF258E-75F927-D4A582DB29")
    .environment(Environment.SANDBOX)
    .resultHandler(this)
    .build()
    .start(context)
```

> :warning:   **UWAGA** - Transakcja zostanie uznana przez system P24 za transakcje mobilną, gdy przy rejestracji zostaną przekazane parametry `p24_mobile_lib = 1` oraz `p24_sdk_version` którego wartością jest aktualna wersja biblioteki mobilnej - można ją uzyskać wywołując metodę `P24SdkVersion.value()` - **brak wskazanych parametrów uniemożliwi poprawne zamknięcie biblioteki**.

### 5.2. TrnDirect

Metoda ta umożliwia inicjalizację płatności, dzięki podaniu danych bezpośrednio w aplikacji.

Jako parametry wejściowe przyjmuje parametry rejestracji transakcji uwzględnione w dokumentacji technicznej P24, poszerzone o wybór środowiska - `Environment.PRODUCTION` lub `Environment.SANDBOX`. Parametry `sessionId`, `amountIngGr`, `currency`, `description`, `email` i `country` są parametrami obowiązkowymi. Dodatkowo, sprzedawca w ramach metody ma możliwość nadpisania domyślnego merchanta opisanego na początku rozdziału, nowym merchantem.

Opcjonalnie, w celu obsłużenia rezultatu transakcji, do metody można przekazać obiekt handlera implementującego interfejs `TrnDirectResultHandler`, posiadający 3 metody wywoływane w momencie otrzymania przez bibliotekę określonego statusu:

```kotlin
override fun onSuccess() {}
override fun onError(errorCode: String) {}
override fun onCanceled() {}
```

> :warning:   **UWAGA** - Należy pamiętać, że metoda `onSuccess()` informuje o tym, że transakcja została przyjęta do realizacji - żeby ustalić czy transakcja jest opłacona, po wywołaniu metody `onSuccess()` aplikacja sprzedawcy powinna odpytać własny backend o status trasakcji.

Przykładowe wywołanie metody `TrnDirect` z wypełnionymi wszystkimi dostępnymi polami:
``` kotlin
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
    .resultHandler(this)
    .build()
    .start(this)
```

Metoda `TrnDirect` umożliwia również implementację funkcjonalności **Pasażu w wersji 2.0**. Konieczne jest do tego przekazanie parametru `passageCart` w ramach `TrnDirectBuilder`. `PassageCart` jest listą obiektów typu `PassageCartItem`:
```kotlin
PassageCartItem.builder()
    .name("Test item")
    .description("Test item description")
    .number(1)
    .price(1)
    .quantity(1)
    .targetAmount(100)
    .targetPosId(another_merchant_id)
    .build()
```

> :warning:   **UWAGA** - Parametry `number` i `description` zgodnie z dokumentacją `Pasażu 2.0` są opcjonalne.

### 5.3. Express

Metoda `Express` umożliwia realizację transakcji w systemie Express. Jako parametr wejściowy przyjmuje `url`, który został uzyskany w wyniku rejestracji transakcji mobilnej w tym systemie.

Do obsługi rezultatu transakcji, należy dodatkowo przekazać obiekt handlera implementujący interfejs `ExpressResultHandler`, posiadający 3 metody wywoływane w momencie otrzymania przez bibliotekę określonego statusu:

```kotlin
override fun onSuccess() {}
override fun onError(errorCode: String) {}
override fun onCanceled() {}
```

> :warning:   **UWAGA** - Należy pamiętać, że metoda `onSuccess()` informuje o tym, że backend P24 przyjął transakcję do realizacji - informacja o tym, czy do faktycznego obciążenia doszło wysyłana jest na adres URL przekazany w parametrze `p24_url_status`

Przykładowe wywołanie:
```kotlin
Express.builder()
    .url("https://e.przelewy24.pl/ESGByXP2S1SMxDm")
    .resultHandler(this)
    .build()
    .start(this)
```

> :warning:   **UWAGA** - Transakcja zostanie uznana przez system P24 za transakcje mobilną, gdy przy rejestracji zostaną przekazane parametry `p24_mobile_lib = 1` oraz `p24_sdk_version` którego wartością jest aktualna wersja biblioteki mobilnej - można ją uzyskać wywołując metodę `P24SdkVersion.value()` - **brak wskazanych parametrów uniemożliwi poprawne zamknięcie biblioteki**.

### 5.4. GooglePay

Proces przepływu danych przy wykorzystaniu metody płatności GooglePay wygląda następująco:

![Google Pay](img/gpay.png?raw=true "Google Pay")

Po wyborze metody Google Pay, aplikacja powinna wysłać request o stokenizowane dane płatnika. W momencie otrzymania informacji zwrotnej z Google, aplikacja sprzedawcy przekazuje stokenizowane dane do własnego backendu, który rejestruje transakcję, a przekazane dane dołącza do żądania jako wartość parametru `p24_method_ref_id`. Po zarejestrowaniu transakcji token powinien trafić do aplikacji sprzedawcy, która wywoła metodę biblioteki z odpowiednim środowiskiem:

```kotlin
 GooglePay.builder()
    .token("received_token")
    .environment("environment_for_which_token_was_registrated")
    .resultHandler(this)
    .build()
    .start(this)
```

Przy poprawnie zarejestrowanej transakcji, biblioteka mobilna wyśle żądanie obciążenia do backendu Przelewy24 - w momencie otrzymania odpowiedzi, biblioteka zamyka się ze statusem adekwatnym do otrzymanej odpowiedzi.

Jako resultHandler w tym przypadku powinna zostać przekazana implementacja interfejsu `GooglePayResultHandler` posiadająca metody:

```kotlin
fun onSuccess()
fun onError(errorCode: String)
fun onCanceled()
```
> :warning:   **UWAGA** - Należy pamiętać, że metoda `onSuccess()` informuje o tym, że backend P24 przyjął transakcję do realizacji - informacja o tym, czy do faktycznego obciążenia doszło wysyłana jest na adres URL przekazany w parametrze `p24_url_status`



