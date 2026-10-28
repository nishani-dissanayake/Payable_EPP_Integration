# Payable IPG - Easy Payment Plan (EPP)

This guide walks you through offering Easy Payment Plans on your checkout: list available plans for an amount, prepare installment figures, then start a one-time card payment with the plan the customer chose.

Call the list and prepare APIs from your server. Keep your **EPP Token** and **Merchant Token** on the server and use them only to build check values.

---

## Prerequisites

1. Ask PAYable to enable EPP plans for your merchant account.
2. In the Merchant Portal, go to **Settings → EPP Integration** and copy your **EPP Token** (this is a secret).
3. Confirm the plans and banks you need are active for your merchant portal.

You will also need:

- **Merchant Key** — Portal → Settings → API Integration
- **Merchant Token** — Portal → Settings → API Integration (used at checkout; this is a secret)
- **EPP Token** — Portal → Settings → EPP Integration (used for list and prepare endpoints; this is a secret)

EPP applies only when:

- **`paymentType`** is **`1`** (one-time card payment).
- **`currencyCode`** is **`LKR`**.

**Note:** EPP cannot be used together with **card discounts** (`discountId`) or **pre-authorization** (`paymentOperation: "AUTHORIZE"`).

---

## List EPP plans

Load plans available for a given transaction amount.

**Sandbox**: `POST https://sandboxipgpayment.payable.lk/ipg/epp/list`

**Live**: `POST https://ipgpayment.payable.lk/ipg/epp/list`

**Note:** Use the sandbox URL while you build and test. Switch to the live URL when you go live. Use the merchant key and EPP token that belong to that same environment.

**Headers**:

```
Content-Type: application/json
```

**Required parameters**:

- `merchantKey` — your merchant key.
- `amount` — transaction amount with two decimal places, for example `50000.00`. Minimum `2.00`.
- `currencyCode` — `LKR` only.
- `checkValue` — the security hash generated as described below.

### Check value (list)

Build the check value on your server, in this order, with `|` between each part:

```
UPPERCASE(SHA512[merchantKey|amount|currencyCode|UPPERCASE(SHA512[eppToken])])
```

```javascript
const crypto = require("crypto");

function sha512(value) {
  return crypto.createHash("sha512").update(String(value), "utf8").digest("hex").toUpperCase();
}

function buildEppListCheckValue(merchantKey, amount, currencyCode, eppToken) {
  const tokenHash = sha512(eppToken);
  return sha512(merchantKey + "|" + amount + "|" + currencyCode + "|" + tokenHash);
}
```

### Sample request

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "amount": "10000.00",
  "currencyCode": "LKR",
  "checkValue": "YOUR_CHECK_VALUE"
}
```

```bash
curl -X POST "https://sandboxipgpayment.payable.lk/ipg/epp/list" \
  -H "Content-Type: application/json" \
  -d "{\"merchantKey\":\"YOUR_MERCHANT_KEY\",\"amount\":\"10000.00\",\"currencyCode\":\"LKR\",\"checkValue\":\"YOUR_CHECK_VALUE\"}"
```

### Success response

A successful call returns **HTTP 200**. The JSON body lists EPP plans for the amount you sent. Each entry in **`eppList`** includes an **`eppConfigId`**, bank details, tenure, and installment figures you can show on your site.

```json
{
  "apikey": "YOUR_MERCHANT_KEY",
  "eppCount": 1,
  "eppList": [
    {
      "eppConfigId": "DFCC-03",
      "bankCode": "DFCC",
      "bankName": "DFCC Bank",
      "tenorMonths": 3,
      "eppModel": "ALL_IN_ONE",
      "nicRequired": true,
      "interestFeeRate": "1.000",
      "fixedFeeAmount": "1500.00",
      "transactionAmount": "10000.00",
      "interestAmount": "100.00",
      "totalPayableAmount": "11600.00",
      "monthlyInstallmentAmount": "3366.67",
      "firstInstallmentAmount": "4866.67",
      "lastInstallmentAmount": "3366.66",
      "currencyCode": "LKR",
      "eligibleBins": null,
      "collectMobile": true,
      "collectEmail": true,
      "termsVersion": "IPG-EPP-TC-1.0",
      "termsText": null
    }
  ],
  "checkValue": "RESPONSE_CHECK_VALUE"
}
```

**Note:** It is recommended to validate the **`checkValue`** in the response before trusting the list.

#### Response check value (list)

Response **`checkValue`** is uppercase hex SHA-512, keyed with your **EPP Token**.

```
secretHash = UPPERCASE(SHA512[eppToken])

