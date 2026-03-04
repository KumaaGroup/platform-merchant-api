# Webhooks

Webhooks notify your server in real time when events occur — such as a payment being completed or an open banking transfer finishing. Instead of polling the API, you register a URL and the platform sends HTTP POST requests to it whenever a relevant event happens.

## Create a Webhook

```bash
curl -X POST https://sandbox-merchants-api.nonprod.paygate.systems/webhooks \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/payments",
    "eventType": "CARD_PAYMENT",
    "enabled": true,
    "headers": [
      { "name": "X-Webhook-Secret", "value": "your-secret-value" }
    ]
  }'
```

### Request Fields

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `url`       | string  | Yes      | HTTPS endpoint to receive notifications (max 2048 chars) |
| `eventType` | string  | Yes      | Event type: `CARD_PAYMENT` or `OPEN_BANKING`      |
| `enabled`   | boolean | Yes      | Whether the webhook is active                     |
| `headers`   | array   | No       | Custom headers sent with each notification (max 5) |

### Custom Headers

You can attach up to 5 custom headers to each webhook. These are included in every notification request. Use them for authentication or routing.

| Field   | Type   | Description                        |
|---------|--------|------------------------------------|
| `name`  | string | Header name (max 256 characters)   |
| `value` | string | Header value (max 4096 characters) |

### Response (201 Created)

```json
{
  "id": "wh_abc123",
  "url": "https://your-server.com/webhooks/payments",
  "eventType": "CARD_PAYMENT",
  "enabled": true,
  "headers": [
    { "name": "X-Webhook-Secret", "value": "your-secret-value" }
  ],
  "createdAt": "2026-03-04T12:00:00Z",
  "modifiedAt": "2026-03-04T12:00:00Z"
}
```

### Constraints

- Only **one webhook per event type** per merchant. If you need to handle both card payments and open banking events, create two separate webhooks.
- The URL **must use HTTPS**. Plain HTTP endpoints are rejected.

## Event Types

| Event Type     | Triggered When                                          |
|----------------|---------------------------------------------------------|
| `CARD_PAYMENT` | A card payment status changes (authorized, captured, declined, refunded, etc.) |
| `OPEN_BANKING` | An open banking transaction status changes (in transit, completed, failed, etc.) |

## List Webhooks

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/webhooks \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Returns all webhooks configured for your merchant account.

## Get a Webhook

```bash
curl https://sandbox-merchants-api.nonprod.paygate.systems/webhooks/wh_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## Update a Webhook

Use PATCH to partially update a webhook. Only include the fields you want to change.

```bash
curl -X PATCH https://sandbox-merchants-api.nonprod.paygate.systems/webhooks/wh_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": false
  }'
```

### Updatable Fields

| Field     | Type    | Description                              |
|-----------|---------|------------------------------------------|
| `url`     | string  | New HTTPS endpoint URL                   |
| `enabled` | boolean | Enable or disable the webhook            |
| `headers` | array   | Replace custom headers (max 5)           |

The `eventType` cannot be changed after creation. To switch event types, delete the webhook and create a new one.

## Delete a Webhook

```bash
curl -X DELETE https://sandbox-merchants-api.nonprod.paygate.systems/webhooks/wh_abc123 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Returns `204 No Content` on success.

## Best Practices

- **Verify the source.** Use custom headers (e.g. a shared secret) to verify that incoming requests are from the platform, not a third party.
- **Respond with 2xx quickly.** Your webhook endpoint should return a `200` or `202` status code promptly. Perform any heavy processing asynchronously after acknowledging receipt.
- **Handle duplicates.** Your endpoint may receive the same event more than once. Use the payment or transaction `id` to deduplicate.
- **Use HTTPS with a valid certificate.** Self-signed certificates are not supported.
