# Error Handling

The Platform Merchants API returns two kinds of error responses, depending on where the request fails.

## Business Errors

Errors raised while processing a well-formed request — a duplicate `externalId`, a missing resource, an authentication failure, a declined operation — are returned as `application/json` with a `message` field:

```json
{
  "message": "duplicate payment request"
}
```

| Field      | Type   | Description                                                  |
|------------|--------|--------------------------------------------------------------|
| `message`  | string | Description of what went wrong                               |
| `errors`   | array  | Field-level details, when applicable (optional)              |

When the `errors` array is present, each entry pinpoints a specific field:

| Field      | Type   | Description                           |
|------------|--------|---------------------------------------|
| `field`    | string | Field name                            |
| `message`  | string | What is wrong with the field          |

For server errors (`5xx`), the `message` is a generic status description — implementation details are never exposed.

## Request Validation Errors

Requests that fail schema validation before processing (missing required fields, pattern or type mismatches, malformed JSON) are rejected with the [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) Problem Details format, content type `application/problem+json`:

```json
{
  "status": 422,
  "title": "Unprocessable Entity",
  "detail": "validation failed",
  "errors": [
    {
      "location": "body.card.cvc",
      "message": "expected string to match pattern ^[0-9]{3,4}$",
      "value": "12"
    }
  ]
}
```

Each entry in `errors` identifies the invalid field by its `location` (dot-notation path), the validation `message`, and the submitted `value`.

## HTTP Status Codes

### Success Codes

| Code | Meaning     | Used By                                       |
|------|-------------|-----------------------------------------------|
| 200  | OK          | Successful GET, POST, PATCH requests (including refund submission) |
| 201  | Created     | Webhook created                               |
| 204  | No Content  | Webhook deleted successfully                  |

### Client Error Codes

| Code | Meaning              | Common Causes                                              |
|------|----------------------|------------------------------------------------------------|
| 400  | Bad Request          | Malformed JSON, invalid field values, unsupported currency, batch size exceeded |
| 401  | Unauthorized         | Missing, expired, or invalid access token                  |
| 403  | Forbidden            | IP not whitelisted (if [IP whitelisting](authentication.md#ip-whitelisting) is enabled) |
| 404  | Not Found            | Payment, webhook, or resource does not exist               |
| 409  | Conflict             | Duplicate `externalId` (see [Idempotency](idempotency.md)) |
| 422  | Unprocessable Entity | Request validation failed, payment declined, partial refund not supported, `useRefunds: true` |

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

### Duplicate external ID (409)

```json
{
  "message": "duplicate payment request"
}
```

### Payment cannot be accepted (422)

```json
{
  "message": "msb is not activated"
}
```

### Expired or invalid token (401)

```json
{
  "message": "missing or invalid authentication headers"
}
```

### Invalid test card (400)

Sandbox payments must use the [published test cards](card-payments.md#testing) with all fields matching exactly:

```json
{
  "message": "card is not valid for test payments"
}
```

> **Note — decline reasons are not HTTP errors:** a payment that is accepted but later declined ends in status `DECLINED` with the reason in the `responseCode` field (`DO_NOT_HONOUR`, `INSUFFICIENT_FUNDS`, `THREE_DS_FAILED`, `CARD_NOT_WHITELISTED`, …), delivered via webhook and visible on the payment record. See the per-flow pages for the decline codes relevant to each lifecycle.
