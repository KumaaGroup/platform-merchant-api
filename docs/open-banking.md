# Open Banking

Open banking transactions allow your customers to pay directly from their bank account. The customer is redirected to a secure widget where they select their bank and authorize the transfer.

All open banking transactions are processed in **EUR**.

> **Note:** Testing open banking transactions is currently not available in the sandbox environment. The documentation below describes the production integration flow for reference.

## Create a Transaction

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/open-banking/transactions \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "externalId": "ob-order-001",
    "amount": 99.99,
    "sender": {
      "email": "jane@example.com",
      "name": "Jane Doe"
    },
    "successUrl": "https://your-site.com/payment/success",
    "metadata": {
      "orderId": "12345"
    }
  }'
```

### Request Fields

| Field               | Type   | Required | Description                                              |
|---------------------|--------|----------|----------------------------------------------------------|
| `externalId`        | string | Yes      | Your unique identifier ([details](idempotency.md))       |
| `amount`            | number | Yes      | Amount in EUR (minimum `0.01`)                           |
| `sender.email`      | string | Yes      | Customer email address                                   |
| `sender.name`       | string | Yes      | Customer full name                                       |
| `successUrl`        | string | Yes      | URL to redirect the customer after completing the payment |
| `metadata`          | object | No       | Key-value pairs for your own reference                   |

### Response

```json
{
  "id": "obt_abc123",
  "externalId": "ob-order-001",
  "status": "PENDING",
  "url": "https://widget.example.com/pay/obt_abc123"
}
```

| Field       | Type   | Description                                        |
|-------------|--------|----------------------------------------------------|
| `id`        | string | Platform-generated transaction ID                  |
| `externalId`| string | Your provided identifier                           |
| `status`    | string | Initial status (always `PENDING`)                  |
| `url`       | string | Widget URL — redirect the customer here to pay     |

## Integration Flow

```
1. Create transaction via POST /open-banking/transactions
2. Redirect customer to the widget URL returned in the response
3. Customer selects their bank and authorizes the transfer
4. Customer is redirected to your successUrl
5. Receive webhook notification when the transfer completes (or poll for status)
```

## Transaction Lifecycle

```
PENDING → IN_TRANSIT → COMPLETED
                     ↘ FAILED
        ↘ CANCELLED
```

| Status       | Description                                       |
|--------------|---------------------------------------------------|
| `PENDING`    | Transaction created, awaiting customer action     |
| `IN_TRANSIT` | Bank transfer initiated, funds in transit         |
| `COMPLETED`  | Funds received successfully                       |
| `FAILED`     | Transfer failed                                   |
| `CANCELLED`  | Transaction was cancelled                         |

## Get Transaction Details

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/open-banking/transactions/obt_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### Response

```json
{
  "id": "obt_abc123",
  "externalId": "ob-order-001",
  "status": "COMPLETED",
  "amount": 99.99,
  "currency": "EUR",
  "sender": {
    "email": "jane@example.com",
    "name": "Jane Doe"
  },
  "successUrl": "https://your-site.com/payment/success",
  "iban": "DE89370400440532013000",
  "createdAt": "2026-03-04T12:00:00Z",
  "modifiedAt": "2026-03-04T12:05:00Z",
  "finishedAt": "2026-03-04T12:05:00Z",
  "metadata": {
    "orderId": "12345"
  }
}
```

The `iban` field is populated once the customer selects their bank account.

## List Transactions

```bash
curl "https://sandbox-merchants-api.nonprod.paygate.systems/open-banking/transactions?limit=20" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### Query Parameters

| Parameter | Type    | Default | Description                         |
|-----------|---------|---------|-------------------------------------|
| `limit`   | integer | 20      | Number of results per page (1–100)  |
| `cursor`  | string  | —       | Cursor for the next page of results |

## Webhooks

To receive real-time notifications when a transaction status changes, set up a webhook with event type `OPEN_BANKING`. See [Webhooks](webhooks.md) for setup instructions.
