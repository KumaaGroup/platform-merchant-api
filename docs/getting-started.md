# Getting Started

The Platform Merchants API lets you accept payments through a hosted payments page, process refunds, initiate open banking transfers, and manage webhooks — all through a single REST API.

## Base URL

| Environment | URL                                                        |
|-------------|------------------------------------------------------------|
| Sandbox     | `https://sandbox-merchants-api.nonprod.paygate.systems`    |

> **Warning — Sandbox Data Policy:** The sandbox environment is strictly for testing with **synthetic (fictitious) data only**. The use of real personally identifiable information (PII) or real cardholder data (CHD) is **forbidden**. Always use the provided [test card numbers](card-payments.md#testing), fabricated names, addresses, and email addresses. Violating this policy may result in account suspension.

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

- Monitor transactions and payment activity
- Invite additional users to the dashboard

> **Note:** MFA is enabled by default for all users and cannot be disabled. API credentials are provided via email during onboarding and cannot be retrieved from the portal.

## Quick Start

### 1. Obtain an access token

Exchange your credentials for a Bearer token using the OAuth2 client credentials flow.

```bash
curl -X POST https://sandbox-auth.nonprod.paygate.systems/oauth2/token \
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

### 2. Whitelist the customer's card

Card whitelisting is **mandatory**: register each card and allow approximately 72 hours for the cooldown period to expire before it can be used to pay. See [Card Whitelist](blocklist-and-whitelist.md#card-whitelist) for details.

### 3. Initialize a payment

Use the access token to initialize a payment. You never collect card data yourself — the response contains an `actionUrl` for the hosted payments page where your customer completes the payment.

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/payment/crypto/initialize \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "externalId": "order-001",
    "customerEmail": "jane@example.com",
    "customerFirstName": "Jane",
    "customerLastName": "Doe",
    "currency": "EUR",
    "amount": 29.99,
    "successUrl": "https://your-shop.com/checkout/success",
    "failureUrl": "https://your-shop.com/checkout/failure"
  }'
```

Response:

```json
{
  "id": "pay_abc123",
  "externalId": "order-001",
  "status": "INITIALIZED",
  "actionUrl": "https://sandbox-topup.nonprod.kumaacrypto.com/cryptopublic/payments/page/eyJhbGciOi..."
}
```

Redirect your customer to the `actionUrl` (valid for 15 minutes). The full flow — including the wallet top-up step — is described in [Crypto Payments](crypto-payments.md).

> **Deprecated:** the direct card endpoints (`POST /payment`, `POST /payment/batch`, `POST /payment/crypto`, `POST /payment/google-pay`, `POST /payment/apple-pay`, `POST /payment/ptc`) are being phased out. New integrations must use the initialize flow above.

### 4. Check the payment

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/payment/record/pay_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The payment record includes its status, the per-method `attempts` the customer made, and the wallet-transfer fields used for settlement — see [Payment Records and Attempts](crypto-payments.md#payment-records-and-attempts).

### 5. Set up webhooks

Receive real-time notifications when the payment and the wallet top-up progress.

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

Create a second webhook with `eventType: WALLET_TRANSFER` to be notified when the customer's wallet top-up is captured — settlement is based on it (see [Crypto Payments](crypto-payments.md#webhooks)).

## Key Concepts

### External ID and Idempotency

Every payment and transaction requires a merchant-provided `externalId`. This identifier:

- Must be unique across your account
- Allows only alphanumeric characters, hyphens, and underscores (`^[a-zA-Z0-9_-]+$`)
- Is limited to 255 characters
- Prevents duplicate transactions — submitting the same `externalId` twice returns a conflict error

See [Idempotency](idempotency.md) for more details.

### Payment Lifecycle

A payment record moves through the following states:

```
INITIALIZED → PENDING → COMPLETED
           ↘         ↘ DECLINED
```

Each payment-method attempt within the record (for example the card payment) has its own sub-lifecycle — see [Crypto Payments — Payment Lifecycle](crypto-payments.md#payment-lifecycle). Refunds, open banking transactions, and push-to-card disbursements each have their own state machines, described on their pages.

### Error Format

Errors are returned as JSON with a `message` describing the problem; request-validation failures additionally pinpoint the offending fields. See [Error Handling](error-handling.md) for the exact shapes.

```json
{
  "message": "duplicate payment request"
}
```

## Next Steps

- [Authentication](authentication.md) — Token lifecycle, refresh strategy, and IP whitelisting
- [Idempotency](idempotency.md) — How `externalId` prevents duplicate transactions
- [Crypto Payments](crypto-payments.md) — The hosted-page payment flow with merchant wallet top-ups
- [Card Payments](card-payments.md) — Payment statuses, deprecated direct endpoints, and test cards
- [Refunds](refunds.md) — Full and partial refund processing
- [Open Banking](open-banking.md) — Bank transfer transactions
- [Webhooks](webhooks.md) — Event notifications setup
- [Error Handling](error-handling.md) — Error codes and troubleshooting
- [Blocklist and Whitelist](blocklist-and-whitelist.md) — Managing blocked customers and allowed cards
