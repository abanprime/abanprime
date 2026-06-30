# AbanPrime API Documentation

## Overview
The Futures API allows authenticated users to manage their futures trading positions, predict potential debts, manage orders, and retrieve market information.

# API Authentication

Authenticate your API requests by signing them with your private key. The server
keeps only your public key — no shared secret is ever sent.

---

## 1. Get your credentials

you can get your API key and API secret from your account -> profile -> API key management section.

---

## 2. Sign each request

Every request carries three headers. Two are plain values; the third,
`API-SIGNATURE`, is a cryptographic signature you compute over the contents of the
request. The server recomputes the same signature from your stored public key — if
the two match, the request is proven to come from you and to be unaltered.

### Required headers

| Header          | Value                                                       |
|-----------------|-------------------------------------------------------------|
| `API-KEY-ID`    | Your `keyId`.                                               |
| `API-TIMESTAMP` | Current time in Unix epoch **milliseconds** (e.g. `1718640000123`). |
| `API-SIGNATURE` | Base64 Ed25519 signature of the canonical message (below).  |

### What you sign: the canonical message

You don't sign the raw HTTP request. Instead you build one text string — the
**canonical message** — that uniquely describes the request, and you sign that. It
is the concatenation, in order and with **no separators between the parts**, of:

```
{timestamp}{METHOD}{path}{sha256_hex(body)}
```

Because the signature is computed over these four parts, changing any of them — the
time, the method, the URL path, or a single byte of the body — produces a different
signature and the request is rejected. This is what protects the request from being
replayed later or tampered with in transit.

### How to build and sign it, step by step

1. **Take the current timestamp** as Unix epoch milliseconds, e.g. `1718640000123`.
   Use this exact value in two places: the `API-TIMESTAMP` header, and the start of
   the message. They must be identical strings.

2. **Take the HTTP method in uppercase** — `GET`, `POST`, `PUT`, or `DELETE`.

3. **Take the request path**, starting with `/`, *without* the host and *without*
   the query string. For `https://api.abanprime.com/cross-margin/order?x=1` the path
   is `/cross-margin/order`.

4. **Hash the body.** Compute the SHA-256 of the request body and write it as
   **lowercase hexadecimal**. Hash the body exactly as you will send it — the same
   bytes, field order, and whitespace. If the request has no body (most `GET` and
   `DELETE` calls), hash the empty string; that hash is always
   `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`.

5. **Concatenate** the four parts into one string, with nothing between them.

6. **Sign** the UTF-8 bytes of that string with your Ed25519 **private key**.

7. **Base64-encode** the signature and put it in the `API-SIGNATURE` header.

Then send the request with the three headers set.

### Worked example

You can verify your implementation against this fixed example. The key below is for
testing only — **do not use it in production**:

| Field                  | Value                                          |
|------------------------|------------------------------------------------|
| Private key (base64)   | `AQIDBAUGBwgJCgsMDQ4PEBESExQVFhcYGRobHB0eHyA=`  |
| Public key (base64)    | `ebVWLo/mVPlAeLES6KmLp5AfhTrmlb7X4OORC60ElmQ=`  |

Signing this request:

| Part      | Value                            |
|-----------|----------------------------------|
| timestamp | `1718640000123`                  |
| method    | `POST`                           |
| path      | `/cross-margin/order`            |
| body      | `{"symbol":"BTCIRT","amount":1}` |

Step 4 — the body hash:

```
fdf9b043f745e3dea5cc04f92cfd24131d22914028009cec4d6093c743b38123
```

Step 5 — the canonical message:

```
1718640000123POST/cross-margin/orderfdf9b043f745e3dea5cc04f92cfd24131d22914028009cec4d6093c743b38123
```

Steps 6–7 — the signature, which becomes `API-SIGNATURE`:

```
bJOBJA/0vY1Wtd2qmKmxi1PHyK56RkYU8YXp+o/WRhSrnXIVBgzzmrTDKJGzY81kMgJNuYObVo19ERXkUEoMBQ==
```

If your code produces the same message string and the same signature for this key
and input, it is signing correctly. (Ed25519 is deterministic: the same key and
message always yield the same signature, so this doubles as a known-answer test.)

### Timing

