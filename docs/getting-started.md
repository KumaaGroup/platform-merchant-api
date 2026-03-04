# Getting Started

The Platform Merchants API lets you accept card payments, process refunds, initiate open banking transfers, and manage webhooks — all through a single REST API.

## Base URL

| Environment | URL                                                        |
|-------------|------------------------------------------------------------|
| Sandbox     | `https://sandbox-merchants-api.nonprod.paygate.systems`    |

## Prerequisites

Before you begin, make sure you have:

- A merchant account with API credentials (`client_id` and `client_secret`)
- HTTPS capability for receiving webhooks

### Obtaining Your Credentials

API credentials are available only after your merchant account has been successfully onboarded. To start the onboarding process, visit:

**[https://sandbox-backoffice.nonprod.paygate.systems/onboarding/register](https://sandbox-backoffice.nonprod.paygate.systems/onboarding/register)**

Once your account is approved and active, you can access the **Merchant Backoffice Portal** at:

**[https://sandbox-backoffice.nonprod.paygate.systems](https://sandbox-backoffice.nonprod.paygate.systems)**

From the backoffice portal you can:

- Manage users and access permissions
- Enable multi-factor authentication (MFA) for your team
- View and retrieve your API credentials
- Monitor transactions and account activity

## Quick Start

### 1. Obtain an access token

Exchange your credentials for a Bearer token using the OAuth2 client credentials flow.

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "grant_type=client_credentials"
```

Response:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

See [Authentication](authentication.md) for full details on token management.

### 2. Create a payment

Use the access token to create a card payment.

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/payment \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "externalId": "order-001",
    "currency": "EUR",
    "amount": 29.99,
    "card": {
      "number": "4111111111111111",
      "name": "Jane Doe",
      "expiry": { "month": 12, "year": 2027 },
      "cvc": "123"
    },
    "customer": {
      "email": "jane@example.com",
      "firstName": "Jane",
      "lastName": "Doe",
      "billingAddress": {
        "address1": "123 Main St",
        "city": "Berlin",
        "country": "DEU",
        "state": "BE",
        "zip": "10115"
      }
    }
  }'
```

Response:

```json
{
  "id": "pay_abc123",
  "externalId": "order-001",
  "status": "AUTH_REQUESTED"
}
```

If 3D Secure is required, the response includes an `actionUrl`. Redirect the customer to that URL to complete authentication.

### 3. Check payment status

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/payment/pay_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 4. Set up a webhook (optional)

Receive real-time notifications when payment status changes.

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/webhooks \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/payments",
    "eventType": "CARD_PAYMENT",
    "enabled": true
  }'
```

## Key Concepts

### External ID and Idempotency

Every payment and transaction requires a merchant-provided `externalId`. This identifier:

- Must be unique across your account
- Allows only alphanumeric characters, hyphens, and underscores (`^[a-zA-Z0-9_-]+$`)
- Is limited to 255 characters
- Prevents duplicate transactions — submitting the same `externalId` twice returns a conflict error

See [Idempotency](idempotency.md) for more details.

### Payment Lifecycle

A card payment moves through the following states:

```
AUTH_REQUESTED → AUTHORIZED → CAPTURED → COMPLETED
                                      ↘ DECLINED
```

Additional states: `PENDING`, `PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `TRANSFERRED`.

### Error Format

All errors follow the [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) Problem Details format with content type `application/problem+json`.

```json
{
  "status": 422,
  "title": "Unprocessable Entity",
  "detail": "Payment was declined",
  "type": "https://example.com/errors/payment-declined",
  "instance": "/payment"
}
```

## Next Steps

- [Authentication](authentication.md) — Token lifecycle, refresh strategy, and IP whitelisting
- [Idempotency](idempotency.md) — How `externalId` prevents duplicate transactions
- [Card Payments](card-payments.md) — Full payment flows, batch payments, and push-to-card
- [Refunds](refunds.md) — Full and partial refund processing
- [Open Banking](open-banking.md) — Bank transfer transactions
- [Webhooks](webhooks.md) — Event notifications setup
- [Error Handling](error-handling.md) — Error codes and troubleshooting
- [Blocklist and Whitelist](blocklist-and-whitelist.md) — Managing blocked customers and allowed cards
