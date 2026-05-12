# Google Pay

Accept payments through Google Pay without ever handling card data yourself. You initiate the payment server-side and we return a hosted button page, which you then embed on your checkout. A customer with a valid payment method in their Google Wallet taps the Google Pay button, completes the sheet, and the platform takes over — the rest follows the standard card-payment workflow, including 3D Secure when required.

## Integration Flow

```mermaid
sequenceDiagram
    participant M as Merchant (server)
    participant API as Merchants API
    participant Page as Hosted Button Page
    participant C as Customer
    participant GP as Google Pay

    M->>API: POST /payment/google-pay (amount, customer, urls)
    API-->>M: 200 status=TOKEN_REQUESTED, actionUrl
    M->>Page: Embed actionUrl on checkout
    C->>Page: Load button page
    Note over Page: Geoip check (same as 3DS).<br/>Redirect to blocked page if unsupported.
    Page-->>C: Render Google Pay button
    C->>Page: Tap Google Pay button
    Page->>GP: Open Google Pay sheet
    C->>GP: Authorize with saved payment method
    GP-->>Page: paymentMethodData
    Page->>API: POST callback
    Note over API: Resume payment workflow<br/>(same as card payments)
    alt 3DS required
        API-->>M: Webhook: status=AUTH_REQUESTED (actionUrl)
        M->>C: Redirect to 3DS actionUrl
        C->>API: Complete 3DS challenge
    end
    API-->>M: Webhook: status=CAPTURED
```

Two checks run inside the `TOKEN_REQUESTED` window before a Google Pay payment ever reaches the standard card flow:

- **Location check.** When the customer loads the button page, we run a geoip check on their IP — the same approach used for [3DS challenges](card-payments.md#3d-secure-3ds), just performed earlier in the lifecycle. If their location is unsupported, the page redirects to a blocked-location page and the payment is declined with `RESTRICTED_COUNTRY` — the Google Pay button is never shown.
- **Authorization window.** If the customer doesn't complete the Google Pay sheet within the allowed time, the payment is declined with `GOOGLE_PAY_EXPIRED`.

Once the platform receives the callback from the hosted page, the payment behaves exactly like a regular [card payment](card-payments.md) — same lifecycle, same webhooks, same 3DS handling.

## Create a Google Pay Payment

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/payment/google-pay \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "externalId": "gp-order-001",
    "currency": "EUR",
    "amount": 29.99,
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
    },
    "successUrl": "https://your-site.com/payment/success",
    "failureUrl": "https://your-site.com/payment/failure"
  }'
```

Notice the request body has **no `card` object** — that's the whole point. Card data never touches your server. Everything else (customer, billing address, currency, amount, `externalId`, redirect URLs, metadata) follows the same shape as a regular [card payment](card-payments.md#create-a-payment). See the API reference for the full field list.

### Response

```json
{
  "id": "pay_abc123",
  "externalId": "gp-order-001",
  "status": "TOKEN_REQUESTED",
  "actionUrl": "https://sandbox-merchants-api.nonprod.paygate.systems/public/payments/google-pay/button/eyJhbGciOi..."
}
```

| Field        | Type   | Description                                                                |
|--------------|--------|----------------------------------------------------------------------------|
| `id`         | string | Platform-generated payment ID                                              |
| `externalId` | string | Your provided identifier                                                   |
| `status`     | string | Initial payment status (always `TOKEN_REQUESTED`)                          |
| `actionUrl`  | string | URL of the hosted Google Pay button page — render this on your checkout    |

## Rendering the Button

The `actionUrl` returned by the create call points to a self-contained, statically rendered page that:

1. Loads the Google Pay JS library
2. Renders the official Google Pay button
3. Opens the Google Pay payment sheet when the customer taps it
4. Hands the customer's authorization back to the platform, which then resumes the payment

**Embed it as an iframe on your checkout page:**

```html
<iframe
  src="https://sandbox-merchants-api.nonprod.paygate.systems/public/payments/google-pay/button/eyJhbGciOi..."
  title="Google Pay"
  style="border: 0; width: 240px; height: 48px;"
  allow="payment">
</iframe>
```

> **Important — `allow="payment"` is mandatory.** Browsers gate the Payment Request API (which Google Pay is built on top of) behind the `payment` Permissions-Policy feature. In a cross-origin iframe like this one, the feature is **disabled by default**: without `allow="payment"` on the `<iframe>` tag, the Google Pay sheet will silently fail to open when the customer taps the button, and your checkout will appear broken with no visible error. Treat this attribute as part of the integration, not a styling choice.

You do **not** need to include the Google Pay JS SDK, set up a merchant ID with Google, or implement the `onPaymentAuthorized` handler — the hosted page does all of that. You only need to render the iframe (with `allow="payment"`) and wait for webhook updates on the payment.

> **Note:** The `actionUrl` is single-use and bound to the payment ID. It expires when the payment leaves the `TOKEN_REQUESTED` state (after the customer pays, after the timeout window, or once we determine the customer's location is unsupported).

## Payment Lifecycle

A Google Pay payment starts in `TOKEN_REQUESTED` and transitions into the regular card-payment lifecycle as soon as the customer authorizes on the Google Pay sheet.

```mermaid
stateDiagram-v2
    [*] --> TOKEN_REQUESTED
    TOKEN_REQUESTED --> REQUESTED: customer authorized
    TOKEN_REQUESTED --> DECLINED: timeout / blocked location
    REQUESTED --> AUTH_REQUESTED: 3DS challenge required
    REQUESTED --> AUTHORIZED: no 3DS, authorized
    REQUESTED --> DECLINED: validation failed or no acquirer
    AUTH_REQUESTED --> AUTHORIZED: 3DS approved
    AUTH_REQUESTED --> DECLINED: 3DS failed / blocked / timed out
    AUTHORIZED --> CAPTURED
    AUTHORIZED --> DECLINED
    CAPTURED --> [*]
    DECLINED --> [*]
