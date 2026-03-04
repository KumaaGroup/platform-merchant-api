# Table of contents

* [API Guide](README.md)
  * [Getting Started](docs/getting-started.md)
  * [Idempotency](docs/idempotency.md)
  * [Authentication](docs/authentication.md)
  * [Webhooks](docs/webhooks.md)
  * [Error Handling](docs/error-handling.md)
  * [Card Payments](docs/card-payments.md)
  * [Refunds](docs/refunds.md)
  * [Blocklist and Whitelist](docs/blocklist-and-whitelist.md)
  * [Open Banking](docs/open-banking.md)
* ```yaml
  type: builtin:openapi
  props:
    models: true
    downloadLink: true
  dependencies:
    spec:
      ref:
        kind: openapi
        spec: merchants-api
  ```
