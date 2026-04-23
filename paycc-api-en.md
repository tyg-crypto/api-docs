# Paycc API Documentation

- [Paycc API](#Paycc-API)
- [Interface Specifications](#Interface-Specifications)
- [1. Institution](#Institution)
     - [1.1 Query Supported Card Types](#Query-Supported-Card-Types)
     - [1.2 Query Institution Balance](#Query-Institution-Balance)
     - [1.3 Query Card Rates](#Query-Card-Rates)
     - [1.4 Estimate Fiat Amount to be Received](#Estimate-Fiat-Amount-to-be-Received)
     - [1.5 Estimate Crypto Amount Required for Deposit](#Estimate-Crypto-Amount-Required-for-Deposit)
- [2. KYC](#KYC)
     - [2.1 Submit User KYC Data](#Submit-User-KYC-Data)
     - [2.2 Query All User KYC Records](#Query-All-User-KYC-Records)
     - [2.3 Query Specific User KYC Record](#Query-Specific-User-KYC-Record)
- [3. Card Issuance and Activation](#Card-Issuance-and-Activation)
     - [3.1 Submit User Card Application](#Submit-User-Card-Application)
     - [3.2 User Activates Card](#User-Activates-Card)
     - [3.3 Query All Card Statuses](#Query-All-Card-Statuses)
     - [3.4 Query Specific User's All Card Statuses](#Query-Specific-Users-All-Card-Statuses)
- [4. Deposit to Card](#Deposit-to-Card)
     - [4.1 Deposit to User Card](#Deposit-to-User-Card)
     - [4.2 Query Coin Pair Price](#Query-Coin-Pair-Price)
     - [4.3 Query Status of a Specific Card Deposit Transaction](#Query-Status-of-a-Specific-Card-Deposit-Transaction)
     - [4.4 Query All Card Deposit Records](#Query-All-Card-Deposit-Records)
     - [4.5 Query Specific User's All Card Deposit Records](#Query-Specific-Users-All-Card-Deposit-Records)
- [8. Bank Card Query](#Bank-Card-Query)
     - [8.1 Query if Card is Activated](#Query-if-Card-is-Activated)
     - [8.2 Query Card Balance](#Query-Card-Balance)
     - [8.3 Query Card Statement](#Query-Card-Statement)
     - [8.4 Query Card Sensitive Information](#Query-Card-Sensitive-Information)
- [9. Error Codes](#Error-Codes)
     - [9.1 Business Logic Error Codes](#Business-Logic-Error-Codes)
     - [9.2 Identity and Permission Authentication Error Codes](#Identity-and-Permission-Authentication-Error-Codes)
     - [9.3 Exception Error Codes](#Exception-Error-Codes)
     - [9.4 KYC Failure Error Codes](#KYC-Failure-Error-Codes)

## Paycc-API

Welcome to the Paycc API documentation. This document is for Paycc's ToB card business, currently supporting physical cards, virtual cards, and shared cards, totaling dozens of card types (different card BINs). Different card types support different fiat currencies and fees, and the interface call parameters also vary slightly. For example:

| Card Name | Fiat | Differences |
| :--------: | :----: | :------------------------------ |
| J Card (card_type_id: 30000001 - 30000009) | USD | KYC review within 48h, KYC filling only supports English, mobile format like +8615821702552, user's name will be printed on the card, card production time 1-2 weeks. A selfie holding the card and passport must be submitted during activation. Arrival time within 4h. |
| P Card (card_type_id: 40000001 - 40000009) | EUR | KYC time within 1h, KYC filling only supports English, mobile format like +86-15821702552, country and nationality filled with two-letter country codes. Arrival time within 1h. |

API Usage Steps:

1. Please register an institution account at [https://customer.paycc.com/](https://customer.paycc.com/). If you cannot access it, please provide your IP address.
2. After our review and approval, the institution can log in successfully.
3. Institution logs in, checks the wallet address, and deposits to the wallet, supporting USDT, BTC, ETH, etc.
4. Institution logs in, creates Appkey and secret, and can optionally configure a webhook callback address.
5. Call the API for operations such as KYC, card issuance, card activation, and deposit. Paycc will notify the institution's server of status changes via the callback address.

![](./img/workflow0.jpeg)
![](./img/workflow-cn.jpeg)

> It is recommended to debug in the test environment before using the production environment.

Institution Dashboard:

* Test Environment (IP whitelist restricted): https://customer-sandbox.paycc.com/
* Production Environment (IP whitelist restricted): https://customer.paycc.com/

API Domains:

* Test Environment: https://api-sandbox.paycc.com/
* Production Environment: https://api.paycc.com/

## Interface Specifications

- Open API requests all use `HMAC` authentication to ensure the integrity and authenticity of the request, as well as for identity authentication and authorization.

- **Pagination**. Query record lists all have pagination. Pagination parameters: `page_num` indicates the page number, `page_size` indicates the page size. The interface `DTO` uniformly returns `total` and `records`.

- **Country**. Two-letter country code, referring to the `ISO 3166-1 alpha-2` encoding standard.

- Time processing. API request and return times are all `UNIX` timestamps, **in milliseconds**, to avoid errors caused by time zones.

- Amount processing. API requests and returns are all of `String` type to avoid precision loss.

- All requests with a `body` are in `JSON` format unless otherwise specified, `Content-Type: application/json`.

- The query time interval for all query interfaces must be **less than one month**.

- Uniform interface return format:

  | Parameter | Type | Description |
  | :--------: | :----: | :------------------------------ |
  | code | int | Error code. `0`: Normal, non-`0`: Exception |
  | msg | String | `SUCCESS` for success, error description for failure |
  | result | Object | Return information |

### HMAC Authentication

First, the institution needs to apply for an API `Key` and API `Secret`, which will be used when accessing the API.

| Term | Explanation |
| --- | --- |
| User ID | User ID is used to mark your developer account, it is your user ID |
| API Key & API Secret | User ID manages multiple API Key + API Secret pairs. API Key corresponds to your application. You can have multiple applications, and each application can apply for different API permissions |

#### Client Implementation Process:

1. Construct the data to be signed, including:
   - UNIX timestamp, `in milliseconds`: `request` time stamp
   - Request method: `HTTP` method
   - Request API Key: Api Key
   - Complete request path, including parameters after the `URL` question mark: request URI
   - If there is a request `body`, add the `string` representation of the request `body`: string representation of the request payload
2. The client uses the `HMAC_SHA256` algorithm to generate a signature based on the signed data and API Secret.
3. Set the Authorization header in the specified order, i.e., key is: `Authorization`, value is: Railone:ApiKey:request time stamp:signature (concatenated with colons).
4. If a password was used when creating the API Key and API Secret on the server side, you need to set the Access-Passphrase header, i.e., `key` is: `Access-Passphrase`, `value` is: the password set at that time.
5. The client sends the data and Authorization header, as well as the Access-Passphrase header (if step 4 exists) to the server. That is, the final http header sent is:
   - Authorization:Railone:ApiKey:request timestamp:signature
   - Access-Passphrase:Your API Secret passphrase

#### How to construct the request body string to be signed:

The request body needs to sort the parameter names in `ASCII` order, concatenate the key and value with `=`, and separate multiple key-values with `&`, converting them into a string.

For example, if the request `body` is:

```json
{
	"from_address":"Ae9ujqUnAtH9yRiepRvLUE3t9R2NbCTZPG",
	"amount":190,
	"to_address":"AUol16ghiT9AtxRDtNeq3ovhWJ5iaY6iyd"
}
```

After conversion, it is:

```text
amount=190&from_address=Ae9ujqUnAtH9yRiepRvLUE3t9R2NbCTZPG&to_address=AUol16ghiT9AtxRDtNeq3ovhWJ5iaY6iyd
```

## Institution

### Query Supported Card Types

```text
url: /api/v1/institution/card/type
method: GET
```

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| page_num | int | Optional | Page number |
| page_size | int | Optional | Page size |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "total": 2,
        "records": [
            {
                "card_type_id": "50000001",
                "currency_type": "USD",
                "bank_id": "5000",
                "description": "card 1",
                "card_network": "visa",
                "card_title": "F",
                "virtual_card": false
            },
            {
                "card_type_id": "50000002",
                "currency_type": "USD",
                "bank_id": "5000",
                "description": "card 2",
                "card_network": "visa",
                "card_title": "F-V",
                "virtual_card": true
            }
        ]
    }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| card_type_id | String | ID corresponding to the bank card type, e.g., 50000001 |
| currency_type | String | Fiat currency type supported by the card |
| bank_id | String | Bank ID |
| description | String | Card type description |
| card_network | String | Card issuing organization: visa, master, unionpay |
| virtual_card | Bool | Whether it is a virtual card |
| card_title | String | Supports F card, J card, P card, etc. The corresponding metal cards are F-M* card, J-M* card, P-M* card, and the corresponding virtual cards are F-V card, J-V card, P-V card |

### Query Institution Balance

- Request:

```text
url: /api/v1/institution/balance
method: GET
```

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": [
        {
            "addresses":[
                {
                    "balance":"100",
                    "address_type":"deposit",
                    "address":"0x0e420097975b6c700f6314914a1b6e66a1edc313"
                },
                {
                    "balance":"200",
                    "address_type":"open_card",
                    "address":"0x88fee6dda9a041e9ebe6c56924765ff54544e40c"
                }],
            "coin_type":"USDT"
        }
    ]
}
```

| Parameter | Type | Description |
| :-----------------------: | :----: | :--------------------------------- |
| coin_type | String | Digital currency type |
| addresses[0].balance | String | Digital currency balance |
| addresses[0].address_type | String | Digital currency address type, card opening (open_card) and deposit (deposit) |
| addresses[0].address | String | Digital currency address |

### Query Card Rates

```text
url: /api/v1/institution/rates?card_type_id={card_type_id}
method: GET
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| card_type_id | String | Required | ID corresponding to the bank card type, e.g., 50000001 |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "exchange_rate": "1.00505392692",
        "card_application_fee": "1",
        "min_deposit":"0",
        "max_deposit":"10000",
        "loading_rate": [
            {
                "min": "0",
                "max": "1000",
                "step_rate": "0.05"
            },
            {
                "min": "1000",
                "max": "2000",
                "step_rate": "0.03"
            }
        ],
        "bank_atm_fee": "0",
        "bank_atm_rate": "0.1",
        "bank_transaction_rate": "0.2"
    }
}
```

| Parameter | Type | Description |
| :-------------------: | :----: | :------------------------------ |
| card_application_fee | String | Card opening fee (USDT) |
| min_deposit | String | Minimum single deposit amount (fiat) |
| max_deposit | String | Maximum single deposit amount (fiat) |
| exchange_rate | String | Exchange rate of USDT to the corresponding fiat currency |
| loading_rate | String | Tiered fee rate paid to Paycc when depositing for users, most cards only have a single-tier rate |
| bank_transaction_rate | String | Fee rate for bank card swiping consumption |
| bank_atm_rate | String | Fee rate for ATM withdrawals |
| bank_atm_fee | String | Fixed fee for ATM withdrawals |

### Estimate Fiat Amount to be Received

- Request:

```text
Internal deduction url: /api/v1/institution/estimation/currency
External deduction url: /api/v1/institution/estimation/currency/external-deduction
method: POST
```

| Parameter | Type | Requirement | Description |
| :---------: | :---: | :--------------: | :-------------------------------------------------------- |
| coin_amount | String | Required | Amount of digital currency deposited |
| coin_type | String | Required | Type of digital currency deposited, USDT |
| card_type_id | String | Required | Card type ID |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "coin_type": "usdt",
        "coin_amount": "106.23",
        "currency_type": "usd",
        "currency_amount": "94.13",
        "exchange_rate": "1.00145966373",
        "fiat_exchange_rate": "1",
        "exchange_fee": "1.0623",
        "exchange_fee_rate": "0.01",
        "loading_fee": "5.3115",
        "coin_price": "1"
    }
}
```

| Parameter | Type | Description |
| :----------------: | :----: | :--------------------------- |
| coin_amount | String | Amount of digital currency deposited |
| coin_type | String | Type of digital currency deposited |
| currency_amount | String | Fiat amount received from deposit |
| currency_type | String | Fiat type received from deposit |
| exchange_fee | String | Fee for exchanging other currencies to USDT, unit is coin_type |
| exchange_fee_rate | String | Fee rate for exchanging other currencies to USDT |
| loading_fee | String | Deposit fee, unit is coin_type |
| exchange_rate | String | USDT/USD exchange rate |
| fiat_exchange_rate | String | Card supported fiat/USD exchange rate |
| coin_price | String | coin_type/USDT price |

> (currency_amount * fiat_exchange_rate) = ((coin_amount - exchange_fee - loading_fee) * coin_price) * exchange_rate 
> Total USD received = Actual USDT deposited * exchange_rate (USDT/USD)

### Estimate Crypto Amount Required for Deposit

- Request:

```text
url: /api/v1/institution/estimation/crypto
method: POST
```

| Parameter | Type | Requirement | Description |
| :---------: | :---: | :--------------: | :--------------------------------------------------------|
| currency_amount | String | Required | Fiat amount required to be received |
| card_type_id | String | Required | Card type ID |
| coin_type | String | Required | Digital currency type to be converted, usdt |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "coin_type": "usdt",
        "coin_amount": "106.23",
        "currency_type": "usd",
        "currency_amount": "94.13",
        "exchange_rate": "1.00145966373",
        "fiat_exchange_rate": "1",
        "exchange_fee": "1.0623",
        "exchange_fee_rate": "0.01",
        "loading_fee": "5.3115",
        "coin_price": "1"
    }
}
```

| Parameter | Type | Description |
| :---------: | :----: | :--------------------------- |
| coin_amount | String | Required digital currency amount |
| coin_type | String | Required digital currency type |
| currency_amount | String | Fiat amount received from deposit |
| currency_type | String | Fiat type received from deposit |
| exchange_fee | String | Fee for exchanging other currencies to USDT, unit is coin_type |
| exchange_fee_rate | String | Fee rate for exchanging other currencies to USDT |
| loading_fee | String | Deposit fee, unit is coin_type |
| exchange_rate | String | USDT/USD exchange rate |
| fiat_exchange_rate | String | Card supported fiat/USD exchange rate |
| coin_price | String | coin_type/USDT price |

> (currency_amount * fiat_exchange_rate) = ((coin_amount - exchange_fee - loading_fee) * coin_price) * exchange_rate 
> Total USD received = Actual USDT deposited * exchange_rate (USDT/USD)

## KYC

Interfaces provided for institutions to use, such as institutions creating KYC for users, querying KYC records, etc.

### Submit User KYC Data

If KYC review fails, also use this interface to resubmit.

```text
url: /api/v1/customers/accounts
method: POST
```

- Request:

| Parameter | Type | Requirement | Description |
| :--------------------: | :-------: | :---------: | :--------------------------------------------------- |
| acct_no | String | Required | Institution-side user number (unique on institution side), max length 64 |
| acct_name | String | Required | Institution-side username, max length 64 |
| first_name | String | Required | Real user first name, max length 50 |
| last_name | String | Required | Real user last name, max length 50 |
| gender | String | Required | male, female, unknown: other, max length 6 |
| birthday | String | Required | Birthday (format is "1990-01-01") |
| city | String | Required | City, max length 100 |
| state | String | Required | Province/State, max length 100 |
| country | String | Required | User's country, max length 50 |
| nationality | String | Required | Nationality, max length 255 |
| doc_no | String | Required | Document number, max length 128 |
| doc_type | String | Required | Document type (currently only supports passport): passport, idcard, max length 8 |
| front_doc | String | Required | Front photo. base64 encoded, photo file size should be less than 2M |
| back_doc | String | Optional | Back photo, required when doc_type is idcard. base64 encoded, photo file size should be less than 2M |
| mix_doc | String | Required | Photo holding document. base64 encoded, photo file size should be less than 2M |
| country_code | String | Required | Mobile international area code, e.g., "+86". Max length 5 |
| mobile | String | Required | Mobile number, max length 32 |
| mail | String | Required | Email, does not support 163.com emails. Max length 64 |
| address | String | Required | Mailing address, bank card will be sent to this address. Max length 256 |
| zipcode | String | Required | Zip code, max length 20 |
| card_type_id | String | Required | ID corresponding to the bank card type, e.g., 10010001 |
| maiden_name | String | Optional | Mother's name (can fill 'no'), max length 255 |
| kyc_info | String | Optional | KYC other information |
| mail_verification_code | String | Optional | Email verification code |
| mail_token | String | Optional | Token returned after sending email |
| cust_tx_id | String | Optional | KYC transaction number |
| sign_img | String | Optional | Signature photo. base64 encoded, photo file size should be less than 1M |
| poa_doc | String[3] | Optional | Proof of address photo (currently not supported). base64 encoded, each photo or PDF file size should be less than 2M |
| card_number | String | Optional | Supported by special card types, bound card number, used for advance card sales, required for Native physical cards |
| print_name_on_card | Bool | Optional | Whether to print name, required for Native physical cards |

- Response:

```json
{
  "code": 0,
  "msg": "string",
  "result": true
}
```

### Query All User KYC Records

```text
url: /api/v1/customers/accounts
method: GET
```

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| page_num | int | Optional | Page number |
| page_size | int | Optional | Page size |
| former_time | long | Optional | Start time, UNIX timestamp, `in seconds` |
| latter_time | long | Optional | End time, UNIX timestamp, `in seconds` |
| time_sort | String | Optional | Time sorting, asc for ascending, desc for descending |

- Response:

```json
{
  "code": 0,
  "msg": "SUCCESS",
  "result": {
    "total": 1,
    "records": [
      {
        "acct_no": "1222",
        "card_type_id": "10010001",
        "status": 3,
        "face_recognition_url": "https://www.hh.io", // Liveness detection link may only exist during authentication
        "reason": "{\"code\":1110032,\"msg\":\"KYC failure, photo error\"}",
        "create_time": 1546300800000
      }
    ]
  }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| acct_no | String | Institution-side user number (unique on institution side) |
| card_type_id | String | ID corresponding to the card type |
| status | int | Status code: 0 Submitted, 1 Authentication passed (card opening successful), 2 Authentication failed, 3 Authenticating, 4 Submitted information processing, 6 Refunded |
| reason | String | Reason for authentication failure. Empty string in other cases |
| create_time | long | Creation time |

> For failure reasons, please refer to KYC Failure Error Codes

### Query Specific User KYC Record

```text
url: /api/v1/customers/accounts
method: GET
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| acct_no | String | Required | Unique id number of the institution user |
| page_num | int | Optional | Page number |
| page_size | int | Optional | Page size |
| former_time | long | Optional | Start time, UNIX timestamp, `in seconds` |
| latter_time | long | Optional | End time, UNIX timestamp, `in seconds` |
| time_sort | String | Optional | Time sorting, asc for ascending, desc for descending |

- Response:

```json
{
  "code": 0,
  "msg": "SUCCESS",
  "result": {
    "total": 1,
    "records": [
      {
        "acct_no": "1222",
        "card_type_id": "10010001",
        "status": 3,
        "face_recognition_url": "https://www.hh.io", // Liveness detection link may only exist during authentication
        "reason": "{\"code\":1110032,\"msg\":\"KYC failure, photo error\"}",
        "create_time": 1546300800000
      }
    ]
  }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| acct_no | String | Institution-side user number (unique on institution side) |
| card_type_id | String | ID corresponding to the card type |
| status | int | Status code: 0 Submitted, 1 Authentication passed (card opening successful), 2 Authentication failed, 3 Authenticating, 4 Submitted information processing, 6 Refunded |
| reason | String | Reason for authentication failure. Empty string in other cases |
| create_time | long | Creation time |

> For failure reasons, please refer to KYC Failure Error Codes

### Query Specific User KYC Information

```text
url: /api/v1/customers/accounts/kyc
method: GET
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| acct_no | String | Required | Unique id number of the institution user |

- Response:

```json
{
  "code": 0,
  "msg": "SUCCESS",
  "result": {
    "total": 1,
    "records": [
      {
        "acct_no": "1222",
        ......
      }
    ]
  }
}
```

| Parameter | Type | Description |
| :------------: | :----: |:---------- |
| acct_no | String | Institution-side user number (unique on institution side), max length 64 |
| acct_name | String | Institution-side username, max length 64 |
| first_name | String | Real user first name, max length 50 |
| last_name | String | Real user last name, max length 50 |
| gender | String | male, female, unknown: other, max length 6 |
| birthday | String | Birthday (format is "1990-01-01") |
| city | String | City, max length 100 |
| state | String | Province/State, max length 100 |
| country | String | User's country, max length 50 |
| nationality | String | Nationality, max length 255 |
| doc_no | String | Document number, max length 128 |
| doc_type | String | Document type (currently only supports passport): passport, idcard, max length 8 |
| country_code | String | Mobile international area code, e.g., "+86". Max length 5 |
| mobile | String | Mobile number, max length 32 |
| mail | String | Email, does not support 163.com emails. Max length 64 |
| address | String | Mailing address, bank card will be sent to this address. Max length 256 |
| zipcode | String | Zip code, max length 20 |
| maiden_name | String | Mother's name, max length 255 |
| card_type_id | String | ID corresponding to the bank card type, e.g., 10010001 |
| kyc_info | text | KYC other information |
| cust_tx_id | String | KYC transaction number |
| status | int | Status code: 0 Submitted, 1 Authentication passed (card opening successful), 2 Authentication failed, 3 Authenticating, 4 Submitted information processing, 6 Refunded |

## Card Issuance and Activation

Provides interfaces for institutions to issue cards to users, query records, etc.

### Submit User Card Application

```text
url: /api/v1/debit-cards
method: POST
```

- Request

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| acct_no | String | Required | Institution-side user number (unique on institution side) |
| card_type_id | String | Required | ID corresponding to the card type |
| card_number | String | Optional | Required only for card types that support card binding. User fills in the card number to bind |
| urn | String | Optional | Required only for card types that support card binding. Envelope number |

- Response:

```json
{
  "code": 0,
  "msg": "string",
  "result": {
 	"card_no": "xxxx",
	"card_number": "430021******1144",
	"status": 2
  }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| card_no | String | Assigned bank card ID, use card_no when querying to avoid real card number information leakage. Generation rule: institution id + 5 random digits + card type id + last four digits of card |
| card_number | String | Assigned real bank card number, only shows first 6 and last 4 digits |
| status | int | Status: 2. Card application successful (unactivated), 5. Application failed (card is being produced, please apply again later) |

### User Activates Card

After the virtual card is successfully activated, the real card number, cvv, and expiration time can be obtained by querying the card information through the bank interface.

```text
url: /api/v1/debit-cards/status
method: PUT
```

- Request

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| card_no | String | Required | Bank card ID |
| acct_no | String | Required | Institution-side user number (unique on institution side) |
| cvv | String | Optional | Some physical cards require submitting cvv during activation |

- Response:

```json
{
  "code": 0,
  "msg": "string",
  "result": {
    "card_type_id":"72000001",
    "card_no":"4360396720000016920",
    "acct_no":"test20240529",
    "status":1
  }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| acct_no | String | Institution-side user number (unique on institution side) |
| card_type_id | String | Card type ID |
| card_no | int | Card ID |
| status | int | Status code: 0 Frozen, 1 Activation successful, 2 Unactivated, 3. Activation pending review, 4. Activation review failed, 5. Application failed (card is being produced, please apply again later) 6. Cancelled |

### Query All Card Statuses

```text
url: /api/v1/debit-cards
method: GET
```

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| page_num | int | Optional | Page number |
| page_size | int | Optional | Page size |
| former_time | long | Optional | Start time, UNIX timestamp, `in seconds` |
| latter_time | long | Optional | End time, UNIX timestamp, `in seconds` |
| time_sort | String | Optional | Time sorting, asc for ascending, desc for descending |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "total": 1,
        "records": [
            {
                "acct_no": "1",
                "card_type_id":"72000001",
                "card_no": "4385211202597301",
                "status": 4,
                "reason": "{\"code\":1110061,\"msg\":\"Activation failure, hand holding card is not plastic card\"}",
                "create_time": 1576847136000
            }
        ]
    }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| acct_no | String | Institution-side user number (unique on institution side) |
| card_type_id | String | Card type ID |
| card_no | int | Bank card number |
| status | int | Status code: 0 Frozen, 1 Activation successful, 2 Unactivated, 3. Activation pending review, 4. Activation review failed, 5. Application failed (card is being produced, please apply again later) 6. Cancelled |
| reason | String | Reason for activation failure. Empty string in other cases |
| create_time | long | Creation time |

### Query Specific User's All Card Statuses

```text
url: /api/v1/debit-cards?acct_no={acct_no}
method: GET
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| acct_no | String | Required | Institution-side user number (unique on institution side) |
| page_num | int | Optional | Page number |
| page_size | int | Optional | Page size |
| former_time | long | Optional | Start time, UNIX timestamp, `in seconds` |
| latter_time | long | Optional | End time, UNIX timestamp, `in seconds` |
| time_sort | String | Optional | Time sorting, asc for ascending, desc for descending |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "total": 1,
        "records": [
            {
                "acct_no": "1",
                "card_type_id":"72000001",
                "card_no": "4385211202597301",
                "status": 4,
                "reason": "{\"code\":1110061,\"msg\":\"Activation failure, hand holding card is not plastic card\"}",
                "create_time": 1576847136000
            }
        ]
    }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| acct_no | String | Institution-side user number (unique on institution side) |
| card_type_id | String | Card type ID |
| card_no | int | Bank card ID |
| status | int | Status code: 0 Frozen, 1 Activation successful, 2 Unactivated, 3. Activation pending review, 4. Activation review failed, 5. Application failed (card is being produced, please apply again later) |
| reason | String | Reason for activation failure. Empty string in other cases |
| create_time | long | Creation time |

## Deposit to Card

### Deposit to User Card

```text
Internal deduction url: /api/v1/deposit-transactions
External deduction url: /api/v1/deposit-transactions/external-deduction
method: POST
```

- Request:

| Parameter | Type | Requirement | Description |
| :-----------: | :----: | :---------: | :-------------- |
| card_no | String | Required | Bank card ID |
| acct_no | String | Required | Institution-side user number (unique on institution side) |
| amount | String | Required | Amount of the corresponding deposit currency |
| coin_type | String | Required | Currency type. Only supports USDT, RUSD |
| cust_tx_id | String | Required | Institution's transaction serial number |
| remark | String | Optional | Transaction remark |
| card_currency | String | Optional | Card currency, required only for dual-currency cards |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "tx_id": "2020022511324811001637548",
        "coin_type": "USDT",
        "tx_amount": "0.92",         
        "exchange_fee_rate": "0",
        "exchange_fee": "0",
        "loading_fee": "0.0812",        
        "deposit_usdt": "0.9188",
        "currency_type": "USD",
        "currency_amount": "0.92",
        "exchange_rate": "1.00239251357",
        "fiat_exchange_rate": "1"
    }
}
```

| Parameter | Type | Description |
| :------------: | :----------: |:---------- |
| tx_id | String | Transaction serial id |
| coin_type | String | Deposit currency |
| tx_amount | String | Amount corresponding to the deposit currency |
| deposit_usdt | String | Amount of coin_type deposited for the user after deducting fees, unit is coin_type |
| exchange_fee_rate | String | Fee rate for exchanging deposit currency to coin_type |
| exchange_fee | String | Fee for exchanging deposit currency to USDT, unit is coin_type |
| loading_fee | String | Deposit fee, unit is coin_type |
| currency_amount | String | Fiat amount received |
| currency_type | String | Fiat type received |
| exchange_rate | String | coin_type/USD exchange rate |
| fiat_exchange_rate | String | Card supported fiat/USD exchange rate |

> If coin_type is USDT, the USDT fee deducted from the institution = exchange_fee + loading_fee + deposit_usdt.

### Query Status of a Specific Card Deposit Transaction

```text
url: /api/v1/deposit-transactions/{tx_id}/status
method: GET
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| tx_id | String | Required | Transaction serial tx_id or cust_tx_id |

- Response:

```json
{
  "code": 0,
  "msg": "string",
  "result": {
                "cust_tx_id": "1223",
                "acct_no": "03030062",
                "card_no": "8993152800000013334",
                "cust_tx_time": 1584350913000,
                "tx_id": "2020031609283339501898843",
                "coin_type": "USDT",
                "tx_amount": "61.86",     
                "exchange_fee": "0",
                "loading_fee": "1.8558",
                "currency_type": "USD",                
                "currency_amount": "60",
                "exchange_rate": "1.00239251357",
                "fiat_exchange_rate": "1",
                "tx_status": 3,
                "coin_price": "1"
  }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| cust_tx_id | String | Institution serial number |
| acct_no | String | Institution-side user number (unique on institution side) |
| card_no | int | Bank card ID |
| cust_tx_time | long | Creation time |
| tx_id | String | Transaction id |
| coin_type | int | Deposit currency |
| tx_amount | String | Deposit amount |
| exchange_fee | String | Fee for exchanging deposit currency to USDT, unit is `coin_type` |
| loading_fee | String | Deposit fee, unit is `coin_type` |
| currency_type | String | Fiat type received |
| currency_amount | String | Fiat amount received |
| exchange_rate | String | USDT/USD exchange rate |
| fiat_exchange_rate | String | Card supported fiat/USD exchange rate |
| tx_status | int | Transaction status. 0, 3, 4: Pending processing, 1: Deposit successful, 2: Deposit failed, 5: Deposit failed |
| coin_price | String | coin_type/USD exchange rate |

### Query All Card Deposit Records

```text
url: /api/v1/deposit-transactions
method: GET
```

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| page_num | int | Optional | Page number |
| page_size | int | Optional | Page size |
| former_time | long | Optional | Start time, UNIX timestamp, `in seconds` |
| latter_time | long | Optional | End time, UNIX timestamp, `in seconds` |
| time_sort | String | Optional | Time sorting, asc for ascending, desc for descending |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "total": 1,
        "records": [
            {
                "cust_tx_id": "1223",
                "acct_no": "03030062",
                "card_no": "8993152800000013334",
                "cust_tx_time": 1584350913000,
                "tx_id": "2020031609283339501898843",
                "coin_type": "USDT",
                "tx_amount": "61.86",     
                "exchange_fee": "0",
                "loading_fee": "1.8558",
                "currency_type": "USD",                
                "currency_amount": "60",
                "exchange_rate": "1.00239251357",
                "fiat_exchange_rate": "1",
                "tx_status": 3,
                "coin_price": "1"
            }
        ]
    }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| cust_tx_id | String | Institution serial number |
| acct_no | String | Institution-side user number (unique on institution side) |
| card_no | int | Bank card ID |
| cust_tx_time | long | Creation time |
| tx_id | String | Transaction id |
| coin_type | int | Deposit currency |
| tx_amount | String | Deposit amount |
| exchange_fee | String | Fee for exchanging deposit currency to USDT, unit is `coin_type` |
| loading_fee | String | Deposit fee, unit is `coin_type` |
| currency_type | String | Fiat type received |
| currency_amount | String | Fiat amount received |
| exchange_rate | String | USDT/Fiat exchange rate |
| fiat_exchange_rate | String | Card supported fiat/USD exchange rate |
| tx_status | int | Transaction status. 0, 3, 4: Pending processing, 1: Deposit successful, 2, 5: Deposit failed |
| coin_price | String | coin_type/USD exchange rate |

### Query Specific User's All Card Deposit Records

```text
url: /api/v1/deposit-transactions?acct_no={acct_no}
method: GET
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| acct_no | String | Required | Unique id number of the institution user |
| page_num | int | Optional | Page number |
| page_size | int | Optional | Page size |
| former_time | long | Optional | Start time, UNIX timestamp, `in seconds` |
| latter_time | long | Optional | End time, UNIX timestamp, `in seconds` |
| time_sort | String | Optional | Time sorting, asc for ascending, desc for descending |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "total": 1,
        "records": [
            {
                "cust_tx_id": "1223",
                "acct_no": "03030062",
                "card_no": "8993152800000013334",
                "cust_tx_time": 1584350913000,
                "tx_id": "2020031609283339501898843",
                "coin_type": "USDT",
                "tx_amount": "61.86",     
                "exchange_fee": "0",
                "loading_fee": "1.8558",
                "currency_type": "USD",                
                "currency_amount": "60",
                "exchange_rate": "1.00239251357",
                "fiat_exchange_rate": "1",
                "tx_status": 3,
                "coin_price": "1"
            }
        ]
    }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| cust_tx_id | String | Institution serial number |
| acct_no | String | Institution-side user number (unique on institution side) |
| card_no | int | Bank card ID |
| cust_tx_time | long | Creation time |
| tx_id | String | Transaction id |
| coin_type | int | Deposit currency |
| tx_amount | String | Deposit amount |
| exchange_fee | String | Fee for exchanging deposit currency to USDT, unit is `coin_type` |
| loading_fee | String | Deposit fee, unit is `coin_type` |
| currency_type | String | Fiat type received |
| currency_amount | String | Fiat amount received |
| exchange_rate | String | USDT/USD exchange rate |
| fiat_exchange_rate | String | Card supported fiat/USD exchange rate |
| tx_status | int | Transaction status. 0, 3, 4: Pending processing, 1: Deposit successful, 2: Deposit failed, 5: Deposit failed |
| coin_price | String | coin_type/USD exchange rate |

## Bank Card Query

Provides interfaces for bank card consumption records, etc.

### Query if Card is Activated

```text
url: /api/v1/bank/account-status
method: POST
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| card_no | String | Required | Bank card ID |

- Response:

```json
{
  "code": 0,
  "msg": "string",
  "result": true
}
```

### Query Card Balance

```text
url: /api/v1/bank/balance
method: POST
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| card_no | String | Required | Bank card ID |
| card_currency | String | Optional | Card currency, required only for dual-currency cards |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "card_number": "438521******2001",
        "card_type": "EGEN BLUE",
        "current_balance": "121.12454",
        "current_balance_usd": "121.12454",
        "available_balance": "10.23",
        "available_balance_usd": "10.23"
    }
}
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| card_number | String | Real bank card number |
| card_type | String | Bank card type |
| current_balance | String | Current balance (card supported currency) |
| current_balance_usd | String | Current balance (USD) |
| available_balance | String | Available balance (card supported currency) |
| available_balance_usd | String | Available balance (USD) |

### Query Card Statement

```text
url: /api/v1/bank/transaction-record
method: POST
```

- Request:

| Parameter | Type | Requirement | Description |
| :------------: | :----: | :----------: |:---------- |
| card_no | String | Required | Bank card ID |
| card_currency | String | Optional | Card currency, required only for dual-currency cards |
| former_month_year | String | Required | Specify the month to query (format "012020") |
| latter_month_year | String | Required | Specify the month to query (format "012020") |
| tx_id | String | Optional | tx_id, transaction ID |

- Response:

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "result": [
      {
      	  "month_year":"022020",
          "statement_cycle_date": "28/11/2019",
          "opening_balance": "0.00",
          "opening_balance_usd": "0.00",
          "closing_balance": "150.55",
          "closing_balance_usd": "150.55",
          "available_balance": "N/A",
          "available_balance_usd": "N/A",
          "bank_tx_list": [
              {
                  "transaction_date": "20/11/2019",
				  "reason":"",
                  "posting_date": "20/11/2019",
                  "tx_id": "54675678678",                  
                  "description": "MONTHLY FEE",
                  "debit": "2.50",
                  "debit_usd": "2.50",
                  "credit": "",
                  "credit_usd": "",
				  "transaction_time": 1757570339000,
                  "fee": "0",
                  "end_bal": "end_bal", // If 0, means no post-transaction balance
                  "origin_transaction_id": "",
                  "type": 1
              },
              {
                  "transaction_date": "28/11/2019",
				  "reason":"",
                  "posting_date": "28/11/2019",
                  "tx_id": "54675678677",
                  "description": "MONTHLY FEE",
                  "debit": "2.50",
                  "debit_usd": "2.50",
                  "credit": "",
                  "credit_usd": "",
				  "transaction_time": 1757570339000,
                  "fee": "0",
                  "end_bal": "end_bal", // If 0, means no post-transaction balance
                  "origin_transaction_id": "",
                  "type": 1
              }
          ]
      }
    ]
 }   
```

| Parameter | Type | Description |
| :--------: | :----: | :------------------------------ |
| month_year | String | Date, MMyyyy |
| statement_cycle_date | String | Statement generation date |
| opening_balance | String | Opening balance (card supported currency) |
| opening_balance_usd | String | Opening balance (USD) |
| closing_balance | String | Closing balance (card supported currency) |
| closing_balance_usd | String | Closing balance (USD) |
| available_balance | String | Available balance (card supported currency) |
| available_balance_usd | String | Available balance (USD) |
| bank_tx_list[n] | Object | Transaction list |
| bank_tx_list[0].transaction_date | String | Transaction date |
| bank_tx_list[0].transaction_time | long | Transaction date timestamp |
| bank_tx_list[0].posting_date | String | Transaction posting date |
| bank_tx_list[0].tx_id | String | Transaction ID |
| bank_tx_list[0].description | String | Description |
| bank_tx_list[0].debit | String | Consumption amount (card supported currency) |
| bank_tx_list[0].debit_usd | String | Consumption amount (USD) |
| bank_tx_list[0].credit | String | Deposit amount (card supported currency) |
| bank_tx_list[0].credit_usd | String | Deposit amount (USD) |
| bank_tx_list[0].fee | String | Fee, only some cards have a value. |
| bank_tx_list[0].type | int | Transaction type, 1. Consumption 2. Deposit 3. Withdrawal 4. Transfer (in) 5. Transfer (out) 6. Other 7. Settlement adjustment 8. Refund 9. Consumption failed 10. verification (card binding verification transaction), 11. void (cancel, cancel after payment, return funds) |
| bank_tx_list[0].reason | String | Reason for transaction failure |
| bank_tx_list[0].tx_currency | String | Actual transaction currency |
| bank_tx_list[0].tx_amount | String | Transaction amount in actual transaction currency |
| bank_tx_list[0].end_bal | String | Post-transaction balance. If 0, means no post-transaction balance |
| bank_tx_list[0].origin_transaction_id | String | Original consumption transaction id, only refund transactions may have a value |

### Query Card Sensitive Information

- Request:

```text
Virtual card url: /api/v1/bank/virtualcard, Physical card and metal card url: /api/v1/bank/card
method: GET or POST
```

| Parameter | Type | Whether Required | Description |
| :---------: | :---: | :--------------: | :-------------------------------------------------------- |
| card_no | String | Required | card no |
| publickey | String | Optional | We will use this public key instead of the default public key to encrypt data, supports ECIES and RSA |

- Response:

```text
ECIES:
{
    "code": 0,
    "msg": "SUCCESS",
    "result": {
        "encrypt_data": [
            "f48ef2667c8e2d1f80c1e44408099fe7",
            "048ce3a29ac8586e8e7291983ba6ea192d0144c2dec5db147fb9d74ebfc635642bd2051c2932c059e271aab3109c507b92cc2275e4ff03e2548e75a741f6531e58",
            "78263b48508ad00b80552be97c4f74e963de0e722ded84a2c926256b74397b25030ec96f500eeb0bae9d8475c0665b35"
        ],
        "public_key": "031194af2b8ad8cba709509a630dfcc3746c24dfbbe9af48264df5663ad308e16f"
    }
}

RSA:
{
    "code":0,
    "msg":"SUCCESS",
    "result":{
        "encrypt_data":[
            "FwLBNaIiXynp7XagRab3HmQPDnaUv3yXv4ZPzvgJ0MWmJg4I7qug/fcVL/kQNeOrYcDQRycM8teN7l+XoBin4AlOVeHNxYEOxnmuLDWcWl88y18D7JFtvQfBHn+KQ96r8B/IFopmUFbqWKexOdLIp/hPwKHpsJv5Thu3t/JrWDk="],
        "public_key":"MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCDtxc+w3LlhtF3DcbgS8zqvjYs8Z+s03DrtbQb8iigeMAbKjXLbrWZwX6IprmVRandLuM3PV3UU7L4SCdJZBRUjgEe0q3F/kUe1XwFhsl4aoW9FHKK/d/em50qHSLXw/KPX2HyZ9Ey9G/bV6OwJXvkUYfStBjHNgPlwFca17QhuQIDAQAB"
    }
}

Physical card decryption:
{"card_number":"123456789101212"}

Virtual card decryption:
{"cvv":"123","card_number":"1001022400001101","expire":"022022"}

A card virtual card decryption:
{"cvv":"***","card_number":"****************","expire":"******","password":"343545"}

```
| Parameter | Type | Description |
| :---------: | :----: | :--------------------------- |
| encrypt_data | String[] | Encrypted data |
| public_key | String | Public key |

## Error Codes

### Business Logic Error Codes

| Status Value | Description |
| :--------------: | --------|
| 0 | Success |
| 111001 | Request parameter error |
| 111002 | KYC status exception |
| 111003 | KYC failed |
| 111004 | KYC duplicate |
| 111005 | Cannot find corresponding KYC |
| 111006 | Cannot find corresponding institution |
| 111007 | Institution status exception |
| 111008 | Cannot find corresponding institution asset configuration |
| 111009 | Institution asset status exception |
| 111010 | Cannot find corresponding institution configuration |
| 111011 | Insufficient institution balance |
| 111012 | Cannot find corresponding card |
| 111013 | Card status exception |
| 111014 | Insufficient cards of this type |
| 111015 | The bank card has been frozen |
| 111016 | Cannot find corresponding card type ID |
| 111017 | The card has already been issued |
| 111018 | Online banking is not activated |
| 111019 | Card ID duplicate |
| 111020 | User can only open one card of each type |
| 111021 | The institution does not have permission to open this type of card |
| 111022 | Cannot find corresponding transaction |
| 111023 | No permission to query this transaction |
| 111024 | Email format is invalid |
| 111025 | Email verification code error |
| 111026 | Email duplicate |
| 111027 | This email provider is not supported |
| 111028 | Mobile number duplicate |
| 111029 | Mobile verification code error |
| 111030 | Cannot get bank data |
| 111031 | Cannot find corresponding bank |
| 111032 | Cannot find corresponding configuration |
| 111033 | Birthday parameter cannot be empty |
| 111034 | Cannot upload photo |
| 111035 | Query time is not allowed to exceed one month |
| 111036 | Time format error |
| 111037 | Queried bank statement time is not allowed to exceed 6 months |

### Identity and Permission Authentication Error Codes

| Status Value | Description |
| :--------------: | --------|
| 112001 | Request timeout |
| 112002 | Illegal permission |
| 112003 | Illegal IP address |
| 112004 | Illegal timestamp |
| 112005 | Signature verification failed |
| 112006 | Authentication format error |
| 112007 | Signature error |
| 112008 | Cannot find corresponding app key |
| 112009 | Invalid app key secret |
| 112010 | Request header error |

### Exception Error Codes

| Status Value | Description |
| :----: | --------- |
| 119001 | Service unavailable |
| 119002 | Communication error |
| 119003 | Data encryption error |
| 119004 | Data decryption error |
| 119005 | API called too frequently |
| 119006 | The API is not authorized |
| 119007 | Public key format error |

### KYC Failure Error Codes

| Status Value | Description |
| :-----: | ------------------------------- |
| 111003 | KYC failed |
| 1110031 | KYC failed, country or nationality filled incorrectly |
| 1110032 | KYC failed, document photo error |
| 1110033 | KYC failed, photo holding document error or not clear enough |
| 1110034 | KYC failed, address error |
| 1110035 | KYC failed, mobile number error |
| 1110036 | KYC failed, birthday error |
| 1110037 | KYC failed, name error |
| 1110038 | KYC failed, zip code error |
| 1110039 | KYC failed, other error |
| 1110040 | KYC failed, passport expired or about to expire |
| 1110041 | KYC failed, passport incomplete or unclear |
| 1110042 | KYC failed, gender filled incorrectly |
| 1110043 | KYC failed, photo holding document unclear or incomplete |
| 1110044 | KYC failed, text on passport in photo holding document is reversed |
| 1110045 | KYC failed, face not shown in photo holding document |

The failure reason is filled in the `reason` parameter of the KYC result, content like:

```json
{
   "code": 1110032,
   "msg": "KYC failure, photo error"
}
```

### Card Activation Failure Error Codes

| Status Value | Description |
| :--------------: | --------|
| 1110060 | Activation failed |
| 1110061 | Activation failed, hand holding card in activation photo is not a plastic card |
| 1110062 | Activation failed, no portrait in activation photo |
| 1110063 | Activation failed, passport or card information in activation photo is incomplete or unclear |
| 1110064 | Activation failed, activation photo error, chip groove not covered by finger |
| 1110065 | Activation failed, activation photo is black and white, need to take color photo |
| 1110066 | Activation failed, text on passport in activation photo is unclear |

The failure reason is filled in the `reason` parameter of the activation result, content like:

```json
{
   "code": 1110061,
   "msg": "Activation failure, hand holding card is not plastic card"
}
```
