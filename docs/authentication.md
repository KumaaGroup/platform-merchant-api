# Authentication

The Platform Merchants API uses **OAuth2 Client Credentials** for authentication. You exchange your `client_id` and `client_secret` for a short-lived access token, then include that token in every API request.

## Obtaining an Access Token

```bash
curl -X POST https://sandbox-auth.nonprod.paygate.systems/oauth2/token \
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
curl https://sandbox-merchants-api.nonprod.paygate.systems/payment \
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

### Example (Python)

```python
import time
import requests

class MerchantAPIClient:
    AUTH_URL = "https://sandbox-auth.nonprod.paygate.systems"
    API_URL = "https://sandbox-merchants-api.nonprod.paygate.systems"

    def __init__(self, client_id, client_secret):
        self.client_id = client_id
        self.client_secret = client_secret
        self.token = None
        self.token_expiry = 0

    def get_token(self):
        if self.token and time.time() < self.token_expiry - 300:
            return self.token

        response = requests.post(
            f"{self.AUTH_URL}/oauth2/token",
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
            method, f"{self.API_URL}{path}", headers=headers, **kwargs
        )

        if response.status_code == 401:
            self.token = None
            headers = {"Authorization": f"Bearer {self.get_token()}"}
            response = requests.request(
                method, f"{self.API_URL}{path}", headers=headers, **kwargs
            )

        return response
```

## IP Whitelisting

IP whitelisting adds an extra layer of security by restricting API access to a predefined set of IP addresses. When enabled, requests from any non-whitelisted IP are rejected with a `403` error.

IP whitelisting is **managed entirely by platform administrators**. To enable it or update your allowed IP addresses, contact your platform admin and provide the server IPs that should be permitted. The admin will configure the whitelist on your behalf.

This is recommended for production integrations where your API calls originate from a fixed set of servers with known IP addresses.

## Security Best Practices

- **Never expose credentials client-side.** All API calls must originate from your backend server.
- **Store `client_id` and `client_secret` securely.** Use environment variables or a secrets manager — never commit them to source control.
- **Use HTTPS exclusively.** All API endpoints require HTTPS. Plain HTTP requests are rejected.
- **Enable IP whitelisting.** Contact your platform admin to restrict API access to your known server IP addresses.
- **Rotate credentials periodically.** Contact support to rotate your client secret if you suspect it has been compromised.

## Error Responses

| Status | Meaning                  | Action                                      |
|--------|--------------------------|---------------------------------------------|
| 400    | Invalid request          | Check `grant_type` and credential format    |
| 401    | Invalid credentials      | Verify `client_id` and `client_secret`      |
| 403    | IP not whitelisted       | Contact your platform admin to add your server IP to the whitelist |
| 500    | Server error             | Retry with exponential backoff              |
