# Inspired Orders API response

## Part A  Risk-based API test design

For this API I would start with the areas that can cause the biggest business impact if they fail. The API is small, but the main risks are still clear: creating orders, changing order status, and retrieving/filtering orders correctly.

### 1. Create an order

Endpoint: `POST /orders`

This is the main scenario I would prioritise first. If order creation is broken, the API is not really usable because customers or client systems cannot create new orders. It also affects anything downstream that depends on an order existing, such as fulfilment, reporting, or later status updates.

Preconditions and request data:

A valid bearer token is required.

```http
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

Example request body:

```json
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
```

Expected result:

The API should return `201 Created`. The response should contain an order with the required fields from the spec: `id`, `customerId`, `items`, `status`, `total`, and `createdAt`.

Checks I would make:

- Status code is `201`.
- `id` is returned and is a UUID.
- `customerId` matches the request.
- `items` is not empty.
- Item values match what was sent.
- `status` is a valid order status.
- `total` is correct, for example `2 * 50 = 100`.

Negative checks:

- No token should return `401`.
- Missing `customerId` should return `400`.
- Empty `items` should return `400`.
- `quantity` below `1` should return `400`.
- Negative `unitPrice` should return `400`.

### 2. Update order status

Endpoint: `PATCH /orders/{orderId}`

This is also high risk because status drives the lifecycle of the order. If the API allows a wrong status update, an order could be shipped when it should not be, cancelled incorrectly, or left in a state the business cannot explain.

Preconditions and request data:

A valid bearer token is required. A valid existing `orderId` is needed, and it must be a UUID.

```http
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

Example request:

```http
PATCH /orders/{{orderId}}
```

Body:

```json
{
  "status": "paid"
}
```

Expected result:

The API should return `200 OK`, and the returned order should show the updated status.

Checks I would make:

- Status code is `200`.
- Response contains the order fields.
- `id` matches the order ID from the path.
- `status` matches the requested value.
- The status is one of the allowed values: `paid`, `shipped`, or `cancelled`.

Negative checks:

- Unknown order ID should return `404`.
- Illegal status transition should return `409`.
- Invalid status value should return `400`.
- No token should return `401`.

### 3. List and filter orders

Endpoint: `GET /orders`

This matters because order listing is normally used by support, operations, or reporting screens. If the list or filters are wrong, people may act on incorrect data or miss orders that need attention.

Preconditions and request data:

A valid bearer token is required. There should be known test orders with different statuses.

Example request:

```http
GET /orders?status=paid&limit=20
Authorization: Bearer {{access_token}}
```

Expected result:

The API should return `200 OK` with `data` and `total`. `data` should be an array of orders, and `total` should be an integer.

Checks I would make:

- Status code is `200`.
- Response has `data` and `total`.
- `data` is an array.
- `total` is an integer.
- Each item in `data` follows the order schema.
- If `status=paid` is used, all returned orders should have status `paid`.
- `limit` should respect the allowed range of `1` to `100`.

Negative checks:

- Invalid status should fail validation.
- `limit=0` should fail validation.
- `limit=101` should fail validation.
- No token should return `401`.

## Part B — Automation path: one scenario

I would automate `POST /orders` first. It is the most useful starting point because it proves the main business function works, and it can also create an `orderId` that can be reused in later tests.

```ts
import { test, expect, request } from '@playwright/test';

const baseURL = process.env.API_BASE_URL ?? 'https://api.assessment.example.com/v1';
const token = process.env.API_TOKEN ?? 'replace-with-valid-token';

test.describe('Inspired Orders API - create order', () => {
  test('creates an order and validates the response', async () => {
    const api = await request.newContext({
      baseURL,
      extraHTTPHeaders: {
        Authorization: `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    });

    const payload = {
      customerId: '11111111-1111-1111-1111-111111111111',
      items: [
        {
          sku: 'SKU-001',
          quantity: 2,
          unitPrice: 50
        }
      ]
    };

    const response = await api.post('/orders', { data: payload });
    expect(response.status()).toBe(201);

    const body = await response.json();

    expect(body.id).toMatch(
      /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i
    );

    expect(body.customerId).toBe(payload.customerId);
    expect(Array.isArray(body.items)).toBe(true);
    expect(body.items.length).toBeGreaterThanOrEqual(1);
    expect(body.items[0].sku).toBe('SKU-001');
    expect(body.items[0].quantity).toBe(2);
    expect(body.items[0].unitPrice).toBe(50);
    expect(['pending', 'paid', 'shipped', 'cancelled']).toContain(body.status);
    expect(body.total).toBe(100);
    expect(new Date(body.createdAt).toString()).not.toBe('Invalid Date');

    await api.dispose();
  });

  test('rejects an order when customerId is missing', async () => {
    const api = await request.newContext({
      baseURL,
      extraHTTPHeaders: {
        Authorization: `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    });

    const invalidPayload = {
      items: [
        {
          sku: 'SKU-001',
          quantity: 2,
          unitPrice: 50
        }
      ]
    };

    const response = await api.post('/orders', { data: invalidPayload });
    expect(response.status()).toBe(400);

    await api.dispose();
  });
});
```

The main checks in the positive test are the `201` status, required response fields, UUID format, item details, valid status, and the total calculation. The negative test uses the same endpoint but removes `customerId`, which is required by the `OrderCreate` schema, so I would expect a `400` validation error.

For dynamic values, I would not hardcode the token or environment URL in the test. These should come from `API_TOKEN` and `API_BASE_URL`. If this test was extended, I would capture the returned `id` and use it in later calls like `GET /orders/{orderId}` or `PATCH /orders/{orderId}`.

For CI stability, the test should create its own order data instead of relying on existing shared data. The token should be stored as a secure CI variable. The environment should be stable enough for repeat runs, and tests that change state should either use isolated test customers or have a cleanup/reset approach.

## Part C — Contract awareness

One change that would break a consumer is removing or renaming the `total` field from the `Order` response. The spec marks `total` as required, so a client may depend on it for displaying the order amount, doing reconciliation, or checking that the order value is correct. If the API changed `total` to something like `amount`, the HTTP call might still return `200` or `201`, but the consumer could still break because the field it expects is missing. A consumer contract test would catch this by calling `POST /orders` or `GET /orders/{orderId}` and checking that `total` still exists, is numeric, and has the expected value.