checkValue = UPPERCASE(SHA512[plain + secretHash])
```

- **`plain`** always ends with `|`.
- Amounts in other EPP fields use two decimal places, rounded half-up (for example `1500.00`).

**Plain text for list:**

```
plain = apikey|eppCount|
```

- **`apikey`** — trimmed merchant key from the response.
- **`eppCount`** — number of entries in **`eppList`** (integer, no decimals).

Example: `MK123|2|`

```javascript
function verifyEppListResponseCheckValue(apikey, eppCount, responseCheckValue, eppToken) {
  const secretHash = sha512(eppToken);
  const plain = String(apikey).trim() + "|" + String(eppCount) + "|";
  const expected = sha512(plain + secretHash);
  return expected === String(responseCheckValue).toUpperCase();
}
```

Use **`eppConfigId`** from the plan the customer selects for the prepare call and for checkout.

---

## Prepare EPP plan

Validate the chosen plan and get installment amounts (interest, monthly installment, total payable) for the amount you will charge.

**Sandbox**: `POST https://sandboxipgpayment.payable.lk/ipg/epp/prepare`

**Live**: `POST https://ipgpayment.payable.lk/ipg/epp/prepare`

**Headers**:

```
Content-Type: application/json
```

**Required parameters**:

- `merchantKey` — your merchant key.
- `eppConfigId` — plan id from the list response.
- `amount` — same format and value as for list (two decimal places).
- `currencyCode` — `LKR`.
- `checkValue` — the security hash generated as described below.

### Check value (prepare)

```
UPPERCASE(SHA512[merchantKey|eppConfigId|amount|currencyCode|UPPERCASE(SHA512[eppToken])])
```

```javascript
function buildEppPrepareCheckValue(merchantKey, eppConfigId, amount, currencyCode, eppToken) {
  const tokenHash = sha512(eppToken);
  return sha512(
    merchantKey + "|" + eppConfigId + "|" + amount + "|" + currencyCode + "|" + tokenHash,
  );
}
```

### Sample request

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "eppConfigId": "PABC-03",
  "amount": "25000.00",
  "currencyCode": "LKR",
  "checkValue": "YOUR_CHECK_VALUE"
}
```

```bash
curl -X POST "https://sandboxipgpayment.payable.lk/ipg/epp/prepare" \
  -H "Content-Type: application/json" \
  -d "{\"merchantKey\":\"YOUR_MERCHANT_KEY\",\"eppConfigId\":\"PABC-03\",\"amount\":\"25000.00\",\"currencyCode\":\"LKR\",\"checkValue\":\"YOUR_CHECK_VALUE\"}"