Sign and send promptly. The server rejects any request whose timestamp is more than
**20 seconds** old, so keep your clock synced (e.g. via NTP).

The code samples in [section 3](#3-code-samples) do all of these steps for you.

---

## 3. Code samples

Each sample signs and sends `POST /cross-margin/order`. Replace the base URL,
`keyId`, and `privateKey` with your own.

### Python

```bash
pip install cryptography requests
```

```python
import base64, hashlib, time, requests
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey

BASE_URL = "https://api.abanprime.com"
KEY_ID = "your-key-id"
PRIVATE_KEY_B64 = "your-base64-private-key"

def signed_headers(method, path, body):
    timestamp = str(int(time.time() * 1000))  # epoch milliseconds
    body_hash = hashlib.sha256(body.encode()).hexdigest()
    message = f"{timestamp}{method.upper()}{path}{body_hash}"
    key = Ed25519PrivateKey.from_private_bytes(base64.b64decode(PRIVATE_KEY_B64))
    signature = base64.b64encode(key.sign(message.encode())).decode()
    return {
        "API-KEY-ID": KEY_ID,
        "API-TIMESTAMP": timestamp,
        "API-SIGNATURE": signature,
        "Content-Type": "application/json",
    }

path = "/cross-margin/order"
body = '{"symbol":"BTCIRT","amount":1}'
resp = requests.post(BASE_URL + path, data=body.encode(), headers=signed_headers("POST", path, body))
print(resp.status_code, resp.text)
```

### .NET (C#)

```bash
dotnet add package BouncyCastle.Cryptography
```

```csharp
using System.Security.Cryptography;
using System.Text;
using Org.BouncyCastle.Crypto.Parameters;
using Org.BouncyCastle.Crypto.Signers;

const string BaseUrl = "https://api.abanprime.com";
const string KeyId = "your-key-id";
const string PrivateKeyB64 = "your-base64-private-key";

string method = "POST", path = "/cross-margin/order";
string body = "{\"symbol\":\"BTCIRT\",\"amount\":1}";

string timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds().ToString(); // epoch milliseconds
string bodyHash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(body))).ToLowerInvariant();
string message = $"{timestamp}{method.ToUpperInvariant()}{path}{bodyHash}";

var key = new Ed25519PrivateKeyParameters(Convert.FromBase64String(PrivateKeyB64), 0);
var signer = new Ed25519Signer();
signer.Init(true, key);
var msg = Encoding.UTF8.GetBytes(message);
signer.BlockUpdate(msg, 0, msg.Length);
string signature = Convert.ToBase64String(signer.GenerateSignature());

using var http = new HttpClient();
var request = new HttpRequestMessage(HttpMethod.Post, BaseUrl + path)
{
    Content = new StringContent(body, Encoding.UTF8, "application/json")
};
request.Headers.Add("API-KEY-ID", KeyId);
request.Headers.Add("API-TIMESTAMP", timestamp);
request.Headers.Add("API-SIGNATURE", signature);

var response = await http.SendAsync(request);
Console.WriteLine((int)response.StatusCode);
Console.WriteLine(await response.Content.ReadAsStringAsync());
```

### Java

```xml
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcprov-jdk18on</artifactId>
    <version>1.78.1</version>
</dependency>
```

```java
import org.bouncycastle.crypto.params.Ed25519PrivateKeyParameters;
import org.bouncycastle.crypto.signers.Ed25519Signer;

import java.net.URI;
import java.net.http.*;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.Base64;

public class ApiSignatureClient {
    static final String BASE_URL = "https://api.abanprime.com";
    static final String KEY_ID = "your-key-id";
    static final String PRIVATE_KEY_B64 = "your-base64-private-key";

    public static void main(String[] args) throws Exception {
        String method = "POST", path = "/cross-margin/order";
        String body = "{\"symbol\":\"BTCIRT\",\"amount\":1}";

        String timestamp = Long.toString(System.currentTimeMillis()); // epoch milliseconds
        String bodyHash = sha256Hex(body);
        String message = timestamp + method.toUpperCase() + path + bodyHash;

        Ed25519PrivateKeyParameters key =
                new Ed25519PrivateKeyParameters(Base64.getDecoder().decode(PRIVATE_KEY_B64), 0);
        Ed25519Signer signer = new Ed25519Signer();
        signer.init(true, key);
        byte[] msg = message.getBytes(StandardCharsets.UTF_8);
        signer.update(msg, 0, msg.length);
        String signature = Base64.getEncoder().encodeToString(signer.generateSignature());

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(BASE_URL + path))
                .header("API-KEY-ID", KEY_ID)
                .header("API-TIMESTAMP", timestamp)
                .header("API-SIGNATURE", signature)
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(body, StandardCharsets.UTF_8))
                .build();

        HttpResponse<String> response =
                HttpClient.newHttpClient().send(request, HttpResponse.BodyHandlers.ofString());
        System.out.println(response.statusCode());
        System.out.println(response.body());
    }

    static String sha256Hex(String input) throws Exception {
        byte[] d = MessageDigest.getInstance("SHA-256").digest(input.getBytes(StandardCharsets.UTF_8));
        StringBuilder sb = new StringBuilder(d.length * 2);
        for (byte b : d) {
            sb.append(Character.forDigit((b >> 4) & 0xF, 16));
            sb.append(Character.forDigit(b & 0xF, 16));
        }
        return sb.toString();
    }
}
```

---

## 5. Common errors

| Message                        | Cause                                                         |
|--------------------------------|---------------------------------------------------------------|
| `Invalid signature`            | Method, path, or body don't match what was signed.            |
| `Timestamp expired`            | Request older than 20 seconds — check your clock.             |
| `Invalid timestamp`            | `API-TIMESTAMP` is not epoch milliseconds.                    |
| `API credential is not active` | The credential was revoked.                                   |
| Permission error               | The action needs a scope your credential wasn't granted.      |



### 1. Get Server Time

This endpoint returns the current server timestamp.

*   **Endpoint:** `GET /public/api/time`
*   **Successful Response (`200 OK`):**
    ```json
    1678886400000 // Example: A long integer representing the server timestamp (e.g., Unix timestamp in milliseconds).
    ```


## Base URL
All API URLs listed in this documentation are relative to the following base URL: https://api.abanprime.com

---

## Orders

### `Order.OrderDto` Definition
The `Order.OrderDto` is a data transfer object used for creating and simulating orders. It contains the following properties:

*   `baseAssetSymbol` (string, required): The symbol of the base asset for the trading pair (e.g., "BTC").
*   `quoteAssetSymbol` (string, required): The symbol of the quote asset for the trading pair (e.g., "USDT").
*   `side` (enum: `Buy`, `Sell`, required): The direction of the order.
*   `type` (enum: `Limit`, `Market`, required): The type of order to place.
    *   `Market`: An order to buy or sell immediately at the current market price.
    *   `Limit`: An order to buy or sell at a specified price or better.
*   `quantity` (decimal, optional): The amount of the `baseAssetSymbol` to trade. Required for `Market` orders. Cannot be set with `quote`.
*   `quote` (decimal, optional): The amount of the `quoteAssetSymbol` to trade. Required for `Market` orders. Cannot be set with `quantity`.
*   `price` (decimal, optional): The price at which to execute the order. Required for `Limit` orders.
*   `track` (GUID, optional): A unique identifier to track the order. (Typically generated by the client if needed for tracking).

**Note on `quantity` and `quote`:** For market orders, you must specify either the `quantity` (amount of base asset) or the `quote` (amount of quote asset), but not both. For limit orders, `quantity` is typically used.

---

### 1. Place Order
`POST /cross-margin/order`

**Required scope:** `Order`

**Purpose:** Allows the authenticated user to create a new order (Market or Limit) which will be associated with their cross margin account.

**Request Body (`application/json`):**
A JSON object (`Order.OrderDto`) containing the order details.

**Example Request Body (Market Order - Buy BTC with 100 USDT):**

```{ "baseAssetSymbol": "BTC", "quoteAssetSymbol": "USDT", "side": "Buy", "type": "Market", "quote": 100, "marketType": "P2P" }```

**Response:**
Returns the complete `Order` object that was created, including its current state and other relevant details.

### 2. Simulate Order
Simulates an order without actually placing it.

`POST /cross-margin/order/simulate`

**Required scope:** `Order`

**Purpose:** Allows user to simulate an order to understand its potential outcome and effects on their cross margin account (e.g., estimated filled quantity, price, fees) before actual execution.

**Request Body (`application/json`):**
A JSON object (`Order.OrderDto`) containing the order details for simulation. The structure is the same as for `Create Order`.

**Example Request Body (Simulate Buy BTC with 100 USDT at market):**

```{ "baseAssetSymbol": "BTC", "quoteAssetSymbol": "USDT", "side": "Buy", "type": "Market", "quote": 100, "marketType": "P2P" }```


**Response:**
Returns the simulated `Order` object with updated information (e.g., estimated fill price, estimated filled quantity, estimated fees). This order will not be persisted.

### 3. Get Order by ID
Retrieves the details of a specific order by its integer ID.

`GET /cross-margin/order/{orderId:int}`

**Purpose:** Allows user to fetch the full details of a specific order they own.

**URL Parameters:**
*   `orderId` (integer, required): The unique integer identifier of the order.

**Response:**
Returns the complete `Order` object.

### 4. Get Order by Track
Retrieves the details of a specific order by its GUID (track ID).

`GET /cross-margin/order/{orderTrack:guid}`

**Purpose:** Allows the authenticated user to fetch the full details of a specific order they own using its globally unique identifier.

**URL Parameters:**
*   `orderTrack` (GUID, required): The unique GUID that tracks the order.

**Response:**
Returns the complete `Order` object if the authenticated user is the owner of the order.

### 5. Cancel Limit Order by ID
Cancels an open limit order by its integer ID.

`DELETE /cross-margin/order/{orderId:int}`

**Required scope:** `Order`

**Purpose:** Allows user to cancel an existing limit order they own before it is filled or partially filled. Only limit orders can be canceled this way.

**URL Parameters:**
*   `orderId` (integer, required): The unique integer identifier of the limit order to cancel.

**Response:**
Returns the updated `Order` object, reflecting its "Cancelled" status, if the cancellation was successful.

### 6. Cancel Limit Order by Track
Cancels an open limit order by its GUID (track ID).

`DELETE /cross-margin/order/{orderTrack:guid}`

**Required scope:** `Order`

**Purpose:** Allows user to cancel an existing limit order they own before it is filled or partially filled, using its track Id. Only limit orders can be canceled this way.

**URL Parameters:**
*   `orderTrack` (GUID, required): The unique GUID of the limit order to cancel.

**Response:**
Returns the updated `Order` object, reflecting its "Cancelled" status, if the cancellation was successful.

### 7. List Orders
Retrieves a paginated list of orders for the authenticated user, with filtering options.

`GET /cross-margin/orders`

**Purpose:** Provides a way for user to view all their active and historical orders, with optional filtering and pagination.

**Query Parameters:**
*   `pagination.page` (integer, optional, default: 0): The page number to retrieve (0-indexed).
*   `pagination.pageSize` (integer, optional, default: 20): The number of items per page.
*   `filter.type` (enum: `Limit`, `Market`, optional): Filters orders by their type.
*   `filter.state` (enum: `New`, `PartiallyFilled`, `Filled`, `Cancelled`, `Rejected`, optional): Filters orders by their current state.
*   `filter.side` (enum: `Buy`, `Sell`, optional): Filters orders by their side.
*   `filter.marketType` (enum: `P2P`, `OTC`, `Manual`, optional): Filters orders by the market type.
*   `filter.baseAssetSymbol` (string, optional): Filters orders by the base asset symbol (e.g., "BTC").
*   `filter.quoteAssetSymbol` (string, optional): Filters orders by the quote asset symbol (e.g., "USDT").
*   `filter.createdAtGte` (datetime, optional): Filters orders created on or after this timestamp (ISO 8601 format).
*   `filter.createdAtLte` (datetime, optional): Filters orders created on or before this timestamp (ISO 8601 format).

**Response:**
Returns an object containing a list of `Order` objects belonging to the authenticated user and pagination metadata.


#### 8. Change Order Price

*   **Endpoint:** `PUT /futures/order/{orderId:int}`
*   **Required scope:** `Order`
*   **Description:** Changes the price of an existing limit order.
*   **Parameters:**
    *   `orderId` (route, `int`): The ID of the limit order to modify.
    *   `changePriceDto` (body, `LimitOrder.ChangePriceDto`): Data transfer object containing the new price.
*   **Authentication:** Requires authentication.
*   **Responses:**
    *   `200 OK`: Returns the updated order object.
    *   `404 Not Found`: If the `orderId` does not correspond to an existing order.

---



## Market Data

### 1. Get Market Information
Retrieves detailed information about a specific trading market.

`GET /public/cross-margin/market/{baseSymbol}-{quoteSymbol}`

**Purpose:** Provides details about a specific trading pair, including its type (P2P, OTC, etc.). This endpoint is publicly accessible without authentication.

**URL Parameters:**
*   `baseSymbol` (string, required): The symbol of the base asset (e.g., "BTC").
*   `quoteSymbol` (string, required): The symbol of the quote asset (e.g., "USDT").

**Response:**
Returns the `Market` object, which contains information like current prices, trading rules, and potentially order book details for OTC markets.

---


## Market Positions

### 1. Get Market Position
Retrieves the market position for the authenticated user in a specific market.

`GET /cross-margin/market-position/{baseSymbol}-{quoteSymbol}`

**Purpose:** Allows the authenticated user to view their market position (e.g., open quantity, PNL, average entry price) for a given trading pair within their cross margin account.

**URL Parameters:**
*   `baseSymbol` (string, required): The symbol of the base asset (e.g., "BTC").
*   `quoteSymbol` (string, required): The symbol of the quote asset (e.g., "USDT").

**Response:**
Returns the `MarketPosition` object for user in the specified market. This object typically includes details like current exposure, Taker fee, Maker fee, realized and unrealized profit/loss, and other position-related metrics.

---


## Market Data


#### 1. Get Market Information

*   **Endpoint:** `GET /public/futures/market/{baseSymbol}-{quoteSymbol}`
*   **Description:** Retrieves market information for a given base and quote asset pair.
*   **Parameters:**
    *   `baseSymbol` (route, `string`): The symbol of the base asset (e.g., "BTC").
    *   `quoteSymbol` (route, `string`): The symbol of the quote asset (e.g., "USD").
*   **Authentication:** Anonymous access allowed (`AllowAnonymous`).
*   **Responses:**
    *   `200 OK`: Returns the market object.
    *   `404 Not Found`: If the market for the specified symbols and type does not exist.

---


#### 2. List Markets (General)

*   **Endpoint:** `GET /public/futures/markets`
*   **Description:** Retrieves a paginated, filterable, and searchable list of markets. If authenticated, filters by user; otherwise, retrieves public markets.
*   **Parameters:**
    *   `page` (query): Page number.
    *   `pageSize` (query): Page size.
*   **Responses:**
    *   `200 OK`: Returns a paginated list of market objects.

---


## `Bank Withdraw` API Documentation


### 1. List Bank Withdraw Requests

This endpoint allows a user to retrieve a paginated and filtered list of their bank withdrawal requests.

*   **Endpoint:** `GET /bank-withdraw/bank-withdraw-requests`
*   **Description:** Returns a list of bank withdrawal requests for the authenticated user, supporting pagination, filtering, and sorting.
*   **Query Parameters:**
    *   **`pagination` parameters (from `BankWithdrawRequest.EntityPagination`):**
        *   `pagination.page` (integer, optional): The page number to retrieve. Default is 1.
        *   `pagination.pageSize` (integer, optional): The number of items per page. Default is 10.
    *   **`filter` parameters (from `BankWithdrawRequest.BankWithdrawRequestFilter`):**
        *   `filter.startDate` (string, optional): Start date for filtering requests (ISO 8601 format, e.g., `2024-01-01T00:00:00Z`).
        *   `filter.endDate` (string, optional): End date for filtering requests (ISO 8601 format, e.g., `2025-01-01T00:00:00Z`).
        *   `filter.state` (string, optional): Filters by the state of the bank withdraw request (e.g., `New`, `Processing`, `Completed`, `Cancelled`).
        *   `filter.userTimeZone` (string, optional): The user's time zone (e.g., "America/New_York") to correctly interpret date/time filters.
        *   `filter.userId` (integer, automatically set by the API): The `userId` is automatically populated with the authenticated user's ID. Any provided `userId` in the query will be ignored.
        *   *Other relevant filter properties from `TransferRequest.TransferRequestFilter`.*
*   **Successful Response (`200 OK`):**
    ```json
    {
      "items": [
        {
          "id": 101,
          "userId": 456,
          "transferType": "BankWithdraw",
          "transferRequestState": "New",
          "amount": 500.00,
          "asset": {
            "id": 1,
            "symbol": "USD",
            "name": "United States Dollar"
          },
          "destinationAccount": {
            "id": 789,
            "iban": "IRXXXXXXXXXXXXXXXXXXXXXX",
            "ownerFullName": "User Full Name",
            "bankName": "Sample Bank"
          },
          "createdAt": "2025-09-05T15:30:00Z",
          "memo": "Withdrawal for personal use"
          // ... other fields of BankWithdrawRequest
        },
        {
          "id": 102,
          "userId": 456,
          "transferType": "BankWithdraw",
          "transferRequestState": "Completed",
          "amount": 200.00,
          "asset": {
            "id": 1,
            "symbol": "USD",
            "name": "United States Dollar"
          },
          "destinationAccount": {
            "id": 789,
            "iban": "IRXXXXXXXXXXXXXXXXXXXXXX",
            "ownerFullName": "User Full Name",
            "bankName": "Sample Bank"
          },
          "createdAt": "2025-09-04T10:00:00Z",
          "memo": null
          // ... other fields
        }
      ],
      "pagination": {
        "page": 1,
        "pageSize": 10,
        "totalItems": 2,
        "totalPages": 1
      }
    }
    ```

### 2. Create Bank Withdraw Request

This endpoint allows a user to initiate a new bank withdrawal request.

*   **Endpoint:** `POST /bank-withdraw/bank-withdraw-request`
*   **Required scope:** `FiatWithdrawal`
*   **Description:** Creates a new bank withdrawal request for the authenticated user.
*   **Request Body (`application/json`):**
    ```json
    {
      "assetSymbol": "IRT",       // Required: Symbol of the asset to withdraw (e.g., "USD", "EUR").
      "iban": "IRXXXXXXXXXXXXXXXXXXXXXX", // Required: The IBAN of the destination bank account.
      "amount": 500.00,           // Required: The amount to withdraw. Must be greater than 0.
      "paymentId": "OptionalPaymentId", // Optional: A reference or payment ID.
      "note": "Optional note about the withdrawal", // Optional: A note for the request.
      "trackId": "GUID_STRING" // Optional: A unique ID for tracking the request. If not provided, a new one will be generated.
      // "otpCode": "123456", // Required: One-Time Password if 2FA is enabled for the user.
    }
    ```
    *   **Note:**
        *   The `Fee` is calculated automatically by the system and any provided `fee` in the request body will be ignored.
        *   If Two-Factor Authentication (2FA) is enabled for the user, an `otpCode` will be required in the request body.
*   **Successful Response (`200 OK`):**
    ```json
    {
      "id": 103,
      "userId": 456,
      "transferType": "BankWithdraw",
      "transferRequestState": "New", // Initial state of the request
      "amount": 500.00,
      "asset": {
        "id": 1,
        "symbol": "USD"
      },
      "destinationAccount": {
        "id": 789,
        "iban": "IRXXXXXXXXXXXXXXXXXXXXXX",
        "ownerFullName": "User Full Name",
        "bankName": "Sample Bank"
      },
      "createdAt": "2025-09-06T11:00:00Z",
      "memo": "OptionalPaymentId",
      "fee": 5.00, // Calculated fee
      "newClientOrderId": "GUID_STRING", // TrackId
      "info": { "Note": "Optional note about the withdrawal" } // If note was provided
      // ... other fields of the created BankWithdrawRequest
    }
    ```
*   **Error Responses:**
    *   `400 Bad Request`: If `amount` is 0, or if required fields are missing, or if IBAN is invalid.
    *   `401 Unauthorized`: If the user is not authenticated or 2FA is required but `otpCode` is missing/invalid.

### 3. Preview Bank Withdraw Request

This endpoint allows a user to preview the details of a bank withdrawal request without actually creating it. This can be used to see calculated fees or other system-generated details.

*   **Endpoint:** `POST /bank-withdraw/bank-withdraw-request/preview`
*   **Required scope:** `FiatWithdrawal`
*   **Description:** Provides a preview of a bank withdrawal request based on the provided details, including estimated fees. The request is not persisted.
*   **Request Body (`application/json`):**
    ```json
    {
      "assetSymbol": "USD",
      "iban": "IRXXXXXXXXXXXXXXXXXXXXXX",
      "amount": 500.00,
      "paymentId": "OptionalPaymentId",
      "note": "Optional note about the withdrawal"
    }
    ```
*   **Successful Response (`200 OK`):**
    ```json
    {
      "userId": 456,
      "transferType": "BankWithdraw",
      "transferRequestState": "New", // This is a preview, so state is "New"
      "amount": 500.00,
      "asset": {
        "id": 1,
        "symbol": "USD"
      },
      "destinationAccount": {
        "id": 789,
        "iban": "IRXXXXXXXXXXXXXXXXXXXXXX",
        "ownerFullName": "User Full Name",
        "bankName": "Sample Bank"
      },
      "memo": "OptionalPaymentId",
      "fee": 5.00, // Estimated fee
      "newClientOrderId": "GUID_STRING", // A generated trackId for the preview
      "info": { "Note": "Optional note about the withdrawal" }
      // ... other fields of the previewed BankWithdrawRequest
    }
    ```

### 4. Get Single Bank Withdraw Request

This endpoint allows a user to retrieve the details of a specific bank withdrawal request by its ID.

*   **Endpoint:** `GET /bank-withdraw/bank-withdraw-request/{requestId:int}`
*   **Description:** Retrieves the full details of a specific bank withdrawal request. Users can only access their own requests unless they have admin privileges.
*   **Path Parameters:**
    *   `requestId` (integer, required): The unique identifier of the bank withdrawal request.
*   **Successful Response (`200 OK`):**
    ```json
    {
      "id": 101,
      "userId": 456,
      "transferType": "BankWithdraw",
      "transferRequestState": "Completed",
      "amount": 500.00,
      "asset": {
        "id": 1,
        "symbol": "USD",
        "name": "United States Dollar"
      },
      "destinationAccount": {
        "id": 789,
        "iban": "IRXXXXXXXXXXXXXXXXXXXXXX",
        "ownerFullName": "User Full Name",
        "bankName": "Sample Bank"
      },
      "createdAt": "2025-09-05T15:30:00Z",
      "memo": "Withdrawal for personal use",
      "fee": 5.00,
      "newClientOrderId": "some-guid-string",
      "info": null
      // ... full details of the BankWithdrawRequest
    }
    ```
*   **Error Responses:**
    *   `400 Bad Request`: If the requested bank withdrawal request does not belong to the authenticated user (and the user is not an admin).
    *   `404 Not Found`: If no bank withdrawal request with the given `requestId` exists.

# Transfer API


### 1. List Your Transfer Requests

**GET** `/transfer/transfer-requests`

List your own transfer requests (withdrawals, deposits, internal, etc).

**Query Parameters:**
| Name                 | Type    | Description                                    |
|----------------------|---------|------------------------------------------------|
| pagination           | object  | Pagination info (see below)                    |
| filter               | object  | Filtering options (see major fields)           |
| search               | object  | Search criteria (optional)                     |

**Example Filter Fields:**
- `AssetSymbol` (string): Only transfers involving this currency/asset symbol
- `State` (string): Transfer state (e.g. `New`, `Processing`, `Done`)
- `TransferRequestType` (string): Type (`Deposit`, `Withdraw`, etc)
- `CreatedAtGte` (date): Created after this date/time
- `CreatedAtLte` (date): Created before this date/time

**Example Pagination:**
- `Page`: which page to return (default is 1)
- `PageSize`: how many records per page (default is 20)

**Sample Call:**
**GET** `/transfer/transfer-requests?filter.AssetSymbol=USDT&pagination.Page=1&pagination.PageSize=20`


**Response:**
Returns paginated list of your transfer requests and statuses.

---
