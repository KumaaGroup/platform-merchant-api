# Error Handling

The Platform Merchants API returns errors in the [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) Problem Details format. All error responses use the content type `application/problem+json`.

## Error Response Format

```json
{
  "status": 422,
  "title": "Unprocessable Entity",
  "detail": "Payment was declined by the issuing bank",
  "type": "https://api.paygate.systems/errors/payment-declined",
  "instance": "/payment",
  "errors": [
    {
      "location": "card.number",
      "message": "Card number is invalid",
      "value": "411111111111"
    }
  ]
}
```

### Fields

| Field      | Type   | Description                                                  |
|------------|--------|--------------------------------------------------------------|
| `status`   | integer| HTTP status code                                             |
| `title`    | string | Short human-readable summary of the problem                  |
| `detail`   | string | Specific explanation of what went wrong                      |
| `type`     | string | URI identifying the error type                               |
| `instance` | string | URI identifying the specific request that caused the error   |
| `errors`   | array  | Field-level validation errors (optional)                     |

### Validation Error Details

When the `errors` array is present, each entry pinpoints a specific field that failed validation:

| Field      | Type   | Description                           |
|------------|--------|---------------------------------------|
| `location` | string | Dot-notation path to the invalid field |
| `message`  | string | What is wrong with the field          |
| `value`    | string | The value that was submitted          |

## HTTP Status Codes

### Success Codes

| Code | Meaning     | Used By                                       |
|------|-------------|-----------------------------------------------|
| 200  | OK          | Successful GET, POST, PATCH requests          |
| 201  | Created     | Webhook created, refund submitted             |
| 204  | No Content  | Webhook deleted successfully                  |

### Client Error Codes

| Code | Meaning              | Common Causes                                              |
|------|----------------------|------------------------------------------------------------|
| 400  | Bad Request          | Malformed JSON, invalid field values, batch size exceeded   |
| 401  | Unauthorized         | Missing, expired, or invalid access token                  |
| 403  | Forbidden            | IP not whitelisted (if [IP whitelisting](authentication.md#ip-whitelisting) is enabled) |
| 404  | Not Found            | Payment, webhook, or resource does not exist               |
| 409  | Conflict             | Duplicate `externalId` (see [Idempotency](idempotency.md)) |
| 422  | Unprocessable Entity | Payment declined, refund exceeds payment amount            |

### Server Error Codes

| Code | Meaning              | Action                                      |
|------|----------------------|---------------------------------------------|
| 500  | Internal Server Error| Retry with exponential backoff              |
| 502  | Bad Gateway          | Retry with exponential backoff              |
| 503  | Service Unavailable  | Retry with exponential backoff              |

## Retry Strategy

For `5xx` errors and network timeouts, retry with exponential backoff:

```
Attempt 1: wait 1 second
Attempt 2: wait 2 seconds
Attempt 3: wait 4 seconds
Attempt 4: wait 8 seconds
(stop after 4 retries)
```

For `4xx` errors, do **not** retry — fix the request first. The exception is `401 Unauthorized`, where you should refresh your access token and retry once (see [Authentication](authentication.md)).

## Common Error Scenarios

### Invalid card details (400)

```json
{
  "status": 400,
  "title": "Bad Request",
  "detail": "Validation failed",
  "type": "https://api.paygate.systems/errors/validation",
  "instance": "/payment",
  "errors": [
    { "location": "card.expiry.year", "message": "Card is expired", "value": "2024" },
    { "location": "card.cvc", "message": "CVC must be 3 or 4 digits", "value": "12" }
  ]
}
```

### Duplicate external ID (409)

```json
{
  "status": 409,
  "title": "Conflict",
  "detail": "A transaction with externalId 'order-001' already exists",
  "type": "https://api.paygate.systems/errors/duplicate-external-id",
  "instance": "/payment"
}
```

### Payment declined (422)

```json
{
  "status": 422,
  "title": "Unprocessable Entity",
  "detail": "Payment was declined by the issuing bank",
  "type": "https://api.paygate.systems/errors/payment-declined",
  "instance": "/payment"
}
```

### Expired token (401)

```json
{
  "status": 401,
  "title": "Unauthorized",
  "detail": "Access token has expired",
  "type": "https://api.paygate.systems/errors/unauthorized",
  "instance": "/payment"
}
```
