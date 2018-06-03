# NAV Online Invoice reporter

> A PHP interface for Online Invoice Data Reporting System of Hungarian Tax Office (NAV)

_PHP interfész a NAV Online számla adatszolgáltatásához_

https://github.com/pzs/nav-online-invoice &mdash; https://packagist.org/packages/pzs/nav-online-invoice

NAV Online számla oldala: https://onlineszamla-test.nav.gov.hu/


## Használat

A használathoz a NAV oldalán megfelelő regisztrációt követően létrehozott technikai felhasználó adatainak beállítása szükséges!


### Inicializálás

Technikai felhasználó (és szoftver) adatok beállítása, Reporter példány létrehozása:

```php
$apiUrl = "https://api-test.onlineszamla.nav.gov.hu/invoiceService";
$config = new NavOnlineInvoice\Config($apiUrl, "userData.json");
$config->useApiSchemaValidation(); // opcionális
$config->setCurlTimeout(20); // 20 másodperces cURL timeout (NAV szerver hívásnál), opcionális

$reporter = new NavOnlineInvoice\Reporter($config);

```

Minta JSON fájlok: [userData.json](tests/testdata/userData.sample.json), [softwareData.json](tests/testdata/softwareData.json).
JSON fájl helyett az értékeket tömbben is át lehet adni (lásd lent, Dokumentáció / Config osztály fejezet).
A konstruktor 3. paraméterében a software adatokat is át lehet adni opcionálisan, ez nem kötelező a NAV részéről.


### Adószám ellenőrzése (`queryTaxpayer`)

:warning: Ez az endpoint a NAV onlineszamla test oldalán jelenleg nem elérhető. (2018.05.25.)

```php
try {
    $result = $reporter->queryTaxpayer("12345678");

    if ($result) {
        print "Az adószám valid.\n";
        print "Az adószámhoz tartozó név: " . $result->taxpayerName . "\n";
        if (isset($result->taxpayerAddress)) {
            print "Cím: ";
            print_r($result->taxpayerAddress);
        } else {
            print "Az adószámhoz nem tartozik cím.";
        }
    } else {
        print "Az adószám nem valid.";
    }

} catch(Exception $ex) {
    print get_class($ex) . ": " . $ex->getMessage();
}

```


### Token kérése (`tokenExchange`)

Ezt a metódust célszerű használni a technikai felhasználó adatainak (és a program) tesztelésére is.


```php
try {
    $token = $reporter->tokenExchange();
    print "Token: " . $token;

} catch(Exception $ex) {
    print get_class($ex) . ": " . $ex->getMessage();
}

```


### Adatszolgáltatás (`manageInvoice`)

Az adatszolgáltatás metódus automatikusan lekéri a tokent is (`tokenExchange`), így ezt nem kell külön megtenni.

```php
try {
    $invoices = new NavOnlineInvoice\InvoiceOperations();
    $invoices->useDataSchemaValidation(); // opcionális

    $invoices->add(simplexml_load_file("invoice1.xml"));
    // vagy
    $invoices->add($szamlaXml);

    $transactionId = $reporter->manageInvoice($invoices);

    print "Tranzakciós azonosító a státusz lekérdezéshez: " . $transactionId;

} catch(Exception $ex) {
    print get_class($ex) . ": " . $ex->getMessage();
}

```


### Státusz lekérdezése (`queryInvoiceStatus`)

Az adatszolgáltatás operációval beküldött számla státuszának lekérdezésére szolgáló operáció. `$transactionId`-nak a `manageInvoice` metódus által visszaadott azonosítót kell megadni.

```php
try {
    $transactionId = "...";
    $statusXml = $reporter->queryInvoiceStatus($transactionId);

    print "Válasz XML objektum:\n";
    var_dump($statusXml);

} catch(Exception $ex) {
    print get_class($ex) . ": " . $ex->getMessage();
}

```


### Számla adatszolgáltatások lekérdezése (`queryInvoiceData`)

Beküldött számlák lekérdezése/keresése.

:warning: Ezt az interfészt még tesztelni (és szükség szerint javítani) kell.


```php
try {
    $queryData = [
        "invoiceNumber" => "T20190001",
        "requestAllModification" => true
    ];
    $responseXml = $reporter->queryInvoiceData("invoiceQuery", $queryData);

    print "Válasz XML objektum:\n";
    var_dump($responseXml);

} catch(Exception $ex) {
    print get_class($ex) . ": " . $ex->getMessage();
}

```


### Számla (szakmai) XML validálása küldés nélkül

Ha engedélyezzük a validációt az `InvoiceOperations` példányon, akkor az `add()` metódus hívásakor az átadott XML-ek validálva lesznek. (Hiba esetén `XsdValidationError` exception lesz dobva).


```php
try {
    $invoices = new NavOnlineInvoice\InvoiceOperations();
    $invoices->useDataSchemaValidation(); // validálás az XSD-vel

    $invoices->add(simplexml_load_file("invoice1.xml")); // SimpleXMLElement példány
    $invoices->add(simplexml_load_file("invoice2.xml"));

    // Ezen a ponton a fenti számla XML-ek validak

} catch(Exception $ex) {
    // A számla XML nem valid
    print get_class($ex) . ": " . $ex->getMessage();
}

```


## Dokumentáció

### `Config` osztály

`Config` példány létrehozásakor a `$baseUrl` és a technikai felhasználó adatok (`$user`) megadása kötelező.

`$baseUrl` tipikusan a következő:
- teszt környezetben: `https://api-test.onlineszamla.nav.gov.hu/invoiceService`
- éles környezetben: `https://api.onlineszamla.nav.gov.hu/invoiceService`

