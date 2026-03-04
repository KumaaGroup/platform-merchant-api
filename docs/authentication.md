# Authentication

The Platform Merchants API uses **OAuth2 Client Credentials** for authentication. You exchange your `client_id` and `client_secret` for a short-lived access token, then include that token in every API request.

## Obtaining an Access Token

```bash
curl -X POST https://test-merchants-api.nonprod.paygate.systems/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "grant_type=client_credentials"
```

### Request Parameters

| Parameter       | Type   | Required | Description                          |
|-----------------|--------|----------|--------------------------------------|
| `client_id`     | string | Yes      | Your merchant API client ID          |
| `client_secret` | string | Yes      | Your merchant API client secret      |
| `grant_type`    | string | Yes      | Must be `client_credentials`         |

The request body must be sent as `application/x-www-form-urlencoded`.

### Response

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

| Field          | Type    | Description                                    |
|----------------|---------|------------------------------------------------|
| `access_token` | string  | JWT token to include in API requests           |
| `expires_in`   | integer | Token lifetime in seconds (typically 3600)     |
| `token_type`   | string  | Always `Bearer`                                |

## Using the Token

Include the access token in the `Authorization` header of every API request:

```bash
curl https://test-merchants-api.nonprod.paygate.systems/payment \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## Token Lifecycle

Tokens expire after the number of seconds indicated by `expires_in` (typically 1 hour). When a token expires, the API returns a `401 Unauthorized` response.

### Recommended Refresh Strategy

Rather than requesting a new token for every API call or waiting for a `401` error, cache the token and refresh it proactively:

```
1. Request a token and store it with its expiry time
2. Use the cached token for all API calls
3. When the token is within 5 minutes of expiring, request a new one
4. If you receive a 401 response, request a new token immediately and retry
```

### Example (pseudocode)

```python
import time
import requests

class MerchantAPIClient:
    def __init__(self, client_id, client_secret, base_url):
        self.client_id = client_id
        self.client_secret = client_secret
        self.base_url = base_url
        self.token = None
        self.token_expiry = 0

    def get_token(self):
        if self.token and time.time() < self.token_expiry - 300:
            return self.token

        response = requests.post(
            f"{self.base_url}/oauth2/token",
            data={
                "client_id": self.client_id,
                "client_secret": self.client_secret,
                "grant_type": "client_credentials",
            },
        )
        data = response.json()

        self.token = data["access_token"]
        self.token_expiry = time.time() + data["expires_in"]
        return self.token

    def request(self, method, path, **kwargs):
        headers = {"Authorization": f"Bearer {self.get_token()}"}
        response = requests.request(
            method, f"{self.base_url}{path}", headers=headers, **kwargs
        )

        if response.status_code == 401:
            self.token = None
            headers = {"Authorization": f"Bearer {self.get_token()}"}
            response = requests.request(
                method, f"{self.base_url}{path}", headers=headers, **kwargs
            )

        return response
```

## Security Best Practices

- **Never expose credentials client-side.** All API calls must originate from your backend server.
- **Store `client_id` and `client_secret` securely.** Use environment variables or a secrets manager — never commit them to source control.
- **Use HTTPS exclusively.** All API endpoints require HTTPS. Plain HTTP requests are rejected.
- **Rotate credentials periodically.** Contact support to rotate your client secret if you suspect it has been compromised.

## Auth Server URLs

The authentication endpoint uses environment-specific URLs:

| Environment | Auth URL                                                        |
|-------------|-----------------------------------------------------------------|
| Test        | `https://test-merchants-api.nonprod.paygate.systems/oauth2/token` |
| Sandbox     | `https://sandbox-merchants-api.nonprod.paygate.systems/oauth2/token` |

## Error Responses

| Status | Meaning                  | Action                                      |
|--------|--------------------------|---------------------------------------------|
| 400    | Invalid request          | Check `grant_type` and credential format    |
| 401    | Invalid credentials      | Verify `client_id` and `client_secret`      |
| 500    | Server error             | Retry with exponential backoff              |
