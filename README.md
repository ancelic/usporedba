
# FISKALIZACIJA 2.0 - WORKFLOW DOKUMENTACIJA

## eRačuni B2B - Kompletan Vodič za Implementaciju

  

---

  

## 📋 SADRŽAJ

  

1. [Uvod i Osnove](#1-uvod-i-osnove)

2. [API Konfiguracija](#2-api-konfiguracija)

3. [Statusi i Njihova Značenja](#3-statusi-i-njihova-znacenja)

4. [Glavni Workflow Scenariji](#4-glavni-workflow-scenariji)

5. [API Endpointi - Detaljno](#5-api-endpointi---detaljno)

6. [Konkretni Primjeri za Svaki Poziv](#6-konkretni-primjeri-za-svaki-poziv)

7. [Česti Slučajevi Upotrebe](#7-cesti-slucajevi-upotrebe)

8. [Greške i Rješavanje Problema](#8-greske-i-rjesavanje-problema)

  

---

  

## 1. UVOD I OSNOVE

  

### 1.1 Što je Fiskalizacija 2.0?

  

Fiskalizacija 2.0 proširuje obveze evidentiranja računa kroz sustav Porezne uprave na **eRačune u B2B poslovanju**. Sustav omogućuje:

- Praćenje gospodarskih tokova u realnom vremenu

- Administrativno rasterećenje kroz predispunjavanje obrazaca

- Smanjenje nelikvidnosti u gospodarstvu

  

### 1.2 Osnovni Pojmovi

  

-  **ULAZNI eRačun**: Račun koji tvrtka PRIMA od dobavljača

-  **IZLAZNI eRačun**: Račun koji tvrtka IZDAJE kupcu

-  **Fiskalizacija**: Dostava podataka o eRačunima Poreznoj upravi

-  **eIzvještavanje**: Izvještavanje o naplati/odbijanju računa

  

### 1.3 Obveze Poreznih Obveznika

  

#### Izdavatelj računa (Pošiljatelj):

- ✅ Dostava fiskalizacijskih podataka izlaznog eRačuna

- ✅ Izvještavanje o naplati izlaznog računa

- ✅ Evidentiranje isporuke za koju nije izdan eRačun

  

#### Primatelj računa:

- ✅ Dostava fiskalizacijskih podataka ulaznog eRačuna

- ✅ Izvještavanje o odbijanju ulaznog računa

  

---

  

## 2. API KONFIGURACIJA

  

### 2.1 Base URL-ovi

  

**TEST Okolina:**

```

https://test.sveracun.hr/api/rest/v1

```

  

**PRODUKCIJSKA Okolina:**

```

https://prod.sveracun.hr/api/rest/v1

```

*(napomena: produkcijski URL će biti drugačiji, provjerite s providerom)*

  

### 2.2 Autentifikacija

  

Svi API pozivi zahtijevaju **Authorization header** sa API ključem.

  

**TEST API Key:**

```

kNLTChNR14jwid+RDhNB6UgeFRRV0vhnWTsTE5I9WeJpGku36bIIIKFHZYJfg+GKMUXTXvSCYrwY0ZLIO/QD19WBMQUdUaqpnTvsIoF7gKtaS8GQ4Z4J/171QAMBRTa1Y4Z4zspVVCiXsyplKN7UYTgydQYD5T7mnrxx99yq3EBJUPleNPElV+Zci75TSLiK7Dg90RExHnefesaJ47Bb

```

  

### 2.3 Content-Type Headeri

  

-  **application/json** - Za većinu API poziva

-  **application/octet-stream** - Za slanje XML dokumenata (POST /documents/send)

  

### 2.4 Testni Podaci

  

**Testni Primatelj (API Primatelj d.o.o.):**

- OIB: `HR12345678901`

- Naziv: API Primatelj d.o.o.

  

**Vaša Testna Tvrtka (Izdavatelj):**

- OIB: `HR99392574874`

  

---

  

## 3. STATUSI I NJIHOVA ZNAČENJA

  

### 3.1 INTERNI STATUSI

  

Interni statusi se odnose na obradu računa unutar sustava SveRačun:

  

| Status | Značenje | Opis |

|--------|----------|------|

| **UNKNOWN** | U obradi | Račun je još uvijek u procesu obrade |

| **FAILED** | Neuspješno | Račun nije mogao biti poslan (XML greška, neispravan OIB, parsing greška) |

| **OK** | Uspješno | Račun je uspješno poslan na AS4 mrežu (od 1.1.2026) |

| **NEW** | Novi | Ulazni račun je stigao na sustav, ali još nije preuzet |

  

**Napomena:** Status se mijenja iz NEW u OK kada primatelj preuzme račun.

  

### 3.2 EKSTERNI STATUSI

  

Eksterni statusi se odnose na uspješnost fiskalizacije i eIzvještavanja:

  

| Status | Značenje | Opis |

|--------|----------|------|

| **UNKNOWN** | Bez eksternog statusa | Još nema povratne informacije od Porezne uprave |

| **FISCALIZED** | Fiskalizirano | Račun je uspješno fiskaliziran |

| **PAID** | Plaćeno | Evidentirana naplata računa |

| **REJECTED** | Odbijeno | Račun je odbijen od strane primatelja |

| **ERROR** | Greška | Greška prilikom fiskalizacije/izvještavanja |

  

**Važno:** Eksterni status se postavlja samo za radnje koje idu preko SveRačun sustava. Ako fiskalizaciju radite drugim servisom, tu informaciju sustav neće imati.

  

---

  

## 4. GLAVNI WORKFLOW SCENARIJI

  

### 4.1 IZLAZNI eRačun (Izdavanje Računa)

  

```

┌─────────────────────────────────────────────────────────┐

│ IZDAVANJE I FISKALIZACIJA RAČUNA │

└─────────────────────────────────────────────────────────┘

  

1. PRIPREMA XML RAČUNA

└─> Generirate UBL 2.1 XML format računa

(koristi primjere iz projekta)

  

2. SLANJE RAČUNA NA FISKALIZACIJU

└─> POST /documents/send

└─> Body: Binary XML

└─> Response: documentId

└─> Interni status: UNKNOWN → OK

└─> Eksterni status: UNKNOWN → FISCALIZED

  

3. PRAĆENJE STATUSA

└─> GET /documents/{documentId}

└─> Provjeravate interne i eksterne statuse

  

4. EVIDENCIJA NAPLATE (kada kupac plati)

└─> POST /documents/paid

└─> Šaljete podatke o plaćanju

└─> Eksterni status: PAID

```

  

### 4.2 ULAZNI eRačun (Primanje Računa)

  

```

┌─────────────────────────────────────────────────────────┐

│ PRIMANJE I OBRADA RAČUNA │

└─────────────────────────────────────────────────────────┘

  

1. PROVJERA ULAZNIH RAČUNA

└─> POST /documents/inbox/status

└─> Dohvaćate listu ulaznih računa

└─> Dobivate documentId svakog računa

  

2. PREUZIMANJE RAČUNA

└─> POST /documents/get

└─> Body: { "documentId": "..." }

└─> Response: Binary XML računa

└─> Interni status: NEW → OK (nakon preuzimanja)

  

3A. PRIHVAĆANJE RAČUNA

└─> Samo potvrdite primitak

└─> Račun ostaje u statusu OK

  

3B. ODBIJANJE RAČUNA (ako je nešto krivo)

└─> POST /documents/reject

└─> Navedete razlog odbijanja

└─> Eksterni status: REJECTED

```

  

### 4.3 STORNIRANJE / PONIŠTAVANJE RAČUNA

  

```

┌─────────────────────────────────────────────────────────┐

│ POSTUPAK ZA PONIŠTAVANJE │

└─────────────────────────────────────────────────────────┘

  



OPCIJA : Odbijanje od strane primatelja

└─> Primatelj odbija račun metodom /documents/reject

```

  

---

  

## 5. API ENDPOINTI - DETALJNO

  

### 5.1 POST /documents/send

**Svrha:** Slanje eRačuna na fiskalizaciju

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/octet-stream

companyVatNumber: HR99392574874

fiscalization: false

```

  

**Body:** Binary XML dokument (UBL 2.1 format)

  

**Response:**

```json

{

"documentId": "690073298c990405d2e1236a",

"statusCode": 0,

"resultMessage": "Document sent successfully"

}

```

  

---

  

### 5.2 POST /documents/inbox/status

**Svrha:** Dohvaćanje liste ulaznih računa

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/json

companyVatNumber: HR99392574874

fiscalization: false

```

  

**Request Body:**

```json

{

"companyVatNumber": "HR99392574874",

"dateFrom": "2025-01-01T00:00:00Z",

"dateTo": "2025-12-31T23:59:59Z"

}

```

  

**Response:**

```json

{

"items": [

{

"documentId": "68fa22488c990405d2e1207f",

"documentSenderName": "API Pošiljatelj, vl. Ana Petrov-Ilić",

"documentNumber": "43250087",

"internalStatus": "OK",

"externalStatus": null

},

{

"documentId": "68fa226b8c990405d2e1218d",

"documentSenderName": "API Primatelj d.o.o.",

"documentNumber": "43250085",

"internalStatus": "OK",

"externalStatus": null

}

]

}

```

  

---

  

### 5.3 POST /documents/outbox/status

**Svrha:** Dohvaćanje liste izlaznih računa

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/json

companyVatNumber: HR99392574874

fiscalization: false

```

  

**Request Body:**

```json

{

"companyVatNumber": "HR99392574874",

"dateFrom": "2025-01-01T00:00:00Z",

"dateTo": "2025-12-31T23:59:59Z"

}

```

  

**Response:**

```json

{

"items": [

{

"documentId": "690073298c990405d2e1236a",

"documentReceiverName": "API Primatelj d.o.o.",

"documentNumber": "RACUN-TEST-001",

"internalStatus": "OK",

"externalStatus": "UNKNOWN"

},

{

"documentId": "690616d88c990405d2e1257b",

"documentReceiverName": "API Primatelj d.o.o.",

"documentNumber": "RACUN-TEST-001",

"internalStatus": "OK",

"externalStatus": "FISCALIZED"

}

]

}

```

  

---

  

### 5.4 POST /documents/get

**Svrha:** Preuzimanje XML-a specifičnog eRačuna

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/json

companyVatNumber: HR99392574874

fiscalization: false

```

  

**Request Body:**

```json

{

"documentId": "68fa22488c990405d2e1207f"

}

```

  

**Response:** Binary XML dokument

  

**Napomena:** Odgovor je u binarnom formatu (XML datoteka). Spremite ga kao .xml file.

  

---

  

### 5.5 GET /documents/{documentId}

**Svrha:** Dohvaćanje detalja o dokumentu i statusima

  

**Headers:**

```

Authorization: {API_KEY}

companyVatNumber: HR99392574874

fiscalization: false

```

  

**URL Parameter:**

-  `{documentId}` - ID dokumenta dobiven iz send/inbox/outbox metoda

  

**Response:**

```json

{

"documentId": "690073298c990405d2e1236a",

"documentNumber": "RACUN-TEST-001",

"issueDate": "2025-11-06T12:00:00Z",

"senderVatNumber": "HR99392574874",

"receiverVatNumber": "HR12345678901",

"totalAmount": 1250.00,

"currency": "EUR",

"internalStatus": "OK",

"externalStatus": "FISCALIZED",

"fiscalizationJir": "12345-67890-12345",

"fiscalizationDate": "2025-11-06T12:05:00Z"

}

```

  

---

  

### 5.6 POST /documents/paid

**Svrha:** Evidencija naplate izlaznog računa (eIzvještavanje)

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/json

companyVatNumber: HR99392574874

fiscalization: false

```

  

**Request Body:**

```json

{

"documentId": "690073298c990405d2e1236a",

"documentNumber": "RACUN-TEST-001",

"issueDate": "2025-11-06T12:00:00Z",

"senderVatNumber": "HR99392574874",

"receiverVatNumber": "HR12345678901",

"paymentDate": "2025-11-15T10:30:00Z",

"paymentAmount": 1250.00,

"paymentMethod": "TRANSAKCIJSKI_RACUN"

}

```

  

**Request Fields:**

-  `documentId` - ID dobiven prilikom slanja računa

-  `documentNumber` - Broj računa iz XML-a

-  `issueDate` - Datum izdavanja računa

-  `senderVatNumber` - OIB izdavatelja

-  `receiverVatNumber` - OIB primatelja

-  `paymentDate` - Datum kada je izvršena naplata

-  `paymentAmount` - Iznos naplate (može biti manji od ukupnog iznosa za djelomičnu naplatu)

-  `paymentMethod` - Način plaćanja

  

**Načini plaćanja (paymentMethod):**

-  `TRANSAKCIJSKI_RACUN` - Bankovni prijenos

-  `GOTOVINA` - Gotovina

-  `KARTICA` - Platna kartica

-  `CEKOVI` - Ček

-  `OSTALO` - Ostali načini

  

**Response:**

```json

{

"statusCode": 0,

"resultMessage": "Payment reported successfully",

"externalStatus": "PAID"

}

```

  

**Napomena:** Nakon uspješnog poziva, eksterni status dokumenta će se promijeniti u **PAID**.

  

---

  

### 5.7 POST /documents/reject

**Svrha:** Odbijanje ulaznog računa od strane primatelja

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/json

companyVatNumber: HR99392574874

fiscalization: false

```

  

**Request Body:**

```json

{

"documentId": "68fa22488c990405d2e1207f",

"documentNumber": "43250087",

"issueDate": "2025-11-06T12:00:00Z",

"senderVatNumber": "HR12345678901",

"receiverVatNumber": "HR99392574874",

"rejectionDate": "2025-11-07T08:43:00Z",

"rejectionReasonCode": "U",

"rejectionReason": "Kriva cijena artikla - razlika u ukupnom iznosu"

}

```

  

**Request Fields:**

-  `documentId` - ID ulaznog računa

-  `documentNumber` - Broj računa

-  `issueDate` - Datum izdavanja računa

-  `senderVatNumber` - OIB pošiljatelja (dobavljača)

-  `receiverVatNumber` - OIB primatelja (vas)

-  `rejectionDate` - Datum odbijanja

-  `rejectionReasonCode` - Šifra razloga odbijanja

-  `rejectionReason` - Opis razloga (max 1000 znakova)

  

**Šifre razloga odbijanja (rejectionReasonCode):**

-  **N** - Neusklađenost podataka koji NE utječu na obračun poreza

-  **U** - Neusklađenost podataka koji UTJEČU na obračun poreza

-  **O** - Ostalo

  

**Response:**

```json

{

"statusCode": 0,

"resultMessage": "Document rejected successfully",

"externalStatus": "REJECTED"

}

```

  

---

  

### 5.8 POST /organizations/register

**Svrha:** Registracija nove organizacije u sustavu

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/json

```

  

**Request Body:**

```json

{

"companyVatNumber": "HR11131111111",

"companyName": "Moja Tvrtka d.o.o.",

"companyAddressStreet": "Ilica 123",

"companyAddressCity": "Zagreb",

"companyAddressPostalCode": "10000",

"companyAddressState": "Zagreb",

"companyAddressCountry": "Hrvatska",

"mpsRegister": false,

"routingAddress": "11131111111"

}

```

  

**Response:**

```json

{

"organizationId": "68baaa12880b60409344f43b"

}

```

  

---

  

### 5.9 POST /organizations/findByVatNumber

**Svrha:** Pronalaženje organizacije po OIB-u

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/json

```

  

**Request Body:**

```json

{

"vatNumber": "HR11131111111"

}

```

  

**Response:**

```json

{

"id": "68baaccd880b60409344f43e",

"fullName": "Moja Tvrtka d.o.o.",

"vatNumber": "HR11131111111",

"addressCity": "Zagreb",

"addressPostalCode": "10000",

"registrationStatus": "VERIFIED"

}

```

  

---

  

### 5.10 POST /users/new

**Svrha:** Kreiranje novog korisnika za organizaciju

  

**Headers:**

```

Authorization: {API_KEY}

Content-Type: application/json

```

  

**Request Body:**

```json

{

"username": "ante.celic@itrjesenja.hr",

"lastName": "Ćelić",

"firstName": "Ante",

"password": "Sigurna_Lozinka1!",

"email": "ante.celic@itrjesenja.hr",

"organizationId": "68baaa12880b60409344f43b"

}

```

  

**Response:**

```json

{

"userId": "68baaac3880b60409564f43d"

}

```

  

---

  

## 6. KONKRETNI PRIMJERI ZA SVAKI POZIV

  

### PRIMJER 1: Slanje Izlaznog eRačuna sa PDV 25%

  

**Korak 1:** Priprema XML dokumenta

```bash

# Koristite primjer: eRacun-PDV25.xml iz projekta

# Provjerite da su ispravni podaci:

# - OIB izdavatelja (AccountingSupplierParty)

# - OIB primatelja (AccountingCustomerParty)

# - Broj računa

# - Stavke i iznosi

```

  

**Korak 2:** Slanje na API

```bash

curl  -X  POST  "https://test.sveracun.hr/api/rest/v1/documents/send"  \

-H "Authorization: kNLTChNR14jwid+RDhNB6UgeFRRV0vhnWTsTE5I9WeJpGku36bIIIKFHZYJfg+GKMUX..." \

-H  "Content-Type: application/octet-stream"  \

-H "companyVatNumber: HR99392574874" \

-H  "fiscalization: false"  \

--data-binary "@eRacun-PDV25.xml"

```

  

**Response:**

```json

{

"documentId": "690abc1234567890d2e1236a",

"statusCode": 0,

"resultMessage": "Document sent successfully"

}

```

  

**Korak 3:** Provjera statusa

```bash

curl  -X  GET  "https://test.sveracun.hr/api/rest/v1/documents/690abc1234567890d2e1236a"  \

-H "Authorization: kNLTChNR14jwid..." \

-H  "companyVatNumber: HR99392574874"  \

-H "fiscalization: false"

```

  

---

  

### PRIMJER 2: Dohvaćanje Ulaznih Računa

  

**Korak 1:** Dohvat liste ulaznih računa

```bash

curl  -X  POST  "https://test.sveracun.hr/api/rest/v1/documents/inbox/status"  \

-H "Authorization: kNLTChNR14jwid..." \

-H  "Content-Type: application/json"  \

-H "companyVatNumber: HR99392574874" \

-H  "fiscalization: false"  \

-d '{

"companyVatNumber":  "HR99392574874",

"dateFrom":  "2025-11-01T00:00:00Z",

"dateTo":  "2025-11-30T23:59:59Z"

}'

```

  

**Response:**

```json

{

"items": [

{

"documentId": "68fa22488c990405d2e1207f",

"documentSenderName": "Dobavljač d.o.o.",

"documentNumber": "DOB-2025-0123",

"internalStatus": "NEW",

"externalStatus": null

}

]

}

```

  

**Korak 2:** Preuzimanje XML-a računa

```bash

curl  -X  POST  "https://test.sveracun.hr/api/rest/v1/documents/get"  \

-H "Authorization: kNLTChNR14jwid..." \

-H  "Content-Type: application/json"  \

-H "companyVatNumber: HR99392574874" \

-H  "fiscalization: false"  \

-d '{

"documentId":  "68fa22488c990405d2e1207f"

}' \

--output ulazni-racun.xml

```

  

---

  

### PRIMJER 3: Evidencija Naplate

  

**Scenarij:** Izdali ste račun 15.11.2025, kupac je platio 25.11.2025

  

```bash

curl  -X  POST  "https://test.sveracun.hr/api/rest/v1/documents/paid"  \

-H "Authorization: kNLTChNR14jwid..." \

-H  "Content-Type: application/json"  \

-H "companyVatNumber: HR99392574874" \

-H  "fiscalization: false"  \

-d '{

"documentId":  "690abc1234567890d2e1236a",

"documentNumber":  "RAC-2025-0156",

"issueDate":  "2025-11-15T12:00:00Z",

"senderVatNumber":  "HR99392574874",

"receiverVatNumber":  "HR12345678901",

"paymentDate":  "2025-11-25T10:30:00Z",

"paymentAmount":  5000.00,

"paymentMethod":  "TRANSAKCIJSKI_RACUN"

}'

```

  

**Response:**

```json

{

"statusCode": 0,

"resultMessage": "Payment reported successfully",

"externalStatus": "PAID"

}

```

  

---

  

### PRIMJER 4: Odbijanje Ulaznog Računa

  

**Scenarij:** Primili ste račun sa pogrešnom cijenom

  

```bash

curl  -X  POST  "https://test.sveracun.hr/api/rest/v1/documents/reject"  \

-H "Authorization: kNLTChNR14jwid..." \

-H  "Content-Type: application/json"  \

-H "companyVatNumber: HR99392574874" \

-H  "fiscalization: false"  \

-d '{

"documentId":  "68fa22488c990405d2e1207f",

"documentNumber":  "DOB-2025-0123",

"issueDate":  "2025-11-20T10:00:00Z",

"senderVatNumber":  "HR12345678901",

"receiverVatNumber":  "HR99392574874",

"rejectionDate":  "2025-11-22T14:30:00Z",

"rejectionReasonCode":  "U",

"rejectionReason":  "Kriva jedinična cijena artikla XYZ-001. Dogovorena cijena: 100 EUR, fakturirana: 150 EUR. Molimo ispravak računa."

}'

```

  

---

  

### PRIMJER 5: Djelomična Naplata

  

**Scenarij:** Račun na 10,000 EUR, kupac platio samo 6,000 EUR

  

```bash

# Prva naplata - 6,000 EUR

curl  -X  POST  "https://test.sveracun.hr/api/rest/v1/documents/paid"  \

-H "Authorization: kNLTChNR14jwid..." \

-H  "Content-Type: application/json"  \

-H "companyVatNumber: HR99392574874" \

-H  "fiscalization: false"  \

-d '{

"documentId":  "690abc1234567890d2e1236a",

"documentNumber":  "RAC-2025-0200",

"issueDate":  "2025-11-01T12:00:00Z",

"senderVatNumber":  "HR99392574874",

"receiverVatNumber":  "HR12345678901",

"paymentDate":  "2025-11-10T09:00:00Z",

"paymentAmount":  6000.00,

"paymentMethod":  "TRANSAKCIJSKI_RACUN"

}'

  

# Druga naplata - preostalih 4,000 EUR

curl -X POST "https://test.sveracun.hr/api/rest/v1/documents/paid" \

-H "Authorization: kNLTChNR14jwid..." \

-H "Content-Type: application/json" \

-H "companyVatNumber: HR99392574874" \

-H "fiscalization: false" \

-d '{

"documentId":  "690abc1234567890d2e1236a",

"documentNumber":  "RAC-2025-0200",

"issueDate":  "2025-11-01T12:00:00Z",

"senderVatNumber":  "HR99392574874",

"receiverVatNumber":  "HR12345678901",

"paymentDate":  "2025-11-20T11:30:00Z",

"paymentAmount":  4000.00,

"paymentMethod":  "TRANSAKCIJSKI_RACUN"

}'

```

  

---

  

## 7. ČESTI SLUČAJEVI UPOTREBE

  

### 7.1 SCENARIO: Standardno Izdavanje Računa i Naplata

  

**TIMELINE:**

```

Dan 1: Izdavanje računa

↓

POST /documents/send

↓

Dobijete: documentId = "ABC123"

↓

Interni status: UNKNOWN → OK

Eksterni status: UNKNOWN → FISCALIZED

  

Dan 15: Kupac platio račun

↓

POST /documents/paid

↓

Eksterni status: PAID

```

  

**IMPLEMENTACIJA:**

```javascript

// Dan 1: Slanje računa

const  xmlData = fs.readFileSync('racun.xml');

const  sendResponse = await  fetch('https://test.sveracun.hr/api/rest/v1/documents/send', {

method:  'POST',

headers: {

'Authorization':  API_KEY,

'Content-Type':  'application/octet-stream',

'companyVatNumber':  'HR99392574874',

'fiscalization':  'false'

},

body:  xmlData

});

const { documentId } = await  sendResponse.json();

  

// Spremite documentId u bazu podataka povezan s računom

  

// Dan 15: Evidencija naplate

await  fetch('https://test.sveracun.hr/api/rest/v1/documents/paid', {

method:  'POST',

headers: {

'Authorization':  API_KEY,

'Content-Type':  'application/json',

'companyVatNumber':  'HR99392574874',

'fiscalization':  'false'

},

body:  JSON.stringify({

documentId:  documentId,

documentNumber:  'RAC-2025-0001',

issueDate:  '2025-11-01T12:00:00Z',

senderVatNumber:  'HR99392574874',

receiverVatNumber:  'HR12345678901',

paymentDate:  '2025-11-15T10:30:00Z',

paymentAmount:  5000.00,

paymentMethod:  'TRANSAKCIJSKI_RACUN'

})

});

```

  

---

  

### 7.2 SCENARIO: Primanje i Obrada Ulaznog Računa

  

**TIMELINE:**

```

Svaki dan: Provjera novih ulaznih računa

↓

POST /documents/inbox/status

↓

Za svaki novi račun (status = NEW):

↓

POST /documents/get → preuzimanje XML-a

↓

Parsiranje i unos u knjigovodstveni sustav

↓

ODLUKA:

├─> Račun OK → NE TREBA NIŠTA

└─> Račun KO → POST /documents/reject

```

  

**IMPLEMENTACIJA:**

```javascript

// Dnevna provjera ulaznih računa (CRON job)

async  function  checkInboxDaily() {

const  response = await  fetch('https://test.sveracun.hr/api/rest/v1/documents/inbox/status', {

method:  'POST',

headers: {

'Authorization':  API_KEY,

'Content-Type':  'application/json',

'companyVatNumber':  'HR99392574874',

'fiscalization':  'false'

},

body:  JSON.stringify({

companyVatNumber:  'HR99392574874',

dateFrom:  getTodayStart(),

dateTo:  getTodayEnd()

})

});

const { items } = await  response.json();

for (const  invoice  of  items) {

if (invoice.internalStatus === 'NEW') {

// Preuzmi XML

const  xmlResponse = await  fetch('https://test.sveracun.hr/api/rest/v1/documents/get', {

method:  'POST',

headers: {

'Authorization':  API_KEY,

'Content-Type':  'application/json',

'companyVatNumber':  'HR99392574874',

'fiscalization':  'false'

},

body:  JSON.stringify({ documentId:  invoice.documentId })

});

const  xmlData = await  xmlResponse.text();

// Parsiraj i spremi u bazu

await  processIncomingInvoice(xmlData, invoice.documentId);

}

}

}

```

  

---

  

### 7.3 SCENARIO: Pogrešan Račun - Potrebno Storniranje

  

**OPCIJA A: Izdavatelj shvatio grešku**

```

1. Izdavanje Storno Računa

↓

POST /documents/send (negativni iznosi)

↓

Referencira originalni račun

```

  

**OPCIJA B: Primatelj uočio grešku**

```

1. Primatelj odbija račun

↓

POST /documents/reject

↓

Izdavatelj dobiva obavijest o odbijanju

↓

Izdavatelj šalje ispravljen račun

```

  

**IMPLEMENTACIJA - STORNO RAČUN:**

```xml

<!-- Primjer storno računa -->

<Invoice>

<cbc:ID>STORNO-001</cbc:ID>

<cbc:InvoiceTypeCode>381</cbc:InvoiceTypeCode>

<!-- 381 = Credit note -->

<cac:BillingReference>

<cac:InvoiceDocumentReference>

<cbc:ID>RAC-2025-0100</cbc:ID>

</cac:InvoiceDocumentReference>

</cac:BillingReference>

<cac:InvoiceLine>

<cbc:InvoicedQuantity>-1</cbc:InvoicedQuantity>

<cbc:LineExtensionAmount>-1000.00</cbc:LineExtensionAmount>

</cac:InvoiceLine>

</Invoice>

```

  

---

  

### 7.4 SCENARIO: Predujam i Finalni Račun

  

**TIMELINE:**

```

Dan 1: Izdavanje računa za predujam

↓

POST /documents/send (Racun_za_predujam.xml)

↓

Kupac platio predujam

↓

POST /documents/paid (iznos predujma)

  

Dan 30: Završetak posla - izdavanje finalnog računa

↓

POST /documents/send (Finalni_racun-predujam.xml)

↓

Račun referencira predujam

↓

Kupac platio razliku

↓

POST /documents/paid (preostali iznos)

```

  

**VAŽNO:** Finalni račun mora imati referencu na račun za predujam!

  

---

  

### 7.5 SCENARIO: Račun sa Prijenosom Porezne Obveze

  

**Kada se koristi:**

- Građevinski radovi

- Dobava zlata

- Emisijske jedinice stakleničkih plinova

  

**Implementacija:**

```xml

<!-- eRacun-Prijenos_porezne_obveze.xml -->

<Invoice>

<!-- Standardni podaci -->

<cac:TaxTotal>

<cac:TaxSubtotal>

<cac:TaxCategory>

<cbc:ID>AE</cbc:ID>

<!-- AE = Reverse charge (prijenos porezne obveze) -->

<cbc:Percent>0</cbc:Percent>

<cbc:TaxExemptionReasonCode>VATEX-EU-AE</cbc:TaxExemptionReasonCode>

<cbc:TaxExemptionReason>Prijenos porezne obveze</cbc:TaxExemptionReason>

</cac:TaxCategory>

</cac:TaxSubtotal>

</cac:TaxTotal>

</Invoice>

```

  

---

  

## 8. GREŠKE I RJEŠAVANJE PROBLEMA

  

### 8.1 Česte Greške

  

#### GREŠKA: "Bad Request" na /documents/reject

  

**Uzrok:**

- Pogrešan documentId (koristili ste ID izlaznog računa umjesto ulaznog)

- Pogrešan OIB (sender/receiver zamijeneni)

- Nedostaju obavezna polja

  

**Rješenje:**

```javascript

// KRIVO ❌

{

"documentId": "6906802a60c98a6bc755870c", // Ovo je IZLAZNI račun!

"senderVatNumber": "HR99392574874"  // Vi ste PRIMATELJ, ne pošiljatelj!

}

  

// TOČNO ✅

{

"documentId": "68fa22488c990405d2e1207f", // ID iz inbox/status

"senderVatNumber": "HR12345678901", // OIB dobavljača

"receiverVatNumber": "HR99392574874"  // Vaš OIB

}

```

  

---

  

#### GREŠKA: "Fiscal document not found"

  

**Uzrok:**

- DocumentId ne postoji

- Koristite testni documentId u produkciji (ili obrnuto)

- CompanyVatNumber u headeru ne odgovara vlasniku dokumenta

  

**Rješenje:**

- Provjerite da je companyVatNumber header ispravan

- Provjerite okruženje (test vs prod)

- Prvo pozovite inbox/status ili outbox/status da dobijete ispravne ID-ove

  

---

  

#### GREŠKA: XML Parsing Failed

  

**Uzrok:**

- XML nije validan UBL 2.1 format

- Nedostaju obavezna polja

- Krivi OIB format

  

**Rješenje:**

- Validirajte XML online: https://www.moj-eracun.hr/exchange/validateubl

- Koristite primjere iz projekta kao template

- Provjerite da su svi OIB-ovi u formatu: HRxxxxxxxxxxx (HR + 11 znamenki)

  

---

  

### 8.2 Debugging Checklist

  

**Prije nego pozovete API:**

- [ ] Provjeren companyVatNumber u headeru

- [ ] Provjeren Authorization token

- [ ] Provjereno okruženje (test/prod)

- [ ] XML validiran ako šaljete dokument

- [ ] Svi datumi u ISO 8601 formatu (YYYY-MM-DDTHH:mm:ssZ)

- [ ] Iznosi u decimalnom formatu (ne string)

  

**Nakon greške:**

1. Pročitajte `resultMessage` u response-u

2. Provjerite status code

3. Logirajte cijeli request i response

4. Usporedite sa primjerima u dokumentaciji

  

---

  

### 8.3 Status Code Reference

  

| Code | Značenje | Akcija |

|------|----------|--------|

| 0 | Uspješno | OK |

| 400 | Bad Request | Provjerite format podataka |

| 401 | Unauthorized | Provjerite API key |

| 404 | Not Found | DocumentId ne postoji |

| 500 | Server Error | Kontaktirajte support |

  

---

  

## 9. DODATNE NAPOMENE

  

### 9.1 Važni Linkovi

  

-  **UBL Validator**: https://www.moj-eracun.hr/exchange/validateubl

-  **API V1 Dokumentacija**: https://www.moj-eracun.hr/hr/Manual/V1/Api

-  **API V2 Dokumentacija**: https://www.moj-eracun.hr/hr/Manual/Stable/Api

-  **Porezna - Fiskalizacija 2.0**: https://porezna.gov.hr/fiskalizacija/bezgotovinski-racuni

  

### 9.2 Kontakt za Podršku

  

**Email:** integracije@sveracun.hr

  

### 9.3 Migracija na Produkciju

  

Kada prelazite sa TEST na PROD okruženje:

  

1.  **Promijenite Base URL**

```

TEST: https://test.sveracun.hr/api/rest/v1

PROD: https://prod.sveracun.hr/api/rest/v1

```

  

2.  **Dobijete novi API Key** od providera

  

3.  **Koristite stvarne OIB-ove** umjesto testnih

  

4.  **Testirajte sve workflow-e** prije puštanja u produkciju

  

5.  **Postavite monitoring** za statuse dokumenata

  

---

  

## 10. QUICK REFERENCE - Najčešći Pozivi

  

### Izdavanje računa

```bash

POST  /documents/send

Headers:  Authorization,  Content-Type:  application/octet-stream,  companyVatNumber,  fiscalization

Body:  Binary  XML

```

  

### Provjera ulaznih računa

```bash

POST  /documents/inbox/status

Headers:  Authorization,  Content-Type:  application/json,  companyVatNumber,  fiscalization

Body:  {  companyVatNumber,  dateFrom,  dateTo  }

```

  

### Evidencija naplate

```bash

POST  /documents/paid

Headers:  Authorization,  Content-Type:  application/json,  companyVatNumber,  fiscalization

Body:  {  documentId,  documentNumber,  issueDate,  senderVatNumber,  receiverVatNumber,  paymentDate,  paymentAmount,  paymentMethod  }

```

  

### Odbijanje računa

```bash

POST  /documents/reject

Headers:  Authorization,  Content-Type:  application/json,  companyVatNumber,  fiscalization

Body:  {  documentId,  documentNumber,  issueDate,  senderVatNumber,  receiverVatNumber,  rejectionDate,  rejectionReasonCode,  rejectionReason  }

```

  

---

  

## VERZIJA DOKUMENTA

  

-  **Verzija:** 1.0

-  **Datum:** 11.11.2025

-  **Autor:** IT Rješenja d.o.o.

-  **Status:** Finalna verzija za implementaciju

  

---
