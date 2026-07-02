## Part B — Demonstrated scenario

I would demonstrate POST /orders because it is the highest-value flow.

Request:

POST /orders

Authorization: Bearer {{token}}

Content-Type: application/json

{

  "customerId": "11111111-1111-1111-1111-111111111111",

  "items": [

    {

      "sku": "SKU-001",

      "quantity": 2,

      "unitPrice": 50

    }

  ]

}

Expected response:

201 Created

Checks:

- Status code is 201.

- Response has required fields: id, customerId, items, status, total, createdAt.

- id and customerId are valid UUIDs.

- items contains the SKU, quantity and unitPrice sent in the request.

- total is 100.

- status is one of pending, paid, shipped or cancelled.

Negative case:

Send POST /orders without customerId. Because customerId is required, the API should return 400 validation error.

Environment/config:

The base URL and token should be stored as variables, not hardcoded. Created order IDs should be captured from the response and reused in later tests. CI should use isolated test data and secure variables for API_TOKEN and API_BASE_URL.

## Part C  Contract awareness

A breaking API change would be removing or renaming the total field from the Order response. The schema marks total as required, so consumers may rely on it for order display, reconciliation or payment checks. If total was renamed to amount or removed, consumers could fail even if the API still returns 200 or 201. A consumer contract test would detect this by asserting that POST /orders and GET /orders/{orderId} still return total as a numeric field with the expected value.
