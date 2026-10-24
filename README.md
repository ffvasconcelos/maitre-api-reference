# Maitre API Reference

A [Bruno](https://www.usebruno.com/) collection for [Maitre API](https://github.com/ffvasconcelos/maitre-api), using OpenCollection YAML 1.0.0.

## Usage

1. Run Maitre API and apply its database migrations.
2. Open this repository with **Open Collection** in Bruno and select **maitre-local**.
3. Set `api` if the API runs somewhere other than `http://127.0.0.1:8000`.
4. Choose an unused positive `table_number`, then send **Products / Create product** and **Tables / Create**. Successful creates capture `product_id` and `table_id` as runtime variables.
5. Send **Orders / Add order** to capture `order_id`, then get the order or add items.
6. Close **or** cancel the open order. Both actions release its table; trying another mutation on that same order returns 409. Create another order to try the other terminal action.

Alternatively, set `table_id`, `product_id`, and `order_id` in the environment to existing UUIDs. The supplied zero UUIDs are placeholders, not seeded records. Captured runtime values take precedence during the session. Repeated table creation with the same number returns 409.

Requests that create or change resources mutate the selected API database. Choose requests deliberately; this collection is an interactive reference, not a fixture-resetting test suite. In particular, close and cancel are alternative lifecycle paths. No scripts send additional requests automatically.

Order filters `table_id` and `status` are disabled by default. Enable either parameter in **Orders / List orders** when needed; the URL itself contains no duplicated query string.

## Request and response contracts

Each of the 15 request files includes saved request/response examples for its success and applicable error outcomes. Examples were captured from the API using TestClient and a disposable PostgreSQL database; UUIDs and timestamps are illustrative. Saved examples are independent snapshots, not a live seeded dataset. Inspect them in Bruno without sending requests. The DB health 503 example uses an injected failure, with an illustrative exception message.

- JSON requests declare `Content-Type: application/json`.
- UUIDs and timestamps are JSON strings. Decimal response fields (`price`, `unit_price`, `subtotal`) are JSON strings; request examples also use strings for prices.
- Table statuses are `disponível` and `ocupada`; order statuses are `Aberto`, `Fechado`, and `Cancelado`.
- Product creation defaults to `active: true` and `description: null` when omitted.
- `number`, `capacity`, and item `quantity` must be positive. Product names have a 150-character maximum, descriptions 250, and prices at most 10 digits with 2 decimal places. These are the current constraints, not additional validation rules introduced by this collection.
- Adding order items expects a JSON array, not an object wrapping `items`; it returns the updated order with HTTP 201.
- Order lists return summaries without `items`. Single-order and order mutation responses include items, product names, unit prices and computed subtotals.
- Lists are arrays and may be empty; the current API has no pagination envelope.

| Request | Success | Documented error outcomes |
| --- | --- | --- |
| `GET /health` | 200 | — |
| `GET /health/db` | 200 | 503 |
| `POST /products` | 201 | 422 |
| `GET /products` | 200 | — |
| `GET /products/{product_id}` | 200 | 404, 422 |
| `POST /tables` | 201 | 409, 422 |
| `GET /tables` | 200 | — |
| `GET /tables/{table_id}` | 200 | 404, 422 |
| `GET /tables/by_number/{number}` | 200 | 404, 422 |
| `POST /orders` | 201 | 404, 409, 422 |
| `GET /orders` | 200 | 422 |
| `GET /orders/{order_id}` | 200 | 404, 422 |
| `POST /orders/{order_id}/items` | 201 | 404, 409, 422 |
| `POST /orders/{order_id}/close` | 200 | 404, 409, 422 |
| `POST /orders/{order_id}/cancel` | 200 | 404, 409, 422 |

### Domain errors

Domain errors return a stable machine-readable `code` and a human-readable `detail`. Match the code in clients, since messages may contain IDs or table numbers.

```json
{
  "code": "order_not_found",
  "detail": "Pedido não encontrado."
}
```

| HTTP status | Code | Condition |
| --- | --- | --- |
| 404 | `order_not_found` | Order UUID does not exist. |
| 404 | `product_not_found` | Product UUID does not exist. |
| 404 | `table_not_found` | Table UUID does not exist. |
| 404 | `table_number_not_found` | Table number does not exist. |
| 409 | `order_not_open` | Adding items, closing or cancelling an order that is no longer open. |
| 409 | `product_not_active` | Opening an order or adding items with an inactive product. |
| 409 | `table_not_available` | Opening an order on an occupied table. |
| 409 | `table_number_already_exists` | Creating a table with a number already in use. |

Validation failures retain the native FastAPI HTTP 422 format: `detail` is an array containing fields such as `type`, `loc`, `msg`, `input`, and sometimes `ctx`. They do not receive a domain `code`.

The current `/health/db` failure response is HTTP 503 with only `detail` containing the exception text. Unexpected errors remain HTTP 500 and do not have a guaranteed domain-error JSON format. The collection does not imply otherwise.

## Verification

Saved responses were checked against the local API's generated OpenAPI response schemas where declared. The HTTP YAML files were also checked against the official [OpenCollection schema](https://github.com/opencollection-dev/opencollection/blob/main/packages/oc-schema/src/opencollection.schema.json). The simulated DB health failure verifies the current handler without taking a real database down.

Maitre API is in development. Keep these examples aligned with route declarations, Pydantic schemas and the centralized domain exception handlers when contracts change. The live `/docs` and `/openapi.json` endpoints expose the API's current schema.
