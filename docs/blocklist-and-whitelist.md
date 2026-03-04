# Blocklist and Whitelist

The Platform Merchants API provides two complementary tools for managing which cards and customers can transact with your merchant account:

- **Customer Blocklist** — prevent transactions from specific cards or email addresses.
- **Card Whitelist** — restrict transactions to only pre-approved cards.

## Customer Blocklist

Block customers by card (BIN + last 4 digits) or by email address. Blocked customers cannot make payments through your merchant account.

### Block Customers

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/blocklist/customers \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "customers": [
      { "binCode": "411111", "last4": "1111" },
      { "email": "fraudster@example.com" }
    ]
  }'
```

Each entry must use **either** card identification (`binCode` + `last4`) **or** `email` — not both.

#### Request Fields

| Field      | Type   | Required | Description                                |
|------------|--------|----------|--------------------------------------------|
| `binCode`  | string | *        | Card BIN code (6–8 digits)                 |
| `last4`    | string | *        | Last 4 digits of the card number           |
| `email`    | string | *        | Customer email address                     |

*Use either `binCode` + `last4` or `email` per entry.

#### Constraints

- Maximum **100 customers** per request.

### List Blocked Customers

```bash
curl "https://sandbox-merchants-api.nonprod.paygate.systems/blocklist/customers?limit=20" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

#### Query Parameters

| Parameter | Type    | Default | Description                         |
|-----------|---------|---------|-------------------------------------|
| `limit`   | integer | 20      | Number of results per page (1–100)  |
| `cursor`  | string  | —       | Cursor for the next page of results |

#### Response

```json
{
  "items": [
    {
      "id": "blc_abc123",
      "binCode": "411111",
      "last4": "1111",
      "createdAt": "2026-03-04T12:00:00Z"
    },
    {
      "id": "blc_def456",
      "email": "fraudster@example.com",
      "createdAt": "2026-03-04T12:01:00Z"
    }
  ],
  "nextCursor": "eyJpZCI6ImJsX2RlZjQ1NiJ9"
}
```

### Get a Blocked Customer Entry

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/blocklist/customers/blc_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

---

## Card Whitelist

Depending on your merchant account configuration, payments may be processed in one of two modes:

- **Trusted mode** — all cards are accepted without prior registration. This is the default for most accounts.
- **Whitelist mode** — only pre-approved cards can be used for payments. Every card must be whitelisted before a payment can be processed with it.

If your account requires card whitelisting, you must call the card whitelist API for each card you intend to use for payments. An accepted whitelisting request triggers a **cooldown period of approximately 72 hours** before the card becomes active and can be used for an actual payment. During the cooldown, the `cooldownExpiresAt` field on the whitelist entry indicates when the card will be ready.

> **Important:** Attempting a payment with a card that has not been whitelisted (or is still in cooldown) will result in a declined payment.

You can check whether your account requires card whitelisting from the [Merchant Backoffice Portal](https://sandbox-backoffice.nonprod.paygate.systems).

### Whitelist Cards

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/whitelist/card \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "cards": [
      { "binCode": "411111", "last4": "1111" },
      { "binCode": "550000", "last4": "0004" }
    ]
  }'
```

#### Request Fields

| Field     | Type   | Required | Description                     |
|-----------|--------|----------|---------------------------------|
| `binCode` | string | Yes      | Card BIN code (6–8 digits)      |
| `last4`   | string | Yes      | Last 4 digits of the card number |

#### Constraints

- Maximum **100 cards** per request.

### List Whitelisted Cards

```bash
curl "https://sandbox-merchants-api.nonprod.paygate.systems/whitelist/card?limit=20" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

#### Query Parameters

| Parameter | Type    | Default | Description                         |
|-----------|---------|---------|-------------------------------------|
| `limit`   | integer | 20      | Number of results per page (1–100)  |
| `cursor`  | string  | —       | Cursor for the next page of results |

#### Response

```json
{
  "items": [
    {
      "id": "cwl_abc123",
      "binCode": "411111",
      "last4": "1111",
      "createdAt": "2026-03-04T12:00:00Z",
      "cooldownExpiresAt": null
    }
  ],
  "nextCursor": null
}
```

The `cooldownExpiresAt` field indicates when the cooldown period expires for a recently added card. During the cooldown period (approximately 72 hours), the whitelisted card is not yet active and cannot be used for payments. Once `cooldownExpiresAt` is `null` or in the past, the card is ready for use.

### Get a Whitelisted Card Entry

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/whitelist/card/cwl_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## Use Cases

### Fraud Prevention with Blocklist

- Block cards that have been used for chargebacks or fraudulent transactions.
- Block email addresses associated with suspicious activity.
- Proactively block known bad actors before they attempt a transaction.

### Controlled Access with Whitelist

- Restrict payments to a known set of corporate cards during a closed beta.
- Limit transactions to pre-verified cards for high-value merchants.
- Gradually roll out access by whitelisting cards in batches.
- Plan ahead for the 72-hour cooldown when onboarding new cards.