```

| Status            | Description                                                                                |
|-------------------|--------------------------------------------------------------------------------------------|
| `TOKEN_REQUESTED` | Hosted button page is live; the platform is waiting for the customer to authorize Google Pay |
| `REQUESTED`       | Customer authorized; standard card-payment processing has begun                            |
| `AUTH_REQUESTED`  | Customer must complete a 3DS challenge (see [3D Secure](card-payments.md#3d-secure-3ds))    |
| `AUTHORIZED`      | Card authorized, funds reserved                                                            |
| `CAPTURED`        | Funds captured (terminal, success)                                                         |
| `DECLINED`        | Payment declined (terminal) — inspect `responseCode` for the reason                        |

### Google Pay–specific decline reasons

When a payment is declined while still in `TOKEN_REQUESTED`, the `responseCode` field tells you why:

| `responseCode`        | When it is set                                                                  |
|-----------------------|---------------------------------------------------------------------------------|
| `GOOGLE_PAY_EXPIRED`  | The customer did not complete the Google Pay sheet within the allowed window.   |
| `RESTRICTED_COUNTRY`  | The customer's detected location is not supported for Google Pay (checked at button-page load, before the Google Pay sheet is shown). |
| `INTERNAL_ERROR`      | The platform could not recover a usable payment method from the Google Pay sheet. |

In all three cases, the payment transitions directly from `TOKEN_REQUESTED` to `DECLINED`, a `CARD_PAYMENT` webhook is fired, and no card-side processing happens. The customer can retry by creating a new payment.

Declines that occur **after** the customer has authorized (validation failures, 3DS failures, issuer declines, etc.) follow exactly the same conventions as regular [card payments](card-payments.md#payment-lifecycle).

## Webhooks

Google Pay payments emit the standard `CARD_PAYMENT` webhook events — there is no separate event type, and the rules are identical to regular [card payments](card-payments.md). Webhooks fire on the key actionable transitions only:

- `AUTH_REQUESTED` — when a 3DS challenge is required (carries the `actionUrl` to redirect the customer to).
- `CAPTURED` — terminal, success.
- `DECLINED` — terminal, failure. Inspect `responseCode` for the reason.

Intermediate transitions such as `TOKEN_REQUESTED → REQUESTED` (when the customer authorizes on the Google Pay sheet) and `REQUESTED → AUTHORIZED` (when the issuer approves without 3DS) do **not** emit their own webhook. If you need to observe them, poll `GET /payment/{id}`. See [Webhooks](webhooks.md) for setup instructions.

## Get Payment Details

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/payment/pay_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Google Pay payments are returned by the same [`GET /payment/{id}`](card-payments.md#get-payment-details) endpoint as card payments. Once the customer authorizes, the response includes the masked card details (`binCode`, `last4`, `issuer`) of the underlying payment method that Google Pay returned.

## Testing

To test Google Pay in the **sandbox** environment:

1. Use a Google account that is enrolled in the [Google Pay TEST environment](https://developers.google.com/pay/api/web/guides/test-and-deploy/integration-checklist) and has a test card saved in its Google Wallet.
2. Open the `actionUrl` iframe in a Chrome browser on a device that supports Google Pay.
3. Tap the Google Pay button and complete the sheet.
4. Observe the payment transition through `TOKEN_REQUESTED → REQUESTED → ... → CAPTURED` via the `CAPTURED` webhook, or by polling `GET /payment/{id}`.

> **Warning:** Only **synthetic (fictitious) data** may be used in the sandbox environment. The use of real personally identifiable information (PII) or real cardholder data (CHD) is **strictly forbidden**.

### Simulating decline outcomes

| To simulate…                  | Do this                                                                                  |
|-------------------------------|------------------------------------------------------------------------------------------|
| `GOOGLE_PAY_EXPIRED`          | Create a payment and do not interact with the button page; wait for the window to elapse |
| `RESTRICTED_COUNTRY`          | Open the `actionUrl` from an IP in an unsupported region — the page short-circuits at load time and redirects to the blocked-location page before the Google Pay button is rendered |
| 3DS challenge / failure / issuer declines | Use one of the test cards listed in [Card Payments → Testing](card-payments.md#testing) as the payment method saved in your Google Wallet |