Konstruktorban a `$user` paraméter lehet egy JSON fájl neve, vagy egy array, mely a következő mezőket tartalmazza (NAV oldalán létrehozott technikai felhasználó adatai):
- `login`
- `password`
- `taxNumber`
- `signKey`: XML aláírókulcs
- `exchangeKey`: XML cserekulcs

A `$software` adatok megadása _nem kötelező_ a specifikáció alapján. Amennyiben mégis megadjuk, úgy a következő mezőket tartalmazhatja a JSON fájl, vagy az átadott array (figyeljünk, hogy az értékek megfeleljenek az XSD-nek!):

- `softwareId`
- `softwareName`
- `softwareOperation`
- `softwareMainVersion`
- `softwareDevName`
- `softwareDevContact`
- `softwareDevCountryCode`
- `softwareDevTaxNumber`


__Metódusok__

- `__construct(string $baseUrl, $user [, $software = null])`
- `setBaseUrl($baseUrl)`
- `useApiSchemaValidation([$flag = true])`: NAV szerverrel való kommunikáció előtt a kéréseket (envelop XML) validálja az XSD-vel. A példány alapértelmezett értéke szerint a validáció nincs bekapcsolva.
- `setSoftware($data)`
- `loadSoftware($jsonFile)`
- `setUser($data)`
- `loadUser($jsonFile)`
- `setCurlTimeout($timeoutSeconds)`: NAV szerver hívásánál (cURL hívás) timeout értéke másodpercben. Alapértelmezetten nincs timeout beállítva. Megjegyzés: manageInvoice hívásnál 2 szerver hívás is történik (token kérés és számlák beküldése), itt külön-külön kell érteni a timeout-ot.


### `Reporter` osztály

A `Reporter` osztály példányosításakor egyetlen paraméterben a korábban létrehozott `Config` példányt kell átadni.

Ezen az osztályon érhetjük el a NAV interfészén biztosított szolgáltatásokat. A metódusok nevei megegyeznek a NAV által biztosított specifikációban szereplő operáció nevekkel.


- `__construct(Config $config)`
- `manageInvoice(InvoiceOperations $invoiceOperations)`: A számla adatszolgáltatás beküldésére szolgáló operáció. Visszatérési értékként a transactionId-t adja vissza string-ként.
- `queryInvoiceData(string $queryType, array $queryData [, int $page = 1])`: A számla adatszolgáltatások lekérdezésére szolgáló operáció
- `queryInvoiceStatus(string $transactionId [, $returnOriginalRequest = false])`: A számla adatszolgáltatás feldolgozás aktuális állapotának és eredményének lekérdezésére szolgáló operáció
- `queryTaxpayer(string $taxNumber)`: Belföldi adószám validáló és címadat lekérdező operáció. Visszatérési éréke lehet `null` nem létező adószám esetén, `false` érvénytelen adószám esetén, vagy TaxpayerDataType XML elem név és címadatokkal valid adószám esetén
- `tokenExchange()`: Token kérése manageInvoice művelethez (közvetlen használata nem szükséges, viszont lehet használni, mint teszt hívás). Visszatérési értékként a dekódolt tokent adja vissza string-ként.


### `InvoiceOperations` osztály

`manageInvoice` híváshoz használandó collection, melyhez a feladni kívánt számlákat lehet hozzáadni. Ez az osztály opcionálisan validálja is az átadott szakmai XML-t az XSD-vel.

- `__construct()`
- `useDataSchemaValidation([$flag = true])`: Számla adat hozzáadásakor az XML-t (szakmai XML) validálja az XSD-vel. Alapértelmezetten nincs bekapcsolva a validáció.
- `setTechnicalAnnulment([$technicalAnnulment = true])`: `technicalAnnulment` flag állítása. Alapértelmezett érték false.
- `add(SimpleXMLElement $xml [, $operation = "CREATE"])`: Számla XML hozzáadása a listához
- `getTechnicalAnnulment()`
- `getInvoices()`


### Exception osztályok

- `XsdValidationError`: XSD séma validáció esetén, ha hiba történt (a requestXML-ben vagy szakmai XML-ben; a válasz XML nincs vizsgálva). Ez az exception a kliens oldali XML ellenőrzéskor keletkezhet még a szerverrel való kommunikáció előtt.
- `CurlError`: cURL hiba esetén, pl. nem tudott csatlakozni a szerverhez (pl. nincs internet, nem elérhető a szerver).
- `HttpResponseError`: Ha nem XML válasz érkezett, vagy nem sikerült azt parse-olni.
- `GeneralExceptionResponse`: NAV által visszaadott hibaüzenet, ha nem sikerült náluk technikailag valamit feldolgozni (lásd NAV-os leírás Hibakezelés fejezetét).
- `GeneralErrorResponse`: NAV által visszaadott hibaüzenet, ha az XML válaszban `GeneralErrorResponse` érkezett, vagy ha a `funcCode !== 'OK'`.


### PHP verzió és modulok

A NavOnlineInvoice modul tesztelve PHP 5.5 és 7.2 alatt.

Szükséges modulok:

- cURL
- OpenSSL


### Linkek

- https://onlineszamla-test.nav.gov.hu/dokumentaciok
- https://onlineszamla-test.nav.gov.hu/
- https://onlineszamla.nav.gov.hu/


## TODO

- Műveletek (queryTaxpayer, queryInvoiceData) manuális tesztelése, amint elérhető lesz az interfész a NAV szerverén
- További tesztek írása, ami a NAV szerverét is meghívja teszt közben


## License

[MIT](http://opensource.org/licenses/MIT)

Copyright (c) 2018 github.com/pzs

https://github.com/pzs/nav-online-invoice