```

### Success response

A successful call returns **HTTP 200**. Confirm the plan, card schemes, and installment figures before you start checkout.

```json
{
  "apikey": "Merchant key",
  "eppConfigId": "PABC-03",
  "bankName": "Pan Asia Bank",
  "tenorMonths": 3,
  "eppModel": "ALL_IN_ONE",
  "nicRequired": true,
  "paymentTypes": [
    {
      "scheme": "MASTERCARD"
    }
  ],
  "validBins": [
    "222300",
    "XXXXXXXX",
    "XXXXXXXX",
    "XXXXXXXX"
  ],
  "finalCardCharge": "25000.00",
  "interestAmount": "225.00",
  "monthlyInstallmentAmount": "8408.33",
  "firstInstallmentAmount": "8408.33",
  "totalPayableAmount": "25225.00",
  "currencyCode": "LKR",
  "eppCheckValue": "EPP_CHECK_VALUE",
  "checkValue": "CHECK_VALUE"
}
```

**Note:** It is recommended to validate **`checkValue`** and **`eppCheckValue`** in the response before trusting the prepare result.

#### Response check values (prepare)

Uppercase hex SHA-512 with **`secretHash = UPPERCASE(SHA512[eppToken])`** and **`checkValue = UPPERCASE(SHA512[plain + secretHash])`**. **`plain`** always ends with `|`.

**Prepare — `checkValue`**

```
plain = apikey|eppConfigId|monthlyInstallmentAmount|
```

- Use **`monthlyInstallmentAmount`** exactly as returned (two decimal places).

Example: `MK123|PABC-03|8408.33|`

**Prepare — `eppCheckValue`**

```
plain = apikey|eppConfigId|finalCardCharge|
```

- **`finalCardCharge`** is the transaction amount (two decimal places, same format as list/prepare **`amount`**).

Example: `MK123|PABC-03|25000.00|`

```javascript
function verifyEppPrepareCheckValues(body, eppToken) {
  const secretHash = sha512(eppToken);
  const key = String(body.apikey).trim();
  const cfg = String(body.eppConfigId);
  const plainCheck = key + "|" + cfg + "|" + body.monthlyInstallmentAmount + "|";
  const plainEpp = key + "|" + cfg + "|" + body.finalCardCharge + "|";
  return {
    checkValueOk: sha512(plainCheck + secretHash) === String(body.checkValue).toUpperCase(),
    eppCheckValueOk: sha512(plainEpp + secretHash) === String(body.eppCheckValue).toUpperCase(),
  };
}
```

For checkout, send **`eppConfigId`**, **`tenorMonths`**, **`interestAmount`**, **`monthlyInstallmentAmount`**, and **`totalPayableAmount`** from this response using the same decimal strings (two decimal places). Use **`paymentTypes`** and **`validBins`** on your site to tell the customer which cards qualify. When **`nicRequired`** is `true`, **`customerNic`** is required at checkout.

Your checkout **`amount`** should match the transaction amount you used for list and prepare.

---

## Use EPP at checkout

After the customer selects a plan, accepts terms, and you have their contact details, start payment with your usual one-time checkout fields **plus** the EPP fields below.

Send these **additional** fields when using EPP (**required**):

- `eppConfigId` — from list / prepare.
- `tenorMonths` — integer from prepare.
- `interestAmount` — from prepare, two decimal places.
- `monthlyInstallmentAmount` — from prepare, two decimal places.
- `totalPayableAmount` — from prepare, two decimal places.
- `termsAccepted` — must be `true`.
- `customerNic` — 12 digits, or 9 digits followed by `V`or `v`.
- `customerMobilePhone` — 10–15 characters; digits, `+`, and spaces only.
- `customerEmail` — valid email address.

Example (other mandatory checkout fields omitted for brevity):

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "invoiceId": "Inv001",
  "paymentType": 1,
  "currencyCode": "LKR",
  "amount": "50000.00",
  "checkValue": "YOUR_CHECK_VALUE",
  "notifyUrl": "https://your-domain.com/payable/webhook",
  "returnUrl": "https://your-domain.com/payable/return",
  "eppConfigId": "SEYL-06",
  "tenorMonths": 6,
  "interestAmount": "1250.00",
  "monthlyInstallmentAmount": "8541.67",
  "totalPayableAmount": "51250.00",
  "termsAccepted": true,
  "customerNic": "199012345678",
  "customerMobilePhone": "0771234567",
  "customerEmail": "customer@example.com"
}
```

A successful session create returns **`uid`**, **`statusIndicator`**, and **`paymentPage`**. Send the customer to **`paymentPage`** to complete payment on the hosted card page.

---

## Check value at checkout (one-time payment with EPP)

For checkout with EPP, extend the one-time payment hash with the EPP segments. Use the **exact** strings you send in the JSON body. For `termsAccepted`, use the literal string `true` in the hash when the JSON value is boolean `true`.

```
UPPERCASE(SHA512[
  merchantKey|invoiceId|amount|currencyCode|eppConfigId|tenorMonths|
  interestAmount|monthlyInstallmentAmount|totalPayableAmount|termsAccepted|
  customerNic|customerMobilePhone|customerEmail|
  UPPERCASE(SHA512[merchantToken])
])
```

**One-time payment without EPP** (for comparison):

```
UPPERCASE(SHA512[merchantKey|invoiceId|amount|currencyCode|UPPERCASE(SHA512[merchantToken])])
```

**Note:** Webhook payload shape and webhook check value rules are the same as for a standard one-time payment without EPP.

---

## Errors

### Validation error

Missing or badly formatted fields return **HTTP 400**. Each field can have one or more messages.

```json
{
  "status": 400,
  "errors": {
    "merchantKey": [
      "Merchant Key is a required field."
    ],
    "checkValue": [
      "Check Value is a required field."
    ]
  }
}
```

### Request failed

A rejected merchant key or check value on list or prepare usually returns **HTTP 404**. The `error` text describes what PAYable rejected.

```json
{
  "status": 404,
  "error": "Invalid authentication"
}
```

Check the following and try again:

1. **`merchantKey`** and tokens are from the same environment as the base URL. You request originates from the same domain used during Merchant Token generation.
2. List and prepare check values use **`eppToken`**; checkout check value uses **`merchantToken`**.
3. Hashes use uppercase hex and a single `|` between each part, in the order shown above.
4. **`amount`**, installment fields, and **`currencyCode`** match between prepare and checkout.
