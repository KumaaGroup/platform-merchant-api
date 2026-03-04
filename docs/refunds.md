# Refunds

You can refund a completed card payment either fully or partially. Refunds are processed against the original payment using its platform-generated `id`.

## Create a Refund

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/payment/pay_abc123/refund \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "externalId": "refund-order-001",
    "amount": 10.00
  }'
```

### Request Fields

| Field        | Type   | Required | Description                                              |
|--------------|--------|----------|----------------------------------------------------------|
| `externalId` | string | Yes      | Your unique identifier for this refund ([details](idempotency.md)) |
| `amount`     | number | No       | Amount to refund (minimum `0.01`). Omit for a full refund. |

### Full vs. Partial Refund

- **Full refund** — omit the `amount` field. The entire payment amount is refunded.
- **Partial refund** — provide an `amount` less than or equal to the original payment amount.

> **Note:** Partial refunds are not always available. Depending on the payment processing path, some payments only support full refunds. If you request a partial refund on a payment that does not support it, the API returns an error. When this happens, you can request a full refund instead.

### Response

**200 OK** — refund processed immediately:

```json
{
  "id": "ref_def456",
  "paymentId": "pay_abc123",
  "amount": 10.00,
  "createdAt": "2026-03-04T12:00:00Z",
  "status": "COMPLETED"
}
```

**201 Created** — refund submitted for processing:

```json
{
  "id": "ref_def456",
  "paymentId": "pay_abc123",
  "amount": 10.00,
  "createdAt": "2026-03-04T12:00:00Z",
  "status": "REQUESTED"
}
```

### Response Fields

| Field       | Type   | Description                              |
|-------------|--------|------------------------------------------|
| `id`        | string | Platform-generated refund ID             |
| `paymentId` | string | ID of the original payment               |
| `amount`    | number | Refund amount                            |
| `createdAt` | string | Timestamp of refund creation (ISO 8601)  |
| `status`    | string | Refund status                            |

### Status Codes

| Code | Meaning                                             |
|------|-----------------------------------------------------|
| 200  | Refund processed successfully                       |
| 201  | Refund submitted for processing                     |
| 422  | Refund cannot be processed (e.g. exceeds payment amount) |

## Checking Refund Status

Refunds are returned as part of the payment details. Use the refund `id` to query its status:

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/payment/ref_def456 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The response includes a `parentPaymentId` field linking the refund back to the original payment, and `type` will indicate it is a refund.

## Refund Lifecycle

```
REQUESTED → PENDING ──────────────→ APPROVED → COMPLETED
          → PENDING_APPROVAL → APPROVED ↗   ↘ DECLINED
          ↘ INVALID              ↘ REJECTED
```

| Status             | Description                                                                                     |
|--------------------|-------------------------------------------------------------------------------------------------|
| `REQUESTED`        | Refund submitted, initial validation in progress                                                |
| `PENDING`          | Validation passed, refund auto-approved (within threshold)                                      |
| `PENDING_APPROVAL` | Refund requires manual approval by platform administration (threshold exceeded or auto-approval disabled) |
| `APPROVED`         | Refund approved, being sent to the payment processor                                            |
| `COMPLETED`        | Refund successfully processed (terminal)                                                        |
| `DECLINED`         | Refund declined by the payment processor (terminal)                                             |
| `REJECTED`         | Refund rejected during manual review (terminal)                                                 |
| `INVALID`          | Refund failed initial validation, e.g. duplicate `externalId` or invalid amount (terminal)      |

> **Note:** When querying refund status, a `PENDING_APPROVAL` state means the refund is waiting for the platform administration to review and approve it. No action is required from the merchant — the administration team will process the approval.

## Best Practices

- **Use a unique `externalId` per refund** to ensure idempotency. If you retry a refund request with the same `externalId`, you'll receive a `409 Conflict` rather than a duplicate refund.
- **Check the original payment status** before requesting a refund. Refunds can only be issued against payments that have been captured or completed.
- **Track partial refunds carefully.** The total of all partial refunds must not exceed the original payment amount.